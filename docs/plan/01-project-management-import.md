# Project Management Import — poc-hermes-server
# Format: Compatible with GitHub Projects, Jira CSV import, and Azure DevOps
# Generated: 2026-06-10

## Instructions for Import

### GitHub Projects
Import each Epic as a Milestone. Import each Task as an Issue with the label matching the Epic, priority, and phase fields as custom fields.

### Jira CSV Import
Use the field mapping:
- `ID` → Issue Key
- `Summary` → Summary
- `Type` → Issue Type (Epic / Story / Task / Sub-task)
- `Priority` → Priority
- `Phase` → Sprint (map phases to sprints)
- `Status` → Status
- `Blocks` → Linked Issue (blocks relationship)
- `AcceptanceCriteria` → Description / Acceptance Criteria field
- `Component` → Component

---

## Milestones

| ID | Title | Target Week | Description |
|---|---|---|---|
| M-1 | Foundation Complete | W6 | All hermes-api endpoints replaced; HAL operational; JWT auth; CI passing; poc-hermes-app gate |
| M-2 | Real-Time Infrastructure Live | W12 | WebSocket gateway; radio telemetry streaming; BullMQ workers; WebRTC LAN audio; poc-hermes-app gate |
| M-3 | Full Messaging Operational | W20 | Chat model; delivery pipeline; offline sync; attachments; HMP interop; poc-hermes-app gate |
| M-4 | Federation Ready | W28 | NATS migration; multi-station delivery; radio bridge; group calls; poc-hermes-app gate |

---

## Epics

| ID | Epic | Phase | Priority | Milestone |
|---|---|---|---|---|
| EP-01 | Project Scaffold & Infrastructure | 1 | Critical | M-1 |
| EP-02 | Database Schema & Repository Layer | 1 | Critical | M-1 |
| EP-03 | JWT Authentication & RBAC | 1 | Critical | M-1 |
| EP-04 | Hardware Abstraction Layer (HAL) | 1 | Critical | M-1 |
| EP-05 | REST API — All hermes-api Endpoints | 1 | Critical | M-1 |
| EP-06 | IEventBus & RedisEventBus | 2 | Critical | M-2 |
| EP-07 | WebSocket Gateway (hermes-v1) | 2 | Critical | M-2 |
| EP-08 | Radio Telemetry Streaming | 2 | High | M-2 |
| EP-09 | BullMQ Worker Infrastructure | 2 | High | M-2 |
| EP-10 | WebRTC LAN Audio | 2 | High | M-2 |
| EP-11 | Observability (Prometheus, OTel, Grafana) | 2 | High | M-2 |
| EP-12 | Messaging Domain Model & REST API | 3 | Critical | M-3 |
| EP-13 | Delivery Pipeline (Online + Offline) | 3 | Critical | M-3 |
| EP-14 | [AUDIT C-1] Missed Delivery Recovery Scanner | 2 | Critical | M-2 |
| EP-15 | [AUDIT C-2/C-3] Sync Engine Correctness | 3 | Critical | M-3 |
| EP-16 | Attachment Pipeline | 3 | High | M-3 |
| EP-17 | Reactions, Threading, Typing Indicators | 3 | High | M-3 |
| EP-18 | HMP/UUCP Interoperability | 3 | High | M-3 |
| EP-19 | NATS JetStream Migration | 4 | Critical | M-4 |
| EP-20 | Multi-Station Federation | 4 | Critical | M-4 |
| EP-21 | Radio-to-WebRTC Bridge | 4 | High | M-4 |
| EP-22 | Group WebRTC Calls | 4 | High | M-4 |
| EP-23 | Security Hardening — Federation | 4 | Critical | M-4 |

---

## Tasks — Phase 1

