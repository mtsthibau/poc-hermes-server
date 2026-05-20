# Architecture Audit — poc-hermes-server

**Auditor**: Architecture Auditor (adversarial review)
**Date**: 2026-05-20
**Scope**: Complete documentation corpus — 11 architecture docs, 7 ADRs, 4 roadmap phases, governance
**Verdict**: Conditionally sound with critical gaps that must be resolved before Phase 2 begins

---

## Severity Classification

| Level | Meaning |
|---|---|
| **CRITICAL** | Will cause silent data loss, security breach, or unrecoverable production failure |
| **HIGH** | Will cause degraded reliability, incorrect behavior, or significant operational risk |
| **MEDIUM** | Creates technical debt, operational blindspots, or incorrect assumptions |
| **LOW** | Minor inconsistency, missing detail, or process gap |

---

## CRITICAL Findings

---

### C-1: Message Delivery Has an Unrecoverable Crash Window

**Documents**: `docs/architecture/event-driven.md §4.1`, `docs/architecture/offline-first.md §6`
**Status**: Open
**Must fix by**: Phase 2

The new message flow is documented as:

```
INSERT INTO messages → INSERT INTO message_deliveries → EventBus.publish('message.new')
  └─► DeliveryWorker (BullMQ, subscribed to 'message.new')
```

**The flaw**: BullMQ delivery jobs are created in response to the Redis Pub/Sub event, not the database write. If the process crashes — or Redis becomes unavailable — after the PostgreSQL commit but before `EventBus.publish()`, the following happens:

- `message_deliveries` rows exist with `status=pending`
- No Redis event fires → no BullMQ job created → no delivery attempt ever happens
- Offline clients never receive the message
- No documented recovery mechanism exists

This is not a theoretical failure mode. Node.js process crashes, OOM kills, Redis network partitions, and deployment restarts all produce this window on every single message send.

**Required fix**: A "missed delivery recovery" job that periodically scans for `message_deliveries WHERE status='pending' AND created_at < NOW() - INTERVAL '60 seconds'` and re-enqueues delivery. This job must be documented, implemented in Phase 2, and its Prometheus metric (`missed_delivery_recovery_total`) alerted on.

Consider making BullMQ job creation the primary trigger (not Redis event), with Redis Pub/Sub as the real-time notification layer only. The delivery pipeline must not depend on a fire-and-forget transport for durability.

---

### C-2: sync_queue Stores Stale Payload — Edited Messages Deliver Wrong Content

**Documents**: `docs/architecture/offline-first.md §6`, `docs/architecture/event-driven.md §4.1`
**Status**: Open
**Must fix by**: Phase 3

The `SyncWorker` appends `payload=messageJson` (the full message body) to `sync_queue` at the moment of message creation. The sync_queue is described as "append-only."

**The flaw**: If a sender edits a message after it is queued for an offline recipient, the sync_queue row contains the pre-edit content. When the offline client reconnects and receives `SYNC_DELTA`, it receives the original (stale) version. The edit event produces a separate sync_queue row, but correct application depends on the client receiving both the create and all subsequent edit rows in the correct order without gaps.

**Required fix**: The sync_queue should store entity references (`entity_id`, `operation`, `entity_type`) without embedding the full payload snapshot. The SyncService fetches the current entity state at sync time. This design:
- Always delivers current state, never a stale snapshot
- Eliminates payload duplication (see M-1)
- Makes the queue a pure event log, not a message store

---

### C-3: Multi-Device Cursor Model Is Broken for Online Clients

**Documents**: `docs/architecture/offline-first.md §4`
**Status**: Open
**Must fix by**: Phase 3

The `SyncCursors` interface tracks `last_event_sequence` per device in the `sync_cursors` table. This is correct for offline clients recovering via `SYNC_REQUEST`. But for online clients receiving events via WebSocket push, the sync_cursor is never advanced.

**The flaw**:

- Device A is online, receives 20 `MESSAGE_NEW` frames via WebSocket push — no sync_queue rows consumed, cursor stays at 0
- Device A disconnects and reconnects
- Device A sends `SYNC_REQUEST { messages: 0 }` (cursor was never updated)
- Server queries sync_queue from cursor=0, finds all 20 messages
- Device A receives 20 duplicate messages

