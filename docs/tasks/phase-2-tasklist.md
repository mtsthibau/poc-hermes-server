# Phase 2 — Real-Time Infrastructure Task List

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 7–12  
**Prerequisites**: Phase 1 complete — M-1 gate passed  
**Generated**: 2026-06-10  
**Master Plan Reference**: [00-master-project-plan.md](../plan/00-master-project-plan.md)

---

## Phase Objectives (from roadmap)

1. WebSocket gateway running with typed `hermes-v1` protocol
2. Radio telemetry streaming in real-time to subscribed clients
3. Internal Redis Pub/Sub event bus wiring all modules
4. BullMQ workers for delivery, attachment processing, cleanup
5. Basic WebRTC audio sessions (same station LAN) via mediasoup
6. coturn STUN/TURN deployed alongside the server

Phase 2 does **not** include the full chat-native messaging model or sync engine.

## Audit Findings Addressed This Phase

- **[AUDIT C-1]** Missed-delivery recovery scanner — `message_deliveries` orphan rescue job
- **[AUDIT C-4]** JWT expiry during WebSocket sessions — periodic session re-validation + `SESSION_INVALIDATED`
- **[AUDIT C-5]** IEventBus interface corrected: add `subscribeExclusive()` for worker-style consumers
- **[AUDIT H-2]** WebSocket presence TOCTOU — fail-fast on `ws.send()` error; ACK timeout only for send-success case
- **[AUDIT H-6]** TURN credential endpoint added to RBAC matrix; userId embedded in HMAC; audit logged
- **[AUDIT M-4]** `WsEnvelope` gains `version: number` field; connections closed on unsupported version
- **[AUDIT M-6]** mediasoup worker count: `max(1, min(cpus-2, 4))`; `MEDIASOUP_WORKER_COUNT` env var

---

## Week 7: IEventBus & RedisEventBus (EP-06)

### [ ] Task 2.1.1: Define IEventBus interface and EventEnvelope type
**Purpose**: Establish the typed contract that all inter-module events must use — the backbone of the event-driven architecture.  
**Description**: Create `src/events/IEventBus.ts`. Define:
- `EventEnvelope<T>`: `{ id: string, topic: string, payload: T, timestamp: string, version: number }`
- `IEventBus` interface with:
  - `publish<T>(topic: string, payload: T): Promise<void>` — fan-out semantics
  - `subscribe<T>(topic: string, handler: (env: EventEnvelope<T>) => Promise<void>): Unsubscribe` — fan-out (all subscribers receive all messages)
  - `subscribeExclusive<T>(topic: string, group: string, handler: (env: EventEnvelope<T>) => Promise<void>): Unsubscribe` — **[AUDIT C-5]** competing-consumer / consumer-group semantics (only one subscriber in the group handles each message)
- `Unsubscribe`: `() => void`
Create `src/events/topics.ts` as a typed `const` record of all topic strings (`radio.telemetry`, `radio.state_changed`, `message.new`, `message.read`, `typing.started`, `typing.stopped`, `sync.queued`, etc.).

**Acceptance Criteria**:
- [ ] `IEventBus` interface compiles under strict TypeScript with no `any`
- [ ] `topics.ts` is a `as const` record — topic string literals are typed, not free strings
- [ ] JSDoc on every interface member explaining fan-out vs. exclusive semantics
- [ ] No module may import Redis directly — only `IEventBus` (enforced by ESLint `no-restricted-imports`)

**Files to Create/Edit**:
- `src/events/IEventBus.ts`
- `src/events/topics.ts`

**Doc Reference**: `docs/architecture/event-driven.md`, `docs/adr/ADR-003-event-broker.md`, Audit C-5  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None directly (internal bus)  
**Blocks**: 2.1.2, 2.2.x, 2.3.x, 2.4.x

---