| ID | Title | Type | Epic | Priority | Phase | Week | Complexity | Blocks | IntegrationImpact |
|---|---|---|---|---|---|---|---|---|---|
| T-1.1.1 | Initialize Node.js 22 + TypeScript 5 project | Task | EP-01 | Critical | 1 | W1 | Low | T-1.1.4 | None |
| T-1.1.2 | Configure ESLint + Prettier + commitlint | Task | EP-01 | High | 1 | W1 | Low | None | None |
| T-1.1.3 | Create Docker Compose — PG17 + TimescaleDB + Redis | Task | EP-01 | Critical | 1 | W1 | Low | T-1.2.x | None |
| T-1.1.4 | Fastify application skeleton + health check | Task | EP-01 | Critical | 1 | W1 | Low | T-1.5.x | None |
| T-1.1.5 | Configure Pino logger with security redaction [AUDIT M-7] | Task | EP-01 | High | 1 | W1 | Low | None | None |
| T-1.1.6 | Set up Vitest + CI pipeline | Task | EP-01 | High | 1 | W2 | Low | None | None |
| T-1.2.1 | Drizzle ORM configuration + migration tooling | Task | EP-02 | Critical | 1 | W2 | Low | T-1.2.2-8 | None |
| T-1.2.2 | Drizzle schema — Identity & Auth tables | Task | EP-02 | Critical | 1 | W2 | Medium | T-1.3.x | Callsign must match poc-hermes-app login |
| T-1.2.3 | Drizzle schema — Conversations & Messaging [AUDIT H-1, C-2] | Task | EP-02 | Critical | 1 | W2 | High | T-1.5.5 | Legacy shim must map from this |
| T-1.2.4 | Drizzle schema — Radio, Geo, Frequency, Schedule [AUDIT M-3] | Task | EP-02 | Critical | 1 | W2 | Medium | T-1.4.x, T-1.5.1 | Radio profiles must map to poc-hermes-app IDs |
| T-1.2.5 | Drizzle schema — Audit & Sync tables [AUDIT L-4] | Task | EP-02 | High | 1 | W3 | Low | T-1.3.x | None |
| T-1.2.6 | Repository layer — UserRepository, SessionRepository | Task | EP-02 | Critical | 1 | W3 | Medium | T-1.3.x | Callsign field |
| T-1.2.7 | Repository layer — RadioRepository, GeoRepository | Task | EP-02 | High | 1 | W3 | Medium | T-1.4.x, T-1.5.1 | Radio status response shape |
| T-1.2.8 | Database seed script | Task | EP-02 | Medium | 1 | W3 | Low | None | Admin callsign |
| T-1.3.1 | RSA keypair + JWT RS256 infrastructure | Task | EP-03 | Critical | 1 | W3 | Medium | T-1.3.2 | Token payload claims |
| T-1.3.2 | Auth endpoints — login/refresh/logout/me [AUDIT H-4] | Task | EP-03 | Critical | 1 | W3 | High | T-1.3.3 | Login response shape |
| T-1.3.3 | JWT verification middleware + RBAC | Task | EP-03 | Critical | 1 | W3 | Medium | T-1.5.x | Bearer token format |
| T-1.4.1 | IRadioDriver interface + HAL types | Task | EP-04 | Critical | 1 | W4 | Low | T-1.4.2, T-1.4.3 | None |
| T-1.4.2 | SimulatedRadioDriver implementation | Task | EP-04 | Critical | 1 | W4 | Medium | T-1.4.5 | None |
| T-1.4.3 | SBitxCLIDriver implementation | Task | EP-04 | Critical | 1 | W4 | High | None | Hz/mode parity with hermes-api |
| T-1.4.4 | RadioCommandQueue + TelemetryPoller [AUDIT H-3] | Task | EP-04 | Critical | 1 | W4 | High | EP-06 | PTT safety |
| T-1.4.5 | IRadioDriver contract test suite | Task | EP-04 | High | 1 | W4 | Medium | None | Output shape parity |
| T-1.5.1 | REST API — Radio endpoints | Task | EP-05 | Critical | 1 | W4 | High | T-1.7.1 | **Critical** wire-compatible |
| T-1.5.2 | REST API — User management endpoints | Task | EP-05 | High | 1 | W5 | Medium | None | callsign in user list |
| T-1.5.3 | REST API — File upload/download + IStorageBackend [AUDIT H-5] | Task | EP-05 | High | 1 | W5 | High | EP-16 | Content-Disposition behavior |
| T-1.5.4 | REST API — System, Geo, Frequency, Schedule, WiFi | Task | EP-05 | Critical | 1 | W5 | Medium | T-1.7.1 | **Critical** wire-compatible |
| T-1.5.5 | REST API — Legacy messaging compatibility shim | Task | EP-05 | Critical | 1 | W5 | Medium | T-1.7.1 | **Critical** backward compat bridge |
| T-1.5.6 | OpenAPI documentation + @fastify/swagger | Task | EP-05 | High | 1 | W6 | Low | None | API contract verification |
| T-1.6.1 | Production Docker Compose + nginx config | Task | EP-01 | High | 1 | W6 | Medium | T-1.7.1 | poc-hermes-app reachability |
| T-1.6.2 | RFC template update — decision authority [AUDIT M-5] | Task | Governance | Medium | 1 | W6 | Low | None | None |
| T-1.7.1 | **GATE: poc-hermes-app Phase 1 Compatibility Checkpoint** | Gate | M-1 | Critical | 1 | W6 | — | Phase 2 start | All REST endpoints |

