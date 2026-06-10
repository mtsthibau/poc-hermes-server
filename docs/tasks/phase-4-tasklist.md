# Phase 4 — Federation & Distribution Task List

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 21–28  
**Prerequisites**: Phase 3 complete and stable in production — M-3 gate passed  
**Generated**: 2026-06-10  
**Master Plan Reference**: [00-master-project-plan.md](../plan/00-master-project-plan.md)

---

## Phase Objectives (from roadmap)

1. Multiple HERMES stations exchange messages via internet or radio bridge
2. Internal event bus migrated from Redis Pub/Sub to NATS JetStream
3. Radio-to-WebRTC bridge: HF radio audio streamed to WebRTC clients
4. Group WebRTC audio calls (multiple participants)
5. Station discovery and presence (which stations are reachable)
6. End-to-end encryption consideration (ADR, not necessarily v1 implementation)

## Audit Findings Addressed This Phase

- **[AUDIT C-5]** NATS migration: `IEventBus.subscribeExclusive()` used for all worker-style subscribers; behavioral semantic difference documented; ADR-003 corrected
- **[AUDIT L-1]** ADR-008 drafted and decided BEFORE Phase 4 schema design begins (station-to-station auth model)
- **[AUDIT L-2]** Phase assignment reconciled: radio audio bridge is Phase 4 (not Phase 3); webrtc.md §2 updated
- **[AUDIT TD-01]** Legacy messaging shim removed (deprecated since Phase 1; removed here)

## Pre-Phase 4 Prerequisites (must complete before Week 21)

### [ ] Pre-4.1: Draft and decide ADR-008 — Federation Authentication Model
**Purpose**: Define station-to-station identity before designing schema that requires credential fields.  
**Description**: Draft `docs/adr/ADR-008-federation-auth.md`. Options to evaluate: (A) shared secret per station pair, (B) PKI with per-station certificate, (C) HMAC with rotating keys. Decision must specify: credential storage in `stations` table, rotation policy, revocation mechanism. Follow RFC process from `docs/governance/RFC-TEMPLATE.md`.  
**Acceptance Criteria**:
- [ ] ADR-008 drafted, reviewed, and accepted (per governance process)
- [ ] `stations` table schema in Phase 4 uses credential fields determined by ADR-008
- [ ] ADR-008 status: Accepted, not Proposed

**Files to Create/Edit**:
- `docs/adr/ADR-008-federation-auth.md`

**Doc Reference**: Audit L-1, `docs/governance/RFC-TEMPLATE.md`  
**Blocks**: 4.1.x (station registry schema)

---

### [ ] Pre-4.2: Correct ADR-003 and update IEventBus documentation
**Purpose**: Formally record the C-5 fix — NATS migration is NOT a zero-code-change operation.  
**Description**: Update `docs/adr/ADR-003-event-broker.md` section on NATS migration. Remove the claim "zero application code change." Document: `subscribe()` = fan-out (WebSocket gateway, telemetry); `subscribeExclusive()` = consumer-group (delivery workers, sync workers). Explain why both are needed for NATS migration. Update `src/events/IEventBus.ts` JSDoc to reflect this.  
**Acceptance Criteria**:
- [ ] ADR-003 no longer claims "zero code change" for NATS migration
- [ ] Document lists which subscribers use `subscribe()` and which use `subscribeExclusive()`
- [ ] `IEventBus.ts` JSDoc updated with concrete examples

**Files to Create/Edit**:
- `docs/adr/ADR-003-event-broker.md`
- `src/events/IEventBus.ts` (JSDoc update)

**Doc Reference**: Audit C-5  
**Blocks**: 4.1.x (NATS implementation)

---