### [ ] Task 2.1.2: Implement RedisEventBus
**Purpose**: Provide the Phase 1–3 event bus implementation using Redis Pub/Sub.  
**Description**: Create `src/events/RedisEventBus.ts`. Implements `IEventBus`. Uses a dedicated Redis subscriber client (`ioredis` recommended). `publish()`: `redis.publish(topic, JSON.stringify(envelope))`. `subscribe()`: `SUBSCRIBE` Redis channel; deserialize envelope; call handler. `subscribeExclusive()`: **[AUDIT C-5]** uses a Redis `LPUSH`/`BRPOP` list per group (simulating consumer-group semantics via a dedicated worker pattern) — NOT fan-out. Document that `subscribeExclusive()` semantics change on NATS migration (Phase 4). Graceful shutdown: unsubscribe all channels.

**Acceptance Criteria**:
- [ ] `publish()` + `subscribe()` unit test: published message received by all subscribers
- [ ] `subscribeExclusive()` unit test: two group members registered; each message received by exactly one member
- [ ] `subscribe()` and `subscribeExclusive()` both return an `Unsubscribe` function that stops delivery
- [ ] Redis connection error is logged at `error` level — does not crash the process

**Files to Create/Edit**:
- `src/events/RedisEventBus.ts`
- `src/events/RedisEventBus.test.ts`

**Stack Notes**: Use `ioredis` — not the `redis` npm package. Separate `ioredis` client for subscriber (cannot send commands on subscriber connection).  
**Doc Reference**: `docs/architecture/event-driven.md §3`, Audit C-5  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: None  
**Blocks**: All event-driven features (2.2.x, 2.3.x, EP-07)

---

## Week 7: WebSocket Gateway Core (EP-07)

### [ ] Task 2.2.1: WebSocket gateway — core infrastructure
**Purpose**: Open the `hermes-v1` WebSocket endpoint that all real-time features are built on.  
**Description**: Create `src/gateway/WebSocketGateway.ts`. Register `wss://host/ws` endpoint in Fastify using the `ws` library. Enforce `Sec-WebSocket-Protocol: hermes-v1` (close with `1002` if not provided). Implement: 10-second auth timeout (close `4001` if no `AUTHENTICATE`), connection registry (`Map<wsConnection, ConnectionState>`), PING every 30 s / close if no PONG within 10 s. **[AUDIT M-4]**: All outgoing `WsEnvelope` must include `version: 1`. Reject connections where client sends `version` < 1 with `4000 PROTOCOL_VERSION_UNSUPPORTED`.

**Acceptance Criteria**:
- [ ] Connecting without `hermes-v1` subprotocol closes with code `1002`
- [ ] No `AUTHENTICATE` within 10 s closes with code `4001`
- [ ] Heartbeat: server PING at 30 s → client must PONG within 10 s or connection is closed
- [ ] Connection registry tracks: `userId`, `deviceId`, `role`, `sessionId`, `subscribedTopics`, `connectedAt`
- [ ] Integration test covers: connect → auth timeout; connect → authenticate → subscribe → receive frame

**Files to Create/Edit**:
- `src/gateway/WebSocketGateway.ts`
- `src/gateway/ConnectionRegistry.ts`
- `src/gateway/WebSocketGateway.test.ts`

**Stack Notes**: Register ws server as Fastify plugin; use `ws` `Server` with `noServer: true` on the Fastify HTTP server  
**Doc Reference**: `docs/architecture/websocket-protocol.md §2, §3`, Audit M-4  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: poc-hermes-app WebSocket client must negotiate `hermes-v1` subprotocol  
**Blocks**: 2.2.2–2.2.4, 2.3.x

---

### [ ] Task 2.2.2: WebSocket authentication and session re-validation
**Purpose**: Enforce JWT authentication on every WebSocket connection and ensure revoked sessions are terminated.  
**Description**: Handle `AUTHENTICATE` message: verify JWT (RS256), set connection state. **[AUDIT C-4]**: Implement periodic session re-validation — every `SESSION_REVALIDATION_INTERVAL` ms (default: 60000), query `user_sessions` for each active connection; if `revoked_at IS NOT NULL` or `expires_at < NOW()`, send `SESSION_INVALIDATED` frame and close connection. Add `SESSION_INVALIDATED` to the `WsMessageType` catalog.