The two delivery paths (real-time WebSocket push vs sync_queue replay) must share a cursor advancement mechanism. Either:

1. The server updates `sync_cursors` whenever it successfully pushes a `MESSAGE_NEW` frame to a device
2. Or the client sends a `CURSOR_ADVANCE` message back to the server acknowledging each received frame

Neither mechanism is documented. This is a fundamental correctness gap in the sync model.

---

### C-4: JWT Expiry During Active WebSocket Sessions Is Unhandled

**Documents**: `docs/architecture/security.md §2.4`, `docs/architecture/websocket-protocol.md §3`
**Status**: Open
**Must fix by**: Phase 2

Access tokens expire after 15 minutes. WebSocket connections authenticate once on connect and the token is never re-validated. A WebSocket session authenticated at T=0 with a valid JWT remains "authenticated" on the server at T=4h.

**This covers two failure scenarios**:

1. **Token revocation doesn't terminate sessions**: If an admin suspends a user, their WebSocket connection remains active indefinitely. The user continues sending messages and radio commands.

2. **Compromised session**: If a refresh token is stolen and revoked, any active WebSocket connection authenticated with the derived access token remains active until natural disconnect.

**Required fix**: The server must periodically re-validate WebSocket sessions against the `user_sessions` table. Options:
- Re-validate session liveness every N minutes (recommended — single DB query per connection)
- Require client to send a refreshed `AUTHENTICATE` message before each JWT expiry
- Add `SESSION_INVALIDATED` server→client message type for forced disconnect

Add `SESSION_INVALIDATED` to the WebSocket message type catalog in `websocket-protocol.md`.

---

### C-5: NATS Migration "Zero Application Code Change" Claim Is Incorrect

**Documents**: `docs/architecture/event-driven.md §7`, `docs/adr/ADR-003-event-broker.md`
**Status**: Open
**Must fix by**: Before Phase 4 design begins

ADR-003 states: *"Switch via configuration: `EVENT_BUS_DRIVER=nats` (zero application code change)."*

**The flaw**: This is architecturally unsound. Redis Pub/Sub has fan-out semantics — every subscriber receives every message. NATS JetStream consumer groups have competing-consumer semantics — only ONE consumer in a group receives each message.

The `DeliveryWorker` subscribes to `message.new` and creates BullMQ jobs. Under Redis, if 3 server processes are running, all 3 receive every `message.new` event and all 3 try to create delivery jobs — a triple-delivery risk already present. Under NATS JetStream with consumer groups, only one process handles each event.

This is a behavioral semantic change, not a transport swap. Code requiring fan-out (WebSocket gateway, telemetry) and code requiring exclusive processing (delivery workers, sync workers) need different subscription semantics. The `IEventBus` interface cannot satisfy both with a single `subscribe()` method.

**Required fix**: Correct ADR-003. Add `subscribeExclusive<T>()` (consumer-group semantics) to the `IEventBus` interface alongside `subscribe<T>()` (fan-out semantics). Use `subscribeExclusive` for all worker-style subscribers. Reflect this in Phase 4 planning.

---

## HIGH Findings

---

### H-1: conversations.last_message_id Creates Hot-Row Write Contention

**Documents**: `docs/architecture/database-schema.md §3.2`
**Status**: Open
**Must fix by**: Phase 1 (schema design)

Every message INSERT must also UPDATE `conversations.last_message_id`. This creates a hot row — every concurrent message to the same conversation serializes on the row lock for that conversation. Under concurrent messaging load, p99 latency on message sends will spike due to PostgreSQL row-level locking.

**Recommended fix**: Compute `last_message_id` on-read via `SELECT id FROM messages WHERE conversation_id = $1 ORDER BY created_at DESC LIMIT 1` with a composite index on `(conversation_id, created_at DESC)`. Remove the denormalized field. The `idx_messages_conversation_created` index in the schema already supports this query.

---

