# Phase 4 — Federation & Distribution

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 21–28  
**Prerequisites**: Phase 3 complete and stable in production  
**Goal**: Multi-station synchronization, NATS JetStream migration, advanced WebRTC, and federation readiness

---

## Objectives

By the end of Phase 4:

1. Multiple Hermes stations can exchange messages via internet or radio bridge
2. Internal event bus migrated from Redis Pub/Sub to NATS JetStream
3. Radio-to-WebRTC bridge: HF radio audio streamed to WebRTC clients
4. Group WebRTC audio calls (multiple participants)
5. Station discovery and presence (which stations are reachable)
6. End-to-end encryption consideration (architecture decision, not necessarily v1 implementation)

---

## Week 21–22: NATS JetStream Migration

### Why Now

Phase 1-3 used Redis Pub/Sub as an event bus (sufficient for single-station). Multi-station federation requires:
- Durable subscriptions (events not lost when a station temporarily disconnects)
- Consumer groups (load-balanced processing across station nodes)
- Subject-based routing (station callsign as subject prefix)
- Hub-spoke leaf node topology (each station as NATS leaf node)

### Migration Steps

- [ ] Deploy NATS JetStream server (Docker Compose addition)
- [ ] Implement `NatsEventBus` implementing `IEventBus` interface
- [ ] Configure all existing topics as JetStream streams with retention policy
- [ ] Switch via config: `EVENT_BUS_DRIVER=nats` (zero application code change)
- [ ] Smoke test: all existing event flows work correctly with NATS
- [ ] Decommission Redis Pub/Sub usage (Redis retained for BullMQ, sessions, rate limiting)

### NATS Stream Configuration

```
Stream: hermes.radio        subjects: radio.>    retention: 24h
Stream: hermes.messaging    subjects: message.>  retention: 7d
Stream: hermes.sync         subjects: sync.>     retention: 7d
Stream: hermes.presence     subjects: presence.> retention: 1h
Stream: hermes.federation   subjects: station.>  retention: 30d
```

### Station Identity

Each station has a unique callsign (e.g., `PY2XA`, `PY8EME`) used as:
- NATS subject prefix for inter-station messages: `station.PY2XA.message.new`
- Federation routing key
- HMP/UUCP addressing

---

## Week 22–23: Multi-Station Synchronization

### Station Registry

- [ ] `stations` table: callsign, last_seen, ip_address, connectivity_type
- [ ] `POST /api/v1/federation/stations` — register remote station
- [ ] `GET /api/v1/federation/stations` — list known stations and online status
- [ ] Station presence via NATS heartbeat (stations publish `station.<callsign>.heartbeat` every 30s)

### Cross-Station Message Delivery

When user A (Station PY2XA) sends a message to user B (Station PY8EME):

- [ ] `DeliveryRouter`: determines if recipient is on local station or remote
- [ ] For remote recipients: `FederationDeliveryWorker`
  - Publish `station.PY8EME.message.new` to NATS JetStream
  - Station PY8EME subscribes to `station.PY8EME.>` on its NATS leaf node
  - Receiving station creates message in local conversation
  - Sends delivery acknowledgment back: `station.PY2XA.delivery.PY8EME.ack`

### Radio-Based Federation (No Internet)

For stations without internet (UUCP-only):

- [ ] Preserve existing HMP/UUCP delivery path from Phase 3
- [ ] HMP ingest service imports messages from remote stations
- [ ] Station marks as `connectivity_type: 'radio'` in station registry
- [ ] Message router falls back to `RadioDeliveryWorker` for radio-only stations

---

## Week 23–25: Radio-to-WebRTC Bridge

### Architecture

```
HF Radio (sBitx) → ALSA audio capture → ffmpeg (Opus encode)
  → RTP stream → mediasoup PlainTransport → mediasoup Router
  → WebRTC consumers (browser clients)

Browser mic → mediasoup WebRtcTransport → mediasoup Router
  → mediasoup PlainTransport → ffmpeg (Opus decode) → ALSA playback
  → Radio TX (with HAL PTT control)
```

### Implementation