**Acceptance Criteria**:
- [ ] Valid JWT → `AUTHENTICATED` response with `{ userId, callsign, role, sessionId }`
- [ ] Expired JWT → `AUTH_ERROR` response and connection close
- [ ] Revoked session (DB query) → `SESSION_INVALIDATED` frame sent; connection closed within next revalidation window
- [ ] Revalidation does not block the event loop — uses async query
- [ ] Integration test: login → get session token → revoke session → WebSocket receives `SESSION_INVALIDATED`

**Files to Create/Edit**:
- `src/gateway/handlers/authenticate.ts`
- `src/gateway/SessionRevalidator.ts`

**Doc Reference**: `docs/architecture/security.md §2.4`, `docs/architecture/websocket-protocol.md §3`, Audit C-4  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: poc-hermes-app WebSocket client must handle `SESSION_INVALIDATED` — gracefully reconnect or prompt re-login  
**Blocks**: All WebSocket features

---

### [ ] Task 2.2.3: WebSocket SUBSCRIBE/UNSUBSCRIBE and rate limiting
**Purpose**: Allow clients to register for specific event topics and enforce rate limits.  
**Description**: Handle `SUBSCRIBE` message: validate topic list against allowed topics; update `ConnectionRegistry`; return `SUBSCRIBED { topics, grantedTopics }` (topics client lacked permission for are excluded from grantedTopics). Handle `UNSUBSCRIBE`. Per-connection rate limiting: 60 messages/min total; 10 `RADIO_COMMAND`/min. On limit exceeded: send `ERROR { code: 'RATE_LIMITED' }` and close connection after 3 violations.

**Acceptance Criteria**:
- [ ] Client subscribing to `radio.telemetry` receives `SUBSCRIBED { grantedTopics: ['radio.telemetry'] }`
- [ ] `readonly` role client subscribing to `radio.commands` receives `SUBSCRIBED { grantedTopics: [] }` (permission denied)
- [ ] Rate limit test: 61st message in 60 s receives `ERROR { code: 'RATE_LIMITED' }`
- [ ] Unit tests for topic permission matrix

**Files to Create/Edit**:
- `src/gateway/handlers/subscribe.ts`
- `src/gateway/RateLimiter.ts`

**Doc Reference**: `docs/architecture/websocket-protocol.md §2.3`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app must send `SUBSCRIBE` after `AUTHENTICATE` to receive telemetry  
**Blocks**: 2.3.x (telemetry), 2.4.x (radio command WS)

---

### [ ] Task 2.2.4: WebSocket frame dispatcher and error handling
**Purpose**: Route incoming messages to handlers and serialize responses reliably.  
**Description**: Create `src/gateway/MessageDispatcher.ts` — maps `WsEnvelope.type` to handler functions. Handle malformed JSON (close `4002 INVALID_JSON`). Handle unknown `type` (send `ERROR { code: 'UNKNOWN_MESSAGE_TYPE' }`). **[AUDIT H-2]**: On `ws.send()` error callback, immediately trigger offline fallback — do NOT start a 30-second timer. Reserve ACK-timeout fallback only for `send()`-success-but-no-ACK cases.

**Acceptance Criteria**:
- [ ] Malformed JSON frame closes connection with `4002`
- [ ] Unknown `type` returns `ERROR` (does not close connection)
- [ ] `ws.send()` error callback immediately triggers sync queue insert — no 30-second wait
- [ ] All handlers invoked with proper TypeScript types (no `unknown` without type guard)

**Files to Create/Edit**:
- `src/gateway/MessageDispatcher.ts`

**Doc Reference**: `docs/architecture/websocket-protocol.md §2.3`, Audit H-2  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Error codes must be documented so poc-hermes-app can handle them  
**Blocks**: All gateway handler tasks

---

## Week 8: Radio Telemetry Streaming (EP-08)