### [ ] Pre-4.3: Update webrtc.md §2 — reconcile phase assignment
**Purpose**: Remove the L-2 documentation inconsistency before Phase 4 begins.  
**Description**: In `docs/architecture/webrtc.md §2`, update use case table: "Radio audio streamed to browser (receive)" → Phase 4 (not Phase 3). Add note explaining the radio-to-WebRTC bridge is Phase 4 and why (requires ffmpeg + ALSA + PlainTransport infrastructure not in Phase 2 scope).  
**Acceptance Criteria**:
- [ ] webrtc.md §2 use case table reflects Phase 4 assignment for radio audio bridge
- [ ] No document claims radio audio bridge is Phase 3

**Files to Create/Edit**:
- `docs/architecture/webrtc.md`

**Doc Reference**: Audit L-2  
**Blocks**: None

---

## Week 21–22: NATS JetStream Migration (EP-19)

### [ ] Task 4.1.1: Deploy NATS JetStream and implement NatsEventBus
**Purpose**: Upgrade the event bus to a durable, federation-capable message broker.  
**Description**: Add `nats` service to `docker-compose.yml` with JetStream enabled. Create `src/events/NatsEventBus.ts` implementing `IEventBus`:
- `publish()`: `nats.publish(subject, JSON.stringify(envelope))`
- `subscribe()`: fan-out via JetStream push consumer with **unique per-instance** consumer name (to ensure all instances receive every message)
- `subscribeExclusive()`: **[AUDIT C-5]** use JetStream pull consumer with shared consumer group name (competing consumers — exactly one handler per message)

Select implementation via `EVENT_BUS_DRIVER` env var (`redis` | `nats`). Wire in `src/index.ts` via factory function.

**NATS Streams to configure**:
```
hermes.radio        subjects: radio.>    retention: 24h, WorkQueue policy for exclusive
hermes.messaging    subjects: message.>  retention: 7d
hermes.sync         subjects: sync.>     retention: 7d
hermes.presence     subjects: presence.> retention: 1h
hermes.federation   subjects: station.>  retention: 30d
```

**Acceptance Criteria**:
- [ ] `EVENT_BUS_DRIVER=nats` → server starts using NATS for all events
- [ ] Fan-out test: 3 server instances subscribe to `radio.telemetry`; each message received by all 3 (fan-out via push consumer)
- [ ] Exclusive test: 3 server instances `subscribeExclusive` to `message.new`; each message received by exactly 1 of the 3 (competing consumer via pull consumer group)
- [ ] Smoke test: all Phase 1–3 event flows work correctly with NATS

**Files to Create/Edit**:
- `docker-compose.yml` (add nats service)
- `src/events/NatsEventBus.ts`
- `src/events/NatsEventBus.test.ts`
- `src/index.ts` (factory switch)

**Stack Notes**: Use `nats.ws` npm package for Node.js NATS client  
**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 21–22`, Audit C-5  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Transparent to app — same `IEventBus` interface; no API changes  
**Blocks**: 4.2.x (federation), 4.3.x (radio bridge)

---

### [ ] Task 4.1.2: NATS migration smoke test and Redis Pub/Sub decommission
**Purpose**: Validate NATS as a drop-in replacement for Redis Pub/Sub, then remove the Redis Pub/Sub subscriber path.  
**Description**: Run full Phase 1–3 integration test suite with `EVENT_BUS_DRIVER=nats`. Fix any failures. After all tests pass: remove `RedisEventBus` subscriber code (retain `RedisEventBus.publish()` for backward compat during transition if needed, but mark deprecated). Retain Redis for: BullMQ, session store, rate limiting, cache. Document migration in `docs/adr/ADR-003-event-broker.md` as "Migrated to NATS JetStream in Phase 4."

**Acceptance Criteria**:
- [ ] All Phase 1–3 integration tests pass with `EVENT_BUS_DRIVER=nats`
- [ ] Redis Pub/Sub subscriber code removed from `RedisEventBus.ts`
- [ ] ADR-003 updated with migration completion note
- [ ] Redis still used for BullMQ, sessions, rate limiting — no Redis removal

**Files to Create/Edit**:
- `src/events/RedisEventBus.ts` (remove subscriber code)
- `docs/adr/ADR-003-event-broker.md` (update)

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 21–22`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None — transparent migration  
**Blocks**: 4.2.x