### H-2: WebSocket Presence TOCTOU — 30-Second ACK Timeout Is Dangerous

**Documents**: `docs/architecture/offline-first.md §6`
**Status**: Open
**Must fix by**: Phase 2

```
[ONLINE] → Push MESSAGE_NEW via WebSocket
  └─► Wait for MESSAGE_ACK (30s timeout)
        └─► [No ACK] → Fall through to offline queue
```

**The flaws**:

1. A client that disconnects after the presence check but before receiving the frame causes a 30-second delay before fallback queuing. For emergency radio coordination, 30 seconds of silent non-delivery is unacceptable.
2. `ws.send()` in Node.js fails synchronously if the socket is closed. This failure should immediately trigger the fallback path, not start a 30-second timer.
3. The mechanism for "fall through to offline queue" is not wired into the event-driven.md flow diagram. The gap between these two documents means the implementation path is undefined.

**Required fix**: On WebSocket send failure (`ws.send()` error callback), immediately insert into sync_queue without waiting. Reserve the ACK-timeout fallback only for the case where the send succeeds but no ACK is received (delivery confirmation, not send failure).

---

### H-3: Redis Is a Single Point of Failure for Safety-Critical Radio Control

**Documents**: `docs/architecture/hal.md §2`, `docs/adr/ADR-002-database-strategy.md`
**Status**: Open
**Must fix by**: Phase 1

The `RadioCommandQueue` uses BullMQ, which requires Redis. If Redis becomes unavailable, radio commands cannot be enqueued — PTT cannot be activated or deactivated, frequency cannot be changed. There is no fallback path for radio control.

In field deployments — exactly where Redis failures are more likely (hardware stress, power fluctuation, limited RAM) — radio control silently fails.

**Required fix**: A circuit-breaker fallback that allows direct synchronous HAL command execution (bypassing BullMQ) when Redis is unavailable, using an in-process async mutex for serialization. Critical commands (`PTT`, `stop`) must always be executable. Document this in `hal.md` and add a health check alert for `redis.unavailable` → `radio.command_queue.degraded`.

---

### H-4: Refresh Token Rotation Is Explicitly Optional — This Is a Security Defect

**Documents**: `docs/architecture/security.md §2.3`
**Status**: Open
**Must fix by**: Phase 1

```
4. Optionally rotate refresh token (sliding window)
```

Token rotation must be mandatory. Without it:
- A stolen 30-day refresh token grants an attacker 30 days of undetected access
- There is no mechanism to detect theft until both parties use the token (only possible if rotation is enabled)
- The legitimate user cannot discover theft until their next refresh attempt, if the attacker used the token first

RFC 6819 OAuth 2.0 threat model identifies non-rotating refresh tokens as a high-severity vulnerability. In a system used for emergency radio communications where operator identity is safety-critical, this cannot be optional.

**Required fix**: Remove "Optionally" from the refresh flow. Rotation is mandatory. Add Prometheus metric `auth_refresh_token_reuse_detected_total` (a non-zero value indicates an active attack or a client bug). Alert on reuse.

---

### H-5: Attachment Storage Has No Backend Abstraction

**Documents**: `docs/architecture/messaging-domain.md §attachment pipeline`
**Status**: Open
**Must fix by**: Phase 1

The attachment system stores files at a configured filesystem path with no `IStorageBackend` abstraction. Phase 4 federation requires attachments accessible from multiple stations. Moving to object storage in Phase 4 requires touching every attachment read/write code path.

**Required fix**: Define the abstraction from day one in Phase 1:

```typescript
interface IStorageBackend {
  write(key: string, stream: Readable, metadata: AttachmentMetadata): Promise<void>
  readStream(key: string): Promise<Readable>
  delete(key: string): Promise<void>
  getSignedUrl(key: string, ttlSeconds: number): Promise<string>
}
```

Implement `LocalFileStorageBackend` for Phase 1–3. `MinioStorageBackend` for Phase 4. This mirrors the `IRadioDriver` pattern already used in the HAL — apply it consistently.

---

### H-6: TURN Credential Endpoint Missing from RBAC Matrix