### [ ] Task 2.3.1: Wire TelemetryPoller to IEventBus and WebSocket gateway
**Purpose**: Stream live radio telemetry to subscribed WebSocket clients.  
**Description**: Wire `TelemetryPoller` (created in Phase 1) with `RedisEventBus`. WebSocket gateway subscribes (fan-out via `subscribe()`) to `radio.telemetry` and `radio.state_changed`. On each event: send `RADIO_TELEMETRY` frame to all connections subscribed to `radio.telemetry` topic. Only push `RADIO_STATE_CHANGE` frame on `radio.state_changed` events.

**Acceptance Criteria**:
- [ ] Client subscribing to `radio.telemetry` receives `RADIO_TELEMETRY` frames at ~1 s intervals
- [ ] Client NOT subscribed to `radio.telemetry` does NOT receive frames
- [ ] `RADIO_STATE_CHANGE` frame only sent on significant state delta (not every poll)
- [ ] Integration test: connect + subscribe → receive 3 consecutive `RADIO_TELEMETRY` frames within 4 s

**Files to Create/Edit**:
- `src/gateway/handlers/radioTelemetry.ts`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 8`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app telemetry display depends on this; frame payload field names must match expected format  
**Blocks**: None

---

### [ ] Task 2.3.2: TimescaleDB telemetry ingest worker (TelemetryIngestWorker)
**Purpose**: Durably persist radio telemetry data in TimescaleDB for historical analysis.  
**Description**: Create `src/jobs/TelemetryIngestWorker.ts` (BullMQ Worker). Queue: `telemetry:ingest`. Batch size: 60 rows per insert. Subscribe to `radio.telemetry` events (via `subscribeExclusive()` — one worker instance handles ingest). Only insert on change (deduplicate identical consecutive states). Retention policy: already configured in Phase 1; verify compression is enabled.

**Acceptance Criteria**:
- [ ] 60 telemetry events enqueued → exactly 1 batch insert of 60 rows
- [ ] Identical consecutive telemetry state does NOT insert duplicate rows
- [ ] Worker reports job completion via Prometheus `bullmq_jobs_completed_total{queue='telemetry:ingest'}`
- [ ] TimescaleDB `radio_telemetry` hypertable has data after 1 minute of operation

**Files to Create/Edit**:
- `src/jobs/TelemetryIngestWorker.ts`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 8`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None directly; enables historical telemetry charts  
**Blocks**: None

---

## Week 8–9: HAL Command via WebSocket (EP-07 continued)

### [ ] Task 2.4.1: RADIO_COMMAND WebSocket handler
**Purpose**: Allow authorized clients to control the radio via WebSocket without separate REST calls.  
**Description**: Handle `RADIO_COMMAND` message type. Validate against same JSON schemas as REST radio endpoints. Check permission: `operator`|`admin` role only (send `ERROR { code: 'FORBIDDEN' }` otherwise). Route to `RadioCommandQueue` (same shared queue as REST). Return `RADIO_COMMAND_RESULT { requestId, success, data?, error? }` correlating `requestId` from the request.

**Acceptance Criteria**:
- [ ] `user` role sending `RADIO_COMMAND` receives `ERROR { code: 'FORBIDDEN' }`
- [ ] `operator` role sending valid frequency command receives `RADIO_COMMAND_RESULT { success: true }`
- [ ] Invalid frequency in `RADIO_COMMAND` receives `RADIO_COMMAND_RESULT { success: false, error: { code: 'INVALID_PARAMETER' } }`
- [ ] `requestId` is echoed in `RADIO_COMMAND_RESULT`
- [ ] Rate limit: 10 RADIO_COMMAND/min enforced

**Files to Create/Edit**:
- `src/gateway/handlers/radioCommand.ts`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 8–9`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app may use WebSocket radio commands — frame format must be documented  
**Blocks**: None

---

## Week 9–10: BullMQ Worker Infrastructure (EP-09)

### [ ] Task 2.5.1: WorkerManager and BullMQ infrastructure
**Purpose**: Centralize worker lifecycle management for clean startup and graceful shutdown.  
**Description**: Create `src/jobs/WorkerManager.ts`. Tracks all BullMQ workers. `start()`: initializes all workers. `stop()`: `worker.close()` on each with timeout. Expose all queues in `src/jobs/queues.ts`. Register Bull Board UI at `/admin/queues` (admin-only via Fastify preHandler RBAC check). Wire worker health into `GET /health/ready`.

**Acceptance Criteria**:
- [ ] `/admin/queues` returns `401` for unauthenticated request
- [ ] `/admin/queues` returns `403` for non-admin role
- [ ] `SIGTERM` → `WorkerManager.stop()` completes within 30 s before process exits
- [ ] `/health/ready` returns `503` if any critical worker has not started

**Files to Create/Edit**:
- `src/jobs/WorkerManager.ts`
- `src/jobs/queues.ts`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 9–10`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None  
**Blocks**: All worker tasks