---

## Week 22–23: Multi-Station Synchronization (EP-20)

### [ ] Task 4.2.1: Station registry — schema and REST API
**Purpose**: Track known remote stations and their connectivity status.  
**Description**: Add `stations` table to Drizzle schema: `callsign`, `display_name`, `last_seen_at`, `ip_address`, `connectivity_type` (`internet` | `radio`), credential fields (per ADR-008 decision). Create migration. Implement REST API:
- `POST /api/v1/federation/stations` — register remote station (admin)
- `GET /api/v1/federation/stations` — list known stations with online status (operator+)
- `DELETE /api/v1/federation/stations/:callsign` — deregister (admin)

Station presence: subscribe to NATS `station.<callsign>.heartbeat` — update `last_seen_at` on each heartbeat. Station is "online" if `last_seen_at > NOW() - INTERVAL '60 seconds'`.

**Acceptance Criteria**:
- [ ] `GET /federation/stations` returns list with `isOnline` boolean per station
- [ ] Station heartbeat subscription updates `last_seen_at` in DB
- [ ] `isOnline` = false if no heartbeat within 60 s (tested by stopping heartbeat publisher in test)
- [ ] Credential fields match ADR-008 decision

**Files to Create/Edit**:
- `src/db/schema/federation.ts`
- `src/db/migrations/0005_federation.sql`
- `src/api/routes/federation.ts`
- `src/modules/federation/StationRegistry.ts`

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 22–23`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: poc-hermes-app may surface station list view; online status indicator  
**Blocks**: 4.2.2 (cross-station delivery)

---

### [ ] Task 4.2.2: Cross-station message delivery — DeliveryRouter and FederationDeliveryWorker
**Purpose**: Route messages to users on remote stations.  
**Description**: Create `src/modules/federation/DeliveryRouter.ts`. Determines if a recipient is on the local station or a remote one (based on user's station assignment). For remote recipients: create `FederationDeliveryWorker` job. Create `src/jobs/FederationDeliveryWorker.ts` (BullMQ, uses `subscribeExclusive`):
- Publish `station.<targetCallsign>.message.new` to NATS JetStream
- Remote station subscribes to `station.<ownCallsign>.>` on its NATS leaf node
- Receiving station creates message in local conversation
- Send acknowledgment: publish `station.<sourceCallsign>.delivery.<targetCallsign>.ack`

For internet-unavailable stations: fall back to `RadioDeliveryWorker` (Phase 3 UUCP path).

**Acceptance Criteria**:
- [ ] Message to user on remote station is published to correct NATS subject
- [ ] Receiving station creates message in its local DB
- [ ] Delivery acknowledgment updates `message_deliveries.status = 'delivered'` on sending station
- [ ] Integration test: two-station test using two local server instances

**Files to Create/Edit**:
- `src/modules/federation/DeliveryRouter.ts`
- `src/jobs/FederationDeliveryWorker.ts`

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 22–23`  
**Estimated Complexity**: Very High  
**poc-hermes-app Integration Impact**: poc-hermes-app users on federated stations can receive messages; delivery indicators work cross-station  
**Blocks**: 4.2.3

---

### [ ] Task 4.2.3: Station heartbeat publisher
**Purpose**: Advertise this station's presence to the federation.  
**Description**: Create `src/modules/federation/StationHeartbeat.ts`. Publish `station.<localCallsign>.heartbeat { ip, version, capabilities }` to NATS every 30 s (via `IEventBus.publish()`). Start at server startup; stop on graceful shutdown. Add `station.heartbeat_sent_total` Prometheus counter.

**Acceptance Criteria**:
- [ ] Heartbeat published every 30 s (verified in integration test with mock time)
- [ ] Heartbeat stopped on `SIGTERM`
- [ ] Remote station's `GET /federation/stations` shows this station as `isOnline: true` within 60 s