**Documents**: `docs/architecture/security.md §3`, `docs/architecture/webrtc.md`
**Status**: Open
**Must fix by**: Phase 2

The RBAC permission matrix has no row for `GET /api/v1/webrtc/credentials`. This endpoint generates valid TURN relay credentials. Without explicit RBAC, any authenticated user (including `readonly` role) can obtain relay credentials and use coturn as an arbitrary TCP/UDP relay.

**Required fix**: Add to RBAC matrix — minimum `user` role required. Embed `userId` in the HMAC credential so relay usage is auditable and credentials are non-transferable. Log every issuance in `audit_logs` with `action: 'webrtc.credentials_issued'`. Add rate limiting to this endpoint.

---

## MEDIUM Findings

---

### M-1: sync_queue Payload Duplicated Per Device — Stale and Space-Inefficient

**Documents**: `docs/architecture/offline-first.md §6`
**Status**: Open (addressed by C-2 fix)
**Must fix by**: Phase 3

With N offline devices and M messages, the SyncWorker inserts N×M rows of full JSON into sync_queue. For a broadcast conversation with 50 offline recipients and 100 queued messages, this is 5,000 duplicated rows. Editing a message does not update stale payloads already in sync_queue.

**Fix**: Resolved by C-2 fix (store entity references, fetch current state at sync time).

---

### M-2: No Cleanup Policy for Never-Reconnecting Devices

**Documents**: `docs/architecture/offline-first.md §12`
**Status**: Open
**Must fix by**: Phase 3

The cleanup worker removes sync_queue rows older than 7 days that are processed. It does not address devices that never reconnect. A user who reinstalls the app 10 times without deregistering generates 10 sync_queue rows per incoming message indefinitely.

**Required fix**: Add `last_seen_at` to `user_devices`. Add cleanup policy:
- Mark devices inactive if not seen in 90 days
- Stop inserting sync_queue rows for inactive devices
- Delete sync_queue rows for inactive devices after 7 days
- Surface `inactive_device_count` as a Prometheus gauge

---

### M-3: TimescaleDB Background Jobs Compete With Application Load

**Documents**: `docs/architecture/database-schema.md §TimescaleDB`
**Status**: Open
**Must fix by**: Phase 1

TimescaleDB runs compression, chunk creation, and retention policy jobs within PostgreSQL, competing with application queries for I/O, CPU, and connection slots. On a Raspberry Pi 4 running the full stack, this contention is non-trivial.

**Required fix**: Document and configure:
- `timescaledb.max_background_workers = 2`
- `maintenance_work_mem` for compression jobs
- Maintenance window scheduling (`pg_cron` or TimescaleDB scheduling)
- Grafana panel for TimescaleDB chunk health and compression lag
- Alert for TimescaleDB background job failures

---

### M-4: WsEnvelope Has No Protocol Version Field

**Documents**: `docs/architecture/websocket-protocol.md §2.2`
**Status**: Open
**Must fix by**: Phase 2

The `hermes-v1` protocol version is negotiated in the HTTP Upgrade header only. The `WsEnvelope` has no `version` field. During rolling deployments where old and new server versions coexist behind a load balancer, there is no way to route by protocol version or detect deprecated client usage.

**Required fix**: Add `version: number` to `WsEnvelope`. Increment for breaking changes. Add `ws_client_protocol_version` as a Prometheus label. Close connections on unsupported versions with `4000 PROTOCOL_VERSION_UNSUPPORTED`.

---

### M-5: RFC Process Has No Decision Authority Defined

**Documents**: `docs/governance/RFC-TEMPLATE.md`
**Status**: Open
**Must fix by**: Immediately

The RFC template defines a 2-week comment period with "a decision is made" as the outcome. It does not define who has decision authority, what constitutes consensus, how unresolved objections are handled, or whether authors can self-approve.

**Required fix**: Add to RFC template:
- Who is the decision owner (role, not person)
- What constitutes acceptance (consensus, or explicit approval from N reviewers)
- How objections block or delay acceptance
- What happens after the 2-week window with no responses

---