---

## Tasks — Phase 2

| ID | Title | Type | Epic | Priority | Phase | Week | Complexity | Blocks | IntegrationImpact |
|---|---|---|---|---|---|---|---|---|---|
| T-2.1.1 | IEventBus interface + EventEnvelope + topics.ts [AUDIT C-5] | Task | EP-06 | Critical | 2 | W7 | Medium | T-2.1.2, EP-07 | None (internal) |
| T-2.1.2 | RedisEventBus implementation | Task | EP-06 | Critical | 2 | W7 | High | T-2.2.x, T-2.3.x | None |
| T-2.2.1 | WebSocket gateway — core infrastructure [AUDIT M-4] | Task | EP-07 | Critical | 2 | W7 | High | T-2.2.2 | hermes-v1 subprotocol |
| T-2.2.2 | WebSocket auth + session re-validation [AUDIT C-4] | Task | EP-07 | Critical | 2 | W7 | High | T-2.2.3 | SESSION_INVALIDATED handling |
| T-2.2.3 | WebSocket SUBSCRIBE/UNSUBSCRIBE + rate limiting | Task | EP-07 | High | 2 | W7 | Medium | T-2.3.1 | SUBSCRIBE message format |
| T-2.2.4 | WebSocket frame dispatcher [AUDIT H-2] | Task | EP-07 | High | 2 | W7 | Medium | T-2.4.1 | Error codes |
| T-2.3.1 | Wire TelemetryPoller → IEventBus → WebSocket | Task | EP-08 | High | 2 | W8 | Medium | None | RADIO_TELEMETRY frame format |
| T-2.3.2 | TimescaleDB telemetry ingest worker | Task | EP-08 | High | 2 | W8 | Medium | None | None |
| T-2.4.1 | RADIO_COMMAND WebSocket handler | Task | EP-07 | High | 2 | W8 | Medium | None | RADIO_COMMAND_RESULT format |
| T-2.5.1 | WorkerManager + BullMQ infrastructure + Bull Board | Task | EP-09 | High | 2 | W9 | Medium | T-2.5.2, T-2.5.3 | None |
| T-2.5.2 | CleanupWorker — sessions + sync queue [AUDIT M-2] | Task | EP-09 | Medium | 2 | W9 | Medium | None | None |
| T-2.5.3 | **[AUDIT C-1] MissedDeliveryRecoveryWorker** | Task | EP-14 | **Critical** | 2 | W9 | Medium | None | Prevents message loss |
| T-2.5.4 | ScheduleRunnerWorker | Task | EP-09 | Medium | 2 | W10 | Medium | None | Schedule management UI |
| T-2.6.1 | mediasoup setup + session manager [AUDIT M-6] | Task | EP-10 | High | 2 | W10 | High | T-2.6.2 | WebRTC signaling frames |
| T-2.6.2 | coturn deployment + TURN credential endpoint [AUDIT H-6] | Task | EP-10 | High | 2 | W10 | Medium | None | TURN credentials for client |
| T-2.7.1 | Prometheus metrics — all subsystems | Task | EP-11 | High | 2 | W11 | Medium | None | None |
| T-2.7.2 | OpenTelemetry distributed tracing | Task | EP-11 | Medium | 2 | W11 | Medium | None | None |
| T-2.7.3 | Grafana dashboard provisioning | Task | EP-11 | Medium | 2 | W12 | Medium | None | None |
| T-2.8.1 | **GATE: poc-hermes-app Phase 2 Compatibility Checkpoint** | Gate | M-2 | Critical | 2 | W12 | — | Phase 3 start | WebSocket, telemetry |

---

## Tasks — Phase 3