**Files to Create/Edit**:
- `src/modules/federation/StationHeartbeat.ts`

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 22–23`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: Station online status in poc-hermes-app federation view  
**Blocks**: None

---

## Week 23–25: Radio-to-WebRTC Bridge (EP-21)

### [ ] Task 4.3.1: RadioAudioBridge — ALSA capture to mediasoup
**Purpose**: Allow browser clients to hear HF radio audio in real-time.  
**Description**: Create `src/modules/webrtc/RadioAudioBridge.ts`. Architecture:
```
sBitx ALSA output → ffmpeg (Opus RTP encode) → mediasoup PlainTransport
  → mediasoup Router → WebRTC consumers (browser clients)
```
Implementation:
- Start ffmpeg child process: `ffmpeg -f alsa -i <device> -c:a libopus -ar 48000 -ac 2 -b:a 48k -ssrc <ssrc> -f rtp rtp://127.0.0.1:<port>`
- Create mediasoup `PlainTransport` to receive RTP from ffmpeg
- Create mediasoup `Producer` from PlainTransport
- Distribute audio to all WebRTC consumers in the session
- On browser TX: reverse path + HAL `setPTT(true)` via `RadioCommandQueue`

**TX turn-taking protocol**: Only one WebRTC client can TX at a time. `RadioAudioBridge` maintains a TX queue. PTT activated only after current TX ends or timeout.

**Acceptance Criteria**:
- [ ] `RadioAudioBridge.start()` spawns ffmpeg and creates mediasoup Producer without error
- [ ] Browser client connecting to radio session receives audio stream (WebRTC consumer created)
- [ ] `setPTT(true)` only called when a WebRTC client is actively transmitting
- [ ] TX duration limit enforced (default 60 s; configurable via `RADIO_TX_MAX_DURATION_S`)
- [ ] Only `operator`|`admin` roles can activate TX via WebRTC bridge

**Files to Create/Edit**:
- `src/modules/webrtc/RadioAudioBridge.ts`
- `src/modules/webrtc/TxQueue.ts`

**Stack Notes**: ffmpeg must be available in Docker container (add to Dockerfile)  
**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 23–25`, `docs/architecture/webrtc.md`  
**Estimated Complexity**: Very High  
**poc-hermes-app Integration Impact**: New capability — poc-hermes-app radio audio UI depends on this; new WebSocket message types documented  
**Blocks**: 4.4.1

---

## Week 25–26: Group WebRTC Calls (EP-22)

### [ ] Task 4.4.1: Group WebRTC audio — multi-participant sessions
**Purpose**: Enable multiple users to join a single voice session (linked to a group conversation).  
**Description**: Enhance `WebRtcSessionManager` to support multiple participants. All-to-all audio mixing configuration (all participants hear each other). Active speaker detection via mediasoup `activeSpeaker` event — publish `speaker.changed` event on IEventBus. Participant roster events: `WEBRTC_PARTICIPANT_JOINED`, `WEBRTC_PARTICIPANT_LEFT` pushed via WebSocket to all session participants. Link WebRTC session to a `group` conversation of type `radio`.

**Acceptance Criteria**:
- [ ] 3 participants can join a session and hear each other
- [ ] `WEBRTC_PARTICIPANT_JOINED` sent to all existing participants when a new one joins
- [ ] Active speaker detection fires `speaker.changed` event within 500 ms of speech start
- [ ] Session linked to `radio` conversation; participant joins/leaves logged as conversation events
- [ ] mediasoup `activeSpeaker` event is forwarded as WebSocket frame to all participants

**Files to Create/Edit**:
- `src/modules/webrtc/WebRtcSessionManager.ts` (update)
- `src/gateway/handlers/webrtcGroupSession.ts`

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 25–26`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Group voice call UI in poc-hermes-app depends on participant roster events  
**Blocks**: None

---