---

### [ ] Task 2.5.2: CleanupWorker — expired sessions and stale sync rows
**Purpose**: Prevent unbounded growth of expired session and sync queue data.  
**Description**: Create `src/jobs/CleanupWorker.ts`. Queue: `cleanup`. Schedule: every 6 hours (BullMQ `repeat`). Jobs: (1) Delete `user_sessions WHERE expires_at < NOW() AND revoked_at IS NULL` — mark expired. (2) Delete `sync_queue rows WHERE created_at < NOW() - INTERVAL '7 days'` for processed rows. **[AUDIT M-2]**: Mark `user_devices.last_seen_at` devices inactive if not seen in 90 days; stop inserting sync_queue rows for inactive devices.

**Acceptance Criteria**:
- [ ] Expired sessions are cleaned up in each run
- [ ] Devices inactive for 90 days are marked with `status='inactive'`
- [ ] `sync_queue` rows for inactive devices older than 7 days are deleted
- [ ] Prometheus metric `cleanup_jobs_total{type}` incremented per job type

**Files to Create/Edit**:
- `src/jobs/CleanupWorker.ts`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 9–10`, Audit M-2  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None  
**Blocks**: None

---

### [ ] Task 2.5.3: [AUDIT C-1] Missed-delivery recovery scanner
**Purpose**: Prevent silent message loss from process crashes between DB write and EventBus.publish().  
**Description**: Create `src/jobs/MissedDeliveryRecoveryWorker.ts`. Queue: `delivery:recovery`. Schedule: every 60 seconds. Query: `SELECT * FROM message_deliveries WHERE status='pending' AND created_at < NOW() - INTERVAL '60 seconds'`. For each found row: re-enqueue the delivery via `DeliveryService.enqueueDelivery(messageId, recipientId)`. Increment Prometheus counter `missed_delivery_recovery_total`. Alert should fire if this counter > 0 in production (documented in Grafana dashboard).

**Acceptance Criteria**:
- [ ] Integration test: write `message_delivery` row with `status=pending`, wait 65 s (or mock time), scanner finds and re-enqueues it
- [ ] Prometheus metric `missed_delivery_recovery_total` incremented for each rescued delivery
- [ ] Grafana dashboard panel "Missed Delivery Recoveries" shows metric with alert threshold at > 0
- [ ] Scanner does NOT re-enqueue rows with `status` other than `pending`

**Files to Create/Edit**:
- `src/jobs/MissedDeliveryRecoveryWorker.ts`

**Doc Reference**: Audit C-1  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Ensures messages sent to poc-hermes-app users are not silently lost on server restart  
**Blocks**: None

---

### [ ] Task 2.5.4: ScheduleRunnerWorker
**Purpose**: Execute connection schedules (UUCP call windows) at configured times.  
**Description**: Create `src/jobs/ScheduleRunnerWorker.ts`. Loads `connection_schedules` from DB. For each active schedule: enqueue a UUCP connection job at the specified time via BullMQ `delayed`. On execution: call `SystemService.triggerUucpConnection(scheduleId)`.

**Acceptance Criteria**:
- [ ] Creating a schedule with `POST /api/v1/caller` results in a BullMQ delayed job
- [ ] Deleting a schedule removes the corresponding BullMQ job
- [ ] Worker logs schedule execution result via Pino

**Files to Create/Edit**:
- `src/jobs/ScheduleRunnerWorker.ts`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 9–10`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app schedule management UI depends on this  
**Blocks**: None

---