| ID | Title | Type | Epic | Priority | Phase | Week | Complexity | Blocks | IntegrationImpact |
|---|---|---|---|---|---|---|---|---|---|
| T-3.1.1 | ConversationRepository — full implementation [AUDIT H-1] | Task | EP-12 | Critical | 3 | W13 | High | T-3.1.3 | Unread count, sort order |
| T-3.1.2 | MessageRepository + DeliveryRepository | Task | EP-12 | Critical | 3 | W13 | High | T-3.1.3, T-3.2.x | Message idempotency |
| T-3.1.3 | REST API — Conversations + Messages | Task | EP-12 | Critical | 3 | W13 | High | T-3.2.x | New primary endpoints |
| T-3.2.1 | Online delivery — WebSocket push + fail-fast [AUDIT C-3, H-2] | Task | EP-13 | Critical | 3 | W14 | High | T-3.2.2 | MESSAGE_ACK, MESSAGE_NEW |
| T-3.2.2 | Offline delivery — sync queue entity-reference model [AUDIT C-2] | Task | EP-15 | Critical | 3 | W14 | Very High | T-3.5.1 | SYNC_DELTA, SYNC_COMPLETE |
| T-3.2.3 | Read status — MESSAGE_READ propagation | Task | EP-13 | High | 3 | W14 | Medium | None | Read receipt indicator |
| T-3.3.1 | Message reactions | Task | EP-17 | High | 3 | W15 | Medium | None | REACTION_ADDED frame |
| T-3.3.2 | Reply threading | Task | EP-17 | High | 3 | W15 | Medium | None | replyToMessage in response |
| T-3.3.3 | Typing indicators | Task | EP-17 | Medium | 3 | W15 | Medium | None | TYPING_START/STOP frames |
| T-3.4.1 | Full attachment pipeline + AttachmentProcessingWorker | Task | EP-16 | High | 3 | W15 | High | None | Thumbnail URLs |
| T-3.5.1 | Sync queue population for all entity types [AUDIT C-2] | Task | EP-15 | Critical | 3 | W16 | High | T-3.5.2 | Offline recovery |
| T-3.5.2 | SYNC_REQUEST handling + cursor management [AUDIT C-3] | Task | EP-15 | Critical | 3 | W17 | Very High | T-3.8.1 | Offline recovery |
| T-3.6.1 | HMP ingest service — radio msg to conversation | Task | EP-18 | High | 3 | W17 | High | T-3.6.3 | Inbox compatibility |
| T-3.6.2 | RadioDeliveryWorker — chat to UUCP | Task | EP-18 | High | 3 | W17 | High | None | Outbox compatibility |
| T-3.6.3 | Update legacy messaging shim to use chat model | Task | EP-18 | Critical | 3 | W18 | Medium | T-3.8.1 | **Critical** backward compat |
| T-3.7.1 | Full messaging integration test suite | Task | Testing | Critical | 3 | W18 | High | T-3.8.1 | End-to-end validation |
| T-3.7.2 | Performance validation — all NFR-P targets | Task | Testing | High | 3 | W19 | Medium | T-3.8.1 | Response time baseline |
| T-3.8.1 | **GATE: poc-hermes-app Phase 3 Compatibility Checkpoint** | Gate | M-3 | Critical | 3 | W20 | — | Phase 4 start | Full messaging stack |

---

## Tasks — Phase 4