- [ ] `RadioAudioBridge` service:
  - Starts ffmpeg process to capture ALSA audio and encode to Opus RTP
  - Creates mediasoup PlainTransport to receive RTP
  - Creates mediasoup Producer from PlainTransport
  - Router distributes audio to all WebRTC consumers (browser clients)
  - On WebRTC client transmit: reverse path + HAL.setPTT(true)

- [ ] HAL integration: PTT activated only when WebRTC client is transmitting
  - Prevent simultaneous TX from multiple clients (turn-taking protocol)
  - TX queue: one client transmits at a time, others queued

- [ ] Radio bridge session linked to a `conversation` of type `radio`
  - All text messages in the conversation are also logged
  - Audio session events (join, leave, TX) logged in conversation

### Security

- [ ] Only `operator` and `admin` roles can activate PTT via WebRTC bridge
- [ ] TX duration limit (configurable, default 60 seconds)
- [ ] HAL command queue enforces serial PTT activation

---

## Week 25–26: Advanced WebRTC (Group Audio)

### Group Calls

- [ ] Multiple participants in a WebRTC session (linked to a group conversation)
- [ ] Active speaker detection (mediasoup `activeSpeaker` event)
- [ ] Participant roster: `WEBRTC_PARTICIPANT_JOINED` / `WEBRTC_PARTICIPANT_LEFT` events
- [ ] Audio mixing configuration: all-to-all (everyone hears everyone)

### WebRTC Improvements

- [ ] End-to-end encryption consideration:
  - Evaluate Insertable Streams (Chrome/Edge WebRTC E2EE API)
  - Document tradeoffs: radio bridge breaks with E2EE (audio must be decoded server-side)
  - Decision: E2EE optional, not default — radio bridge takes priority
- [ ] TURN relay performance test at target hardware specs

---

## Week 26–27: Security & Hardening for Federation

### Federation Security

- [ ] Station-to-station authentication via shared secrets or PKI
- [ ] NATS TLS for all connections (leaf node → hub)
- [ ] Message signing (sender station signs with its private key)
- [ ] Receiver verifies signature before accepting federation messages
- [ ] Replay attack protection: message ID uniqueness checked before accepting

### Audit Improvements

- [ ] All federation events logged in `audit_logs`
- [ ] Rate limiting on federation delivery (prevent DoS from malicious station)
- [ ] Station blocklist (ban station by callsign)

---

## Week 27–28: Documentation & Long-Term Readiness

### API Documentation

- [ ] OpenAPI spec fully complete and accurate
- [ ] WebSocket protocol reference client (TypeScript, for hermes-chat import)
- [ ] Federation protocol documentation (how to add a new station)
- [ ] Radio bridge setup guide

### Operational Documentation

- [ ] Station installation guide (from bare OS to running station)
- [ ] Backup and restore procedure
- [ ] Disaster recovery runbook
- [ ] Certificate rotation procedure
- [ ] Redis, PostgreSQL, NATS upgrade guides

### CRDT Evaluation (Architecture Decision)

- [ ] Evaluate CRDT libraries (Yjs, Automerge) for Phase 5 consideration
- [ ] Document which entities would benefit: reactions (OR-Set), read positions (Max-Register)
- [ ] Record decision in ADR-008-conflict-resolution.md

---

## Definition of Done — Phase 4

- [ ] Two Hermes stations exchange messages in real-time via NATS federation
- [ ] HF radio audio bridged to WebRTC group call
- [ ] NATS JetStream replacing Redis Pub/Sub with all tests passing
- [ ] Station registry showing online/offline stations
- [ ] Federation message signing preventing spoofing
- [ ] All documentation updated for multi-station deployment
- [ ] CI/CD deploys to both stations in parallel

---

## Future Considerations (Phase 5+)

- **Matrix protocol compatibility**: Federate with Matrix homeservers for interoperability
- **Mesh networking**: Station-to-station without central NATS hub (NATS leaf cluster)
- **E2EE for non-radio conversations**: Insertable Streams + key exchange
- **SDR support**: Add `SdrRadioDriver` to HAL for software-defined radio hardware
- **Mobile native apps**: React Native with offline-first SQLite sync
- **Satellite connectivity**: Iridium/Inmarsat fallback transport