### [ ] Task 4.4.2: E2EE consideration — ADR and documented tradeoff
**Purpose**: Make an explicit architectural decision about end-to-end encryption for WebRTC.  
**Description**: Draft ADR-009: "WebRTC End-to-End Encryption Decision." Evaluate Insertable Streams (Chrome/Edge WebRTC E2EE API). Document the fundamental tradeoff: **E2EE breaks the radio-to-WebRTC bridge** because audio must be decoded server-side by mediasoup to re-encode for RTP. Decision: E2EE optional, not default — radio bridge takes priority. Document in ADR and in `docs/architecture/webrtc.md §E2EE`.

**Acceptance Criteria**:
- [ ] ADR-009 drafted and accepted
- [ ] webrtc.md contains a clear "E2EE Tradeoffs" section
- [ ] ADR explicitly states that radio bridge + E2EE are mutually exclusive at the mediasoup architecture level

**Files to Create/Edit**:
- `docs/adr/ADR-009-webrtc-e2ee.md`
- `docs/architecture/webrtc.md` (add E2EE section)

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 25–26`  
**Estimated Complexity**: Low (documentation)  
**poc-hermes-app Integration Impact**: None (deferred feature)  
**Blocks**: None

---

## Week 26–27: Security Hardening for Federation

### [ ] Task 4.5.1: Federation message authentication
**Purpose**: Prevent unauthorized stations from injecting messages into the federation.  
**Description**: Implement station authentication per ADR-008 decision. All NATS messages published to `station.>` subjects must be authenticated (e.g., HMAC signature in envelope headers, or NATS credentials-based auth per station). Receiving station must verify the signature before accepting the message. Unauthenticated messages logged and discarded.

**Acceptance Criteria**:
- [ ] Unsigned `station.PY2XA.message.new` published from an unauthorized NATS client is rejected by the receiving station
- [ ] Tampered message (body modified, signature invalid) is rejected
- [ ] Authentication failure logged at `error` level with source subject

**Files to Create/Edit**:
- `src/modules/federation/FederationAuthenticator.ts`

**Doc Reference**: `docs/roadmap/phase-4-federation.md §Week 26–27`, ADR-008  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: None (inter-station security)  
**Blocks**: None

---

### [ ] Task 4.5.2: MinIO storage backend for federated attachments
**Purpose**: Enable attachments to be served to users on remote stations.  
**Description**: Create `src/modules/storage/MinioStorageBackend.ts` implementing `IStorageBackend` (defined in Phase 1). Select via `STORAGE_BACKEND=minio` env var. Configure MinIO endpoint, bucket, credentials from env via `src/shared/config.ts`. `getSignedUrl()` generates time-limited pre-signed URLs. Update attachment access logic: for remote-station users, return signed URL instead of local file path.

**Acceptance Criteria**:
- [ ] `STORAGE_BACKEND=minio` → all attachment writes go to MinIO
- [ ] `STORAGE_BACKEND=local` → still uses `LocalFileStorageBackend` (backward compat)
- [ ] Remote station user can download attachment via signed URL
- [ ] Signed URL expires after configured TTL (default: 3600 s)
- [ ] `IStorageBackend` interface unchanged — no callers modified

**Files to Create/Edit**:
- `src/modules/storage/MinioStorageBackend.ts`
- `src/index.ts` (storage backend factory)
- `docker-compose.yml` (add minio service for development)

**Doc Reference**: Audit H-5, `docs/roadmap/phase-4-federation.md`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Attachment URLs may change from local paths to MinIO signed URLs; client must follow redirects  
**Blocks**: None

---

### [ ] Task 4.5.3: Remove legacy messaging shim
**Purpose**: Complete the deprecation cycle started in Phase 1.  
**Description**: Remove `src/api/routes/messagingShim.ts` and all `/api/v1/message/` routes. Verify poc-hermes-app has been updated to use the new conversation/message endpoints (this must be coordinated with poc-hermes-app team). Add to OpenAPI spec a note on the removal in the changelog. Update API version to reflect breaking change.

**Acceptance Criteria**:
- [ ] `GET /api/v1/message/inbox` returns `404`
- [ ] `POST /api/v1/message/send` returns `404`
- [ ] poc-hermes-app updated to use `/api/v1/conversations` and `/api/v1/conversations/:id/messages`
- [ ] Compatibility report explicitly verifies poc-hermes-app is NOT using the shim

**Files to Create/Edit**:
- `src/api/routes/messagingShim.ts` (delete)
- `docs/qa/compat-phase-4.md` (document shim removal)

**Doc Reference**: Technical Debt TD-01  
**Estimated Complexity**: Low (code removal + coordination)  
**poc-hermes-app Integration Impact**: **Breaking change** — poc-hermes-app MUST be updated before this task; coordinate timing  
**Blocks**: M-4 gate

---

## Week 27–28: Final Integration and Federation Testing

### [ ] Task 4.6.1: Multi-station integration test suite
**Purpose**: Validate the complete federation workflow end-to-end.  
**Description**: Create integration tests using two local server instances representing station PY2XA and PY8EME:
- Message from PY2XA user to PY8EME user → delivered via NATS → appears in PY8EME local DB
- Delivery acknowledgment from PY8EME → PY2XA delivery status updates
- Station heartbeat → PY2XA visible as online in PY8EME's station list
- Radio-only station: message falls back to UUCP path

**Acceptance Criteria**:
- [ ] Cross-station message delivery works end-to-end
- [ ] Delivery status tracked correctly on sending station
- [ ] Station discovery works (heartbeat → online status)
- [ ] UUCP fallback path tested with mock radio-only station

**Files to Create/Edit**:
- `src/__tests__/phase4-federation.test.ts`

**Doc Reference**: `docs/roadmap/phase-4-federation.md`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Federation features only valuable if poc-hermes-app surfaces them correctly  
**Blocks**: M-4 gate

---

## Phase 4 Compatibility Verification Gate — Checkpoint

### [ ] Task 4.7.1: Run poc-hermes-app against Phase 4 server
**Purpose**: Verify full system compatibility after NATS migration and federation additions.  
**Description**: Deploy Phase 4 server with NATS. Run full Phase 1 + Phase 2 + Phase 3 smoke test. Additionally test: legacy shim is removed (poc-hermes-app uses new endpoints), federation station list visible, cross-station message delivery (if second station available in test environment). Produce compatibility report at `docs/qa/compat-phase-4.md`.  
**Acceptance Criteria**:
- [ ] All Phase 1, 2, 3 smoke tests pass (except shim — confirmed poc-hermes-app uses new endpoints)
- [ ] poc-hermes-app does NOT use `/api/v1/message/` endpoints (confirmed removed)
- [ ] NATS migration transparent to poc-hermes-app (no user-visible behavior change)
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] All Phase 4 audit findings marked resolved
- [ ] Compatibility report signed off by Engineering Lead, QA, and Radio Operator SME

**Files to Create/Edit**:
- `docs/qa/compat-phase-4.md`

**Blocks**: v2.0.0 release

---

## Phase 4 Quality Gates

- [ ] `npm run build` passes with zero TypeScript errors
- [ ] `npm test` passes with ≥ 80% coverage on all new modules
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] Multi-station message delivery working end-to-end
- [ ] NATS JetStream event bus running; Redis Pub/Sub subscriber code removed
- [ ] Radio-to-WebRTC bridge streaming audio to browser clients
- [ ] Group WebRTC calls operational (3+ participants)
- [ ] Station discovery and presence tracking operational
- [ ] ADR-008 (federation auth) accepted and implemented
- [ ] ADR-009 (E2EE decision) accepted and documented
- [ ] Legacy messaging shim removed
- [ ] MinIO storage backend available (if `STORAGE_BACKEND=minio`)
- [ ] poc-hermes-app compatibility report approved