| ID | Title | Type | Epic | Priority | Phase | Week | Complexity | Blocks | IntegrationImpact |
|---|---|---|---|---|---|---|---|---|---|
| T-Pre4.1 | Draft + decide ADR-008 — Federation Auth [AUDIT L-1] | Task | Governance | Critical | Pre-4 | Before W21 | Medium | T-4.2.1 | Station credential schema |
| T-Pre4.2 | Correct ADR-003 + update IEventBus docs [AUDIT C-5] | Task | Governance | High | Pre-4 | Before W21 | Low | T-4.1.1 | None |
| T-Pre4.3 | Update webrtc.md §2 — reconcile phase assignment [AUDIT L-2] | Task | Governance | Low | Pre-4 | Before W21 | Low | None | None |
| T-4.1.1 | Deploy NATS JetStream + NatsEventBus [AUDIT C-5] | Task | EP-19 | Critical | 4 | W21 | High | T-4.1.2 | Transparent to app |
| T-4.1.2 | NATS smoke test + Redis Pub/Sub decommission | Task | EP-19 | High | 4 | W22 | Medium | T-4.2.x | None |
| T-4.2.1 | Station registry — schema + REST API (per ADR-008) | Task | EP-20 | Critical | 4 | W22 | High | T-4.2.2 | Station list UI |
| T-4.2.2 | Cross-station delivery — DeliveryRouter + FederationDeliveryWorker | Task | EP-20 | Critical | 4 | W22 | Very High | T-4.2.3 | Cross-station messages |
| T-4.2.3 | Station heartbeat publisher | Task | EP-20 | High | 4 | W23 | Low | None | Online status |
| T-4.3.1 | RadioAudioBridge — ALSA capture to mediasoup | Task | EP-21 | High | 4 | W23 | Very High | T-4.4.1 | Radio audio UI |
| T-4.4.1 | Group WebRTC — multi-participant sessions | Task | EP-22 | High | 4 | W25 | High | None | Group call UI |
| T-4.4.2 | E2EE consideration — ADR-009 [AUDIT L-2] | Task | Governance | Medium | 4 | W26 | Low | None | Documented tradeoff |
| T-4.5.1 | Federation message authentication (per ADR-008) | Task | EP-23 | Critical | 4 | W26 | High | None | Inter-station security |
| T-4.5.2 | MinIO storage backend [AUDIT H-5] | Task | EP-23 | High | 4 | W27 | Medium | None | Attachment URL format |
| T-4.5.3 | Remove legacy messaging shim [TD-01] | Task | EP-23 | High | 4 | W27 | Low | T-4.7.1 | **Breaking** — coordinate with app |
| T-4.6.1 | Multi-station integration test suite | Task | Testing | Critical | 4 | W28 | High | T-4.7.1 | End-to-end federation |
| T-4.7.1 | **GATE: poc-hermes-app Phase 4 Compatibility Checkpoint** | Gate | M-4 | Critical | 4 | W28 | — | v2.0.0 release | Full system |

---

## Labels / Tags for Issue Tracking

```
phase:1, phase:2, phase:3, phase:4
priority:critical, priority:high, priority:medium, priority:low
type:epic, type:task, type:gate, type:governance
audit:c-1, audit:c-2, audit:c-3, audit:c-4, audit:c-5
audit:h-1, audit:h-2, audit:h-3, audit:h-4, audit:h-5, audit:h-6
audit:m-2, audit:m-3, audit:m-4, audit:m-5, audit:m-6, audit:m-7
compat:poc-hermes-app
component:hal, component:auth, component:websocket, component:eventbus
component:messaging, component:sync, component:webrtc, component:federation
component:observability, component:database, component:jobs
security, performance, breaking-change
```

---

## Dependency Graph (Textual for Import)

```
T-1.1.1 → T-1.1.4 → T-1.5.x
T-1.1.3 → T-1.2.x
T-1.2.1 → T-1.2.2 → T-1.3.x
T-1.2.1 → T-1.2.3 → T-1.5.5
T-1.2.1 → T-1.2.4 → T-1.4.x, T-1.5.1
T-1.2.1 → T-1.2.5 → T-1.3.x
T-1.2.2 → T-1.2.6 → T-1.3.x
T-1.2.4 → T-1.2.7 → T-1.4.x
T-1.3.1 → T-1.3.2 → T-1.3.3 → T-1.5.x → T-1.7.1 [M-1 Gate]

T-1.7.1 → T-2.1.1 → T-2.1.2 → T-2.2.x, T-2.3.x
T-2.1.1 → T-2.2.1 → T-2.2.2 → T-2.2.3 → T-2.3.1
T-2.2.1 → T-2.2.4 → T-2.4.1
T-2.1.2 → T-2.5.1 → T-2.5.2, T-2.5.3, T-2.5.4
T-2.6.1 → T-2.6.2
T-2.8.1 [M-2 Gate] ← T-2.2.x, T-2.3.x, T-2.5.x, T-2.6.x, T-2.7.x

T-2.8.1 → T-3.1.1 → T-3.1.3 → T-3.2.1, T-3.2.2
T-3.1.2 → T-3.1.3 → T-3.2.x
T-3.2.1 → T-3.2.2 → T-3.5.1 → T-3.5.2
T-3.6.1 → T-3.6.3
T-3.8.1 [M-3 Gate] ← T-3.1.x, T-3.2.x, T-3.3.x, T-3.4.x, T-3.5.x, T-3.6.x, T-3.7.x

T-3.8.1 → T-Pre4.1, T-Pre4.2 → T-4.1.1 → T-4.1.2 → T-4.2.x
T-4.2.1 → T-4.2.2 → T-4.2.3
T-4.3.1 → T-4.4.1
T-4.7.1 [M-4 Gate] ← T-4.1.x, T-4.2.x, T-4.3.x, T-4.4.x, T-4.5.x, T-4.6.x
```