## Week 10–11: WebRTC (EP-10)

### [ ] Task 2.6.1: mediasoup server setup and session manager
**Purpose**: Enable browser-based voice communication between station users.  
**Description**: Install `mediasoup`. Create `src/modules/webrtc/MediasoupManager.ts`. Worker pool: **[AUDIT M-6]** `numWorkers = Math.max(1, Math.min(os.cpus().length - 2, 4))`, configurable via `MEDIASOUP_WORKER_COUNT`. Router per session with Opus codec (`{ mimeType: 'audio/opus', clockRate: 48000, channels: 2 }`, FEC, DTX enabled). `WebRtcSessionManager`: `createSession()`, `destroySession()`, `addTransport()`, `addProducer()`, `addConsumer()`. Wire signaling to WebSocket gateway under `webrtc.*` topics.

**Acceptance Criteria**:
- [ ] mediasoup workers start without error (`npm run dev`)
- [ ] `WEBRTC_CREATE_SESSION` WebSocket message → session created → `WEBRTC_SESSION_CREATED` response
- [ ] `WEBRTC_END_SESSION` → session destroyed, resources freed
- [ ] Worker count is `min(cpus-2, 4)` (verified via log output at startup)
- [ ] Unit test: session create/destroy lifecycle

**Files to Create/Edit**:
- `src/modules/webrtc/MediasoupManager.ts`
- `src/modules/webrtc/WebRtcSessionManager.ts`

**Stack Notes**: mediasoup v3; use `mediasoup.createWorker()` per worker in pool  
**Doc Reference**: `docs/architecture/webrtc.md`, `docs/roadmap/phase-2-realtime.md §Week 10–11`, Audit M-6  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Enables audio call feature in poc-hermes-app; signaling frame types must be documented  
**Blocks**: 2.6.2

---

### [ ] Task 2.6.2: coturn deployment and TURN credential endpoint
**Purpose**: Enable STUN/TURN relay for WebRTC NAT traversal.  
**Description**: Add `coturn` service to `docker-compose.yml` with: `denied-peer-ip` for all RFC 1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), HMAC secret from env. **[AUDIT H-6]**: Implement `GET /api/v1/webrtc/credentials` requiring minimum `user` role. Response: time-limited TURN credentials (HMAC-SHA1, TTL 300 s). Embed `userId` in the HMAC credential username for auditability. Write `audit_logs` entry (`action: 'webrtc.credentials_issued'`) on every call. Add rate limit (10 req/min/user).

**Acceptance Criteria**:
- [ ] `GET /webrtc/credentials` with `readonly` role returns `403`
- [ ] `GET /webrtc/credentials` with `user` role returns `{ username, credential, ttl, uris }`
- [ ] coturn validates credentials correctly (HMAC-SHA1 match)
- [ ] `audit_logs` contains `webrtc.credentials_issued` entry with `actorId`
- [ ] `denied-peer-ip` blocks access to localhost (tested via coturn config validation)

**Files to Create/Edit**:
- `docker-compose.yml` (add coturn service)
- `src/api/routes/webrtc.ts`

**Doc Reference**: `docs/architecture/webrtc.md`, Audit H-6  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app WebRTC client needs TURN credentials from this endpoint  
**Blocks**: None

---

## Week 11–12: Observability (EP-11)

### [ ] Task 2.7.1: Prometheus metrics — all key subsystems
**Purpose**: Make the system observable and alertable in production.  
**Description**: Install `prom-client`. Create `src/shared/metrics.ts` with all metrics from `docs/architecture/observability.md §3`. Register metrics in Fastify at `GET /metrics` (internal — NOT proxied through nginx to public internet). Add `ws_client_protocol_version` label to WebSocket metrics. Add `missed_delivery_recovery_total` counter. Add `auth_refresh_token_reuse_detected_total` counter. Add `radio.command_queue.degraded` gauge.

**Acceptance Criteria**:
- [ ] `GET /metrics` returns Prometheus text format
- [ ] HTTP request metrics are populated after a test request
- [ ] WebSocket connection gauge increments on connect, decrements on disconnect
- [ ] `GET /metrics` is NOT accessible via the nginx public proxy (verified in integration test)