### M-6: mediasoup Workers Share All CPU Cores With Other Services

**Documents**: `docs/architecture/webrtc.md §mediasoup config`
**Status**: Open
**Must fix by**: Phase 2

`numWorkers = min(os.cpus().length, 4)` on a Raspberry Pi 4 allocates all 4 cores to mediasoup workers, leaving zero headroom for PostgreSQL, Redis, BullMQ, and the Node.js event loop under concurrent WebRTC sessions.

**Required fix**: Change default to `Math.max(1, Math.min(os.cpus().length - 2, 4))`. Make configurable via `MEDIASOUP_WORKER_COUNT` environment variable. Document recommended CPU allocation for target hardware in the deployment guide:

| Component | Cores (RPi 4) |
|---|---|
| PostgreSQL + Redis | 1 |
| Node.js event loop + BullMQ | 1 |
| mediasoup workers | 2 (max) |

---

### M-7: Pino Redact Patterns Miss Nested Payload Fields

**Documents**: `docs/architecture/observability.md §2`
**Status**: Open
**Must fix by**: Phase 1

Pino's `redact` configuration uses `'*.password'` which matches only one level of nesting. Fields at `user.credentials.password`, `session.data.auth.token`, or `payload.token` (in WebSocket frames) are not redacted. Sync_queue payloads containing full message JSON may include PII in metadata fields.

**Required fix**: Expand redact paths to cover deep nesting:
```javascript
paths: [
  'password', '*.password', '**.password',
  'token', '*.token', '**.token',
  'refreshToken', '*.refreshToken',
  'authorization', '*.authorization',
  'payload.token', 'body.password', 'body.token',
]
```

Add an automated test that asserts sensitive fields are redacted in Pino log output for known log call sites.

---

## LOW Findings

---

### L-1: Federation Auth Model Deferred Without Schema Placeholder

**Documents**: `docs/roadmap/phase-4-federation.md §Security`
**Status**: Open

Station-to-station authentication is described as "shared secrets or PKI" with no commitment made. The `stations` table (Phase 4) needs credential fields whose type depends on this decision. Open ADR-008 now to decide the federation authentication model before Phase 4 schema design begins.

---

### L-2: WebRTC Use Case Phase Assignments Contradict Roadmap

**Documents**: `docs/architecture/webrtc.md §2`, `docs/roadmap/phase-4-federation.md §Week 23–25`
**Status**: Open

The WebRTC use case table lists "Radio audio streamed to browser (receive)" as Phase 3. But the radio bridge implementation (ffmpeg → ALSA → mediasoup PlainTransport) is documented in the Phase 4 roadmap. Phase 3 cannot deliver radio audio receive without the bridge. Reconcile the phase assignment in one of these documents.

---

### L-3: `localStorage` Referenced for Storage — Architecture Targets Mobile

**Documents**: `docs/architecture/offline-first.md §3`
**Status**: Open

`localStorage` is a browser-only API. The architecture explicitly targets mobile clients (Capacitor apps). Use storage-agnostic language throughout: "client-side persistent storage." Add platform-specific implementation notes where relevant (Web: `localStorage`, React Native: `AsyncStorage`, Electron: `electron-store`).

---

### L-4: Audit Log Action Strings Are Not Enforced at Compile Time

**Documents**: `docs/architecture/security.md §9`
**Status**: Open

Audit log actions (`auth.login`, `radio.ptt_activated`, etc.) are string literals scattered across the codebase. Missing actions (`message.edited`, `conversation.participant_removed`) have no documentation. Typos in action strings are invisible at compile time.

**Required fix**: Define audit actions as a TypeScript `const` enum or `as const` record:

```typescript
export const AuditAction = {
  AUTH_LOGIN: 'auth.login',
  AUTH_LOGOUT: 'auth.logout',
  AUTH_UNAUTHORIZED_ACCESS: 'auth.unauthorized_access',
  MESSAGE_SENT: 'message.sent',
  MESSAGE_EDITED: 'message.edited',
  MESSAGE_DELETED: 'message.deleted',
  RADIO_PTT_ACTIVATED: 'radio.ptt_activated',
  // ...
} as const
```

