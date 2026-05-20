# Phase 2 — Real-Time Infrastructure

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 7–12  
**Prerequisites**: Phase 1 complete and deployed  
**Goal**: WebSocket gateway, event-driven radio telemetry, WebRTC audio, and background job infrastructure

---

## Objectives

By the end of Phase 2:

1. WebSocket gateway running with typed hermes-v1 protocol
2. Radio telemetry streaming in real-time to subscribed clients
3. Internal Redis Pub/Sub event bus wiring all modules
4. BullMQ workers for delivery, attachment processing, cleanup
5. Basic WebRTC audio sessions (same station LAN) via mediasoup
6. coturn STUN/TURN deployed alongside the server

Phase 2 does **not** include the full chat-native messaging model or sync engine — those are Phase 3.

---

## Week 7: Event Bus + WebSocket Gateway (Core)

### Event Bus

- [ ] Implement `IEventBus` interface
- [ ] Implement `RedisEventBus` (Redis Pub/Sub, pattern subscriptions)
- [ ] Wire EventBus into all modules via dependency injection
- [ ] Topic taxonomy defined and typed (`src/events/topics.ts`)
- [ ] `EventEnvelope<T>` type with id, topic, payload, timestamp, version

### WebSocket Gateway

- [ ] `wss://host/ws` endpoint accepting `Sec-WebSocket-Protocol: hermes-v1`
- [ ] 10-second authentication timeout (close with code 4001 if no AUTHENTICATE received)
- [ ] JWT verification on AUTHENTICATE message
- [ ] Connection registry: track connected users, devices, subscriptions
- [ ] PING/PONG heartbeat: server PING every 30s, close if no PONG within 10s
- [ ] `SUBSCRIBE` / `UNSUBSCRIBE` message handling
- [ ] Rate limiting per connection (60 messages/min, 10 RADIO_COMMAND/min)

### Tests

- [ ] WebSocket connection lifecycle (auth, timeout, heartbeat, disconnect)
- [ ] SUBSCRIBE/UNSUBSCRIBE mechanics
- [ ] Rate limit enforcement

---

## Week 8: Radio Telemetry Streaming

### TelemetryPoller Integration

- [ ] `TelemetryPoller` polling HAL every 1000ms (configurable)
- [ ] Change detection (only publish `radio.state_changed` on significant delta)
- [ ] Publish `radio.telemetry` every poll (regardless of change) for dashboard
- [ ] WebSocket gateway subscribes to `radio.telemetry` and `radio.state_changed`
- [ ] Push `RADIO_TELEMETRY` frame to all connections subscribed to `radio.telemetry`

### TimescaleDB Telemetry Ingest

- [ ] `TelemetryIngestWorker` (BullMQ) batch-inserts telemetry into `radio_telemetry`
- [ ] Batch size: 60 rows per insert (1 minute of data)
- [ ] Insert only on change (no duplicates for static state)
- [ ] Retention policy: 90 days hot, compress after 7 days

### Tests

- [ ] Telemetry event emitted by simulated driver at correct interval
- [ ] WebSocket client receives RADIO_TELEMETRY after subscribing
- [ ] Client does not receive telemetry if not subscribed to topic

---

## Week 8–9: HAL Command via WebSocket

- [ ] `RADIO_COMMAND` WebSocket message type
- [ ] Validated against same schemas as REST radio endpoints
- [ ] Routed to `RadioCommandQueue` (same queue as REST — shared)
- [ ] Response: `RADIO_COMMAND_RESULT` with requestId correlation
- [ ] Permission check: RADIO_COMMAND requires role `operator` or `admin`

---

## Week 9–10: BullMQ Worker Infrastructure

### Workers to Implement

| Worker | Queue | Purpose |
|---|---|---|
| `TelemetryIngestWorker` | `telemetry:ingest` | Batch TimescaleDB inserts |
| `AttachmentProcessingWorker` | `attachment:processing` | Thumbnail/preview generation |
| `CleanupWorker` | `cleanup` | Expired sessions, old sync queue rows |
| `ScheduleRunnerWorker` | `schedule:runner` | Execute connection schedules |

### Infrastructure

- [ ] `WorkerManager` class: start/stop all workers, graceful shutdown
- [ ] Bull Board UI at `/admin/queues` (admin-only, basic auth)
- [ ] Worker health exposed in `/health/ready` check
- [ ] Failed job alerting via `bullmq_jobs_failed_total` Prometheus metric

---

## Week 10–11: WebRTC (Basic — LAN Audio)

### Server Side

- [ ] Install and configure mediasoup v3
- [ ] Worker pool: `min(os.cpus().length, 4)` workers
- [ ] Router per session with Opus codec config (FEC, DTX enabled)
- [ ] `WebRtcSessionManager`: create/destroy sessions, manage transports
- [ ] WebRTC signaling via existing WebSocket gateway (`webrtc.*` topics)
- [ ] Message types: `WEBRTC_CREATE_SESSION`, `WEBRTC_TRANSPORT_CREATED`, `WEBRTC_PRODUCE`, `WEBRTC_CONSUME`, `WEBRTC_END_SESSION`

### Client Side (hermes-chat)

- [ ] WebRTC client module using browser WebRTC API
- [ ] Signal via existing WebSocket connection
- [ ] Basic audio call UI (call, hangup)

### coturn Deployment

- [ ] coturn Docker container in `docker-compose.yml`
- [ ] `denied-peer-ip` for all RFC 1918 ranges
- [ ] HMAC secret from environment variable
- [ ] TURN credential endpoint: `GET /api/v1/webrtc/credentials` (returns time-limited TURN creds)

### Tests

- [ ] mediasoup worker starts without error
- [ ] Session creation/destruction lifecycle
- [ ] TURN credential generation (HMAC-SHA1, correct TTL)

---

## Week 11–12: Observability + Polish

### Prometheus Metrics

- [ ] All key metrics implemented (HTTP, WebSocket, HAL, queues — see observability.md)
- [ ] `/metrics` endpoint (internal, not proxied to public)
- [ ] Grafana dashboard provisioned via `infrastructure/observability/dashboards/`

### Docker Compose — Production-like

- [ ] `docker-compose.prod.yml` with resource limits, restart policies
- [ ] Observability stack: `docker-compose.observability.yml` (Prometheus, Grafana, Loki, Tempo)
- [ ] NGINX reverse proxy config with TLS termination

### Logging

- [ ] All request logs include `requestId`, `userId`, `durationMs`
- [ ] WebSocket lifecycle events logged at INFO
- [ ] HAL errors logged at ERROR with full context
- [ ] `LOG_LEVEL` configurable via environment variable

---

## Definition of Done — Phase 2

- [ ] WebSocket clients receive real-time radio telemetry within 1100ms of hardware state change
- [ ] WebRTC audio calls working between two browser clients on the same LAN
- [ ] All BullMQ workers running without errors in production Docker Compose
- [ ] Grafana dashboard showing real-time metrics
- [ ] `/health/ready` includes WebSocket gateway and worker health
- [ ] Phase 1 endpoints still passing all tests (no regression)
- [ ] Prometheus alerting rules configured for radio hardware unavailability