**Files to Create/Edit**:
- `src/shared/metrics.ts`
- `src/api/routes/metrics.ts`

**Doc Reference**: `docs/architecture/observability.md §3`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None directly  
**Blocks**: None

---

### [ ] Task 2.7.2: OpenTelemetry distributed tracing
**Purpose**: Enable end-to-end request tracing for debugging latency issues.  
**Description**: Create `src/instrumentation.ts` (loaded before all other code via `--require`). Configure `@opentelemetry/sdk-node` with auto-instrumentation for Fastify, `pg`, `ioredis`, `bullmq`. Export traces to OTEL collector (or stdout in dev). Propagate `requestId` as trace attribute.

**Acceptance Criteria**:
- [ ] Server starts with instrumentation loaded (`NODE_OPTIONS='--require ./dist/instrumentation.js'`)
- [ ] HTTP request spans visible in trace output
- [ ] `requestId` appears as span attribute

**Files to Create/Edit**:
- `src/instrumentation.ts`

**Doc Reference**: `docs/architecture/observability.md §4`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None  
**Blocks**: None

---

### [ ] Task 2.7.3: Grafana dashboard provisioning
**Purpose**: Provide out-of-the-box operational dashboards for station operators.  
**Description**: Create `infrastructure/observability/dashboards/hermes-main.json`. Panels: Radio Status (SNR, bitrate, TX/RX), API Performance (req rate, p50/p95/p99, error rate), WebSocket (active connections, message throughput, auth failures), Message Delivery (rate, latency, failures), Queue Health (depth, processing rate, failed jobs), System (CPU, memory, event loop lag). Add "Missed Delivery Recoveries" panel with alert.

**Acceptance Criteria**:
- [ ] `docker compose up` (with Grafana service added) loads dashboards automatically via provisioning
- [ ] "Missed Delivery Recoveries" panel shows `missed_delivery_recovery_total` with threshold annotation at > 0
- [ ] All panels have data after 5 minutes of operation

**Files to Create/Edit**:
- `infrastructure/observability/dashboards/hermes-main.json`
- `infrastructure/observability/prometheus.yml`
- `docker-compose.yml` (add Prometheus + Grafana services)

**Doc Reference**: `docs/architecture/observability.md §3`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None  
**Blocks**: None

---

## Phase 2 Compatibility Verification Gate — Checkpoint

### [ ] Task 2.8.1: Run poc-hermes-app against Phase 2 server
**Purpose**: Verify zero regressions for existing poc-hermes-app functionality after adding real-time layer.  
**Description**: Deploy Phase 2 server. Run full Phase 1 smoke test (all REST endpoints). Additionally test: WebSocket telemetry subscription (poc-hermes-app must negotiate `hermes-v1`), radio command via WebSocket, TURN credential fetch. Produce compatibility report at `docs/qa/compat-phase-2.md`.  
**Acceptance Criteria**:
- [ ] All Phase 1 smoke tests still pass
- [ ] poc-hermes-app successfully connects WebSocket with `hermes-v1` subprotocol
- [ ] poc-hermes-app receives `RADIO_TELEMETRY` frames after subscribing
- [ ] `SESSION_INVALIDATED` message type handled by poc-hermes-app (reconnect flow tested)
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] All Phase 2 audit findings marked resolved
- [ ] Compatibility report signed off by Engineering Lead

**Files to Create/Edit**:
- `docs/qa/compat-phase-2.md`

**Blocks**: Phase 3 start

---

## Phase 2 Quality Gates

- [ ] `npm run build` passes with zero TypeScript errors
- [ ] `npm test` passes with ≥ 80% coverage on all new modules
- [ ] `docker compose up` starts all services including coturn
- [ ] `GET /health/ready` reflects all worker statuses
- [ ] `/metrics` endpoint populated and not publicly exposed
- [ ] Pino logger used throughout — no `console.log`
- [ ] All environment variables declared in `src/shared/config.ts`
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] poc-hermes-app compatibility report approved