Compiler then catches typos and missing cases.

---

### L-5: "poc-" Prefix Signals Prototype Status in All Production Artifacts

**Status**: Open

The project is named `poc-hermes-server`. This name appears in Docker image tags, Prometheus service labels, Pino log base fields, and every document header. If this becomes production software — which the 4-phase roadmap and 28-week timeline confirms — rename to `hermes-server` before Phase 1 ships. Renaming later requires updating CI pipelines, dashboards, alert rules, and log queries. The cost today is one `find/replace`.

---

## Summary Risk Matrix

| ID | Finding | Severity | Phase to Fix |
|---|---|---|---|
| C-1 | Message delivery has unrecoverable crash window | CRITICAL | Phase 2 |
| C-2 | sync_queue stores stale payload for edited messages | CRITICAL | Phase 3 |
| C-3 | Multi-device cursor undefined for online clients | CRITICAL | Phase 3 |
| C-4 | JWT expiry during WebSocket session unhandled | CRITICAL | Phase 2 |
| C-5 | NATS migration "zero code change" claim is false | CRITICAL | Before Phase 4 design |
| H-1 | conversations.last_message_id hot-row contention | HIGH | Phase 1 |
| H-2 | 30s ACK timeout for disconnected client delivery | HIGH | Phase 2 |
| H-3 | Redis SPOF for safety-critical radio control | HIGH | Phase 1 |
| H-4 | Refresh token rotation is optional — security defect | HIGH | Phase 1 |
| H-5 | No IStorageBackend abstraction for attachments | HIGH | Phase 1 |
| H-6 | TURN credential endpoint missing from RBAC matrix | HIGH | Phase 2 |
| M-1 | sync_queue payload duplication per device | MEDIUM | Phase 3 (via C-2 fix) |
| M-2 | No cleanup policy for never-reconnecting devices | MEDIUM | Phase 3 |
| M-3 | TimescaleDB background job isolation undefined | MEDIUM | Phase 1 |
| M-4 | WsEnvelope has no version field | MEDIUM | Phase 2 |
| M-5 | RFC process has no decision authority | MEDIUM | Now |
| M-6 | mediasoup shares all CPU cores on Raspberry Pi | MEDIUM | Phase 2 |
| M-7 | Pino redact patterns miss nested payload fields | MEDIUM | Phase 1 |
| L-1 | Federation auth model deferred without ADR | LOW | Before Phase 4 |
| L-2 | WebRTC phase assignments contradict roadmap | LOW | Now |
| L-3 | localStorage referenced for mobile-targeting arch | LOW | Phase 3 |
| L-4 | Audit log action strings not enforced at compile time | LOW | Phase 1 |
| L-5 | "poc-" prefix in all production artifact names | LOW | Before Phase 1 ships |

---

## What This Architecture Gets Right

These decisions are sound and should not be relitigated:

- **Modular monolith over premature microservices** — correct for this scale and team size
- **IRadioDriver abstraction with `execFile` array args** — textbook security for CLI wrapping
- **Chat-native messaging over email model** — the right call; the email model cannot be extended
- **mediasoup SFU over Janus/Jitsi** — correct on all dimensions given the radio bridge requirement
- **Drizzle ORM over Prisma** — right for this use case; no codegen, no Rust query engine overhead on constrained hardware
- **BullMQ for delivery queues** — correct; Redis-backed persistent queues with retry semantics
- **RS256 JWT over HS256** — asymmetric signing enables future public-key verification by edge nodes
- **TimescaleDB for telemetry on the same PostgreSQL** — pragmatic and correct for the scale
- **IEventBus abstraction** — the right pattern; the semantic mismatch (C-5) is fixable without abandoning the abstraction
- **ADR quality** — all 7 ADRs are well-reasoned, specific, and document rejected alternatives

The architectural bones are solid. The 5 critical gaps are in synchronization correctness (C-1, C-2, C-3), security lifecycle (C-4, H-4), and one dangerous API semantic assumption (C-5). All are fixable before implementation begins without requiring architectural revision.
