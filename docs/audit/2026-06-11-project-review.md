# Project Review — poc-hermes-server

**Reviewer**: Backend Architect  
**Date**: 2026-06-11  
**Scope**: Full project — architecture, technology stack, roadmap, documentation completeness, and operational feasibility  
**Input documents**: All architecture docs, 7 ADRs, master plan, 4-phase roadmap, 4-phase task lists, prior adversarial audit (2026-05-20)  
**Verdict**: Architecture is sound and ready for Phase 1 implementation, with specific changes required in project naming, roadmap sequencing, and missing documentation before the first commit.

---

## Overall Assessment

The architectural foundations are correct and do not require redesign. The key decisions — modular monolith, `IRadioDriver` HAL, chat-native messaging, BullMQ delivery queues, `IEventBus` abstraction, RS256 JWT, Drizzle ORM, and TimescaleDB for telemetry — are well-reasoned, well-documented in ADRs, and appropriate for the scale and target hardware.

The project is **ready to begin Phase 1 implementation** with the blocking actions listed in §6 addressed first.

---

## 1. What Is Sound and Should Not Be Revisited

| Decision | Why It Is Correct |
|---|---|
| Modular monolith over premature microservices | Correct for a single-station deployment and small team; boundaries are clean enough to extract later |
| `IRadioDriver` with `execFile` array args | Textbook secure CLI wrapping; prevents command injection by design |
| Chat-native messaging over email inbox/outbox | The email model cannot be extended to support reactions, threading, delivery tracking, or multi-device sync |
| `IEventBus` abstraction with `subscribe()` / `subscribeExclusive()` | Correct; the Redis→NATS migration semantic mismatch is fixed in docs without abandoning the abstraction |
| mediasoup SFU over Janus / Jitsi | Correct given the radio bridge requirement; PlainTransport is the only clean path for ALSA→WebRTC |
| Drizzle ORM over Prisma | No codegen, no Rust query engine; correct for constrained hardware |
| BullMQ for delivery queues | Redis-backed, persistent, retry-semantics; correct choice for at-least-once delivery |
| RS256 JWT over HS256 | Asymmetric signing enables future public-key verification by edge/leaf nodes |
| TimescaleDB on the same PostgreSQL | No additional process; correct at this scale; migration path to dedicated TSDB is straightforward |
| ADR quality (all 7) | All ADRs are well-reasoned and document rejected alternatives; no ADR should be reopened |

---

## 2. Concerns and Recommended Changes

### 2.1 Stack Weight vs. Raspberry Pi 4 Hardware — HIGH

**Problem**: The full stack — PostgreSQL 17 + TimescaleDB + Redis 7 + Node.js event loop + BullMQ workers + mediasoup workers + coturn — is expected to run on a Raspberry Pi 4 with 4 GB RAM in field conditions (unreliable power, temperature stress, no remote ops).

The adversarial audit (M-6) correctly flagged mediasoup CPU contention and fixed the worker count formula. However, the total memory footprint of all services running concurrently under load has never been estimated or validated.

**Risk**: Memory exhaustion during a peak messaging + telemetry ingest + WebRTC session scenario will cause OOM kills of whichever service the kernel decides to evict. PostgreSQL being killed mid-transaction is the worst outcome for data integrity.

**Required action — before Phase 1 ships (M-1)**:
- Start the full `docker compose` stack locally on equivalent hardware (or emulated ARM environment)
- Run `docker stats` under idle and simulated load
- Establish a memory budget table and document it in the deployment guide:

| Service | Idle RSS Target | Peak RSS Limit |
|---|---|---|
| PostgreSQL + TimescaleDB | 256 MB | 512 MB |
| Redis 7 | 64 MB | 128 MB |
| Node.js app (all workers) | 256 MB | 512 MB |
| mediasoup (2 workers) | 128 MB | 256 MB |
| coturn | 32 MB | 64 MB |
| OS + overhead | 256 MB | 384 MB |
| **Total target** | **~1 GB** | **<3 GB** |

If the total peak exceeds 3 GB, the stack needs to be trimmed or the minimum hardware spec upgraded.

---

### 2.2 WebRTC Is Prioritized Over HMP/UUCP — HIGH

**Problem**: Phase 2 (Weeks 7–12) includes mediasoup setup and WebRTC signaling infrastructure. HMP/UUCP interoperability (EP-18) is Phase 3, Weeks 17–18.

This means the system will have a polished real-time WebSocket gateway, TypeScript APIs, and WebRTC LAN audio by Week 12 — but cannot actually send a message over HF radio until Week 17. The core value proposition of HERMES (HF radio messaging for disconnected communities) ships five weeks after the secondary feature (LAN voice calls).

**Recommended resequencing**:

| Phase | Current | Recommended |
|---|---|---|
| Phase 2 | WebSocket + EventBus + WebRTC (Weeks 7–12) | WebSocket + EventBus + HMP/UUCP ingest + outbound radio delivery (Weeks 7–12) |
| Phase 3 | Messaging + Sync + Attachments (Weeks 13–20) | Messaging + Sync + Attachments + WebRTC backend (Weeks 13–20) |

WebRTC LAN audio is a valuable feature, but it is not the reason HERMES exists. HMP/UUCP radio transport is.

---

### 2.3 Phase 4 Scope Is Not Achievable in 8 Weeks — HIGH

**Problem**: Phase 4 (Weeks 21–28) contains:
1. NATS JetStream migration of the entire event bus
2. Multi-station message delivery via federation
3. Radio-to-WebRTC audio bridge (ffmpeg → ALSA → mediasoup PlainTransport)
4. Group WebRTC audio calls (multiple participants)
5. Station discovery and presence via NATS heartbeats
6. Security hardening for federation (station auth model)
7. End-to-end encryption architecture decision

Each of items 1–6 is a significant engineering effort independently. Item 3 (radio bridge) is the most complex integration in the entire project — it touches HAL, mediasoup, ALSA, PTT coordination, and has no existing reference implementation in this codebase.

**Recommended split**:

**Phase 4a** (Weeks 21–28 — 8 weeks):
- NATS JetStream migration
- Internet-based station-to-station message delivery (federation)
- Station discovery and presence
- Federation security hardening and ADR-008

**Phase 4b** (Weeks 29–36 — 8 weeks):
- Radio-to-WebRTC audio bridge
- Group WebRTC calls
- End-to-end encryption decision and implementation

This keeps each phase deliverable as a shippable milestone rather than a half-finished collection of features.

---

### 2.4 Project Name Contains "poc-" But Is Not a Proof of Concept — MEDIUM

**Problem**: The project is named `poc-hermes-server`. This name propagates into:
- Docker image tags
- Prometheus `service` labels
- Pino log `base` fields
- CI pipeline names and artifact paths
- Grafana dashboard titles and alert rule labels
- Every document header

The 28-week, 4-phase, 23-epic roadmap with production-grade architecture decisions, ADRs, and phase gates is not a proof of concept. It is a production system. Renaming after any artifact is created requires updating CI pipelines, dashboards, alert rules, and log query filters simultaneously.

**Action required — before the first commit**:  
Rename to `hermes-server`. The cost today is one `find/replace` across docs and a repository rename. The cost after Phase 1 ships is a multi-day migration affecting every operational artifact.

**Audit finding reference**: L-5 (2026-05-20 audit)

---

### 2.5 Legacy `hermes-api` Endpoint Inventory Is Missing — MEDIUM

**Problem**: EP-05 ("Replace all `hermes-api` PHP endpoints") is a Phase 1 Critical epic. The `poc-hermes-app` compatibility gate at M-1 requires all legacy endpoints to be replaced. However, there is no document in this repository that lists the actual endpoints currently exposed by `hermes-api`.

If a legacy endpoint is discovered during Phase 1 implementation that was not accounted for in the schema or route plans, the M-1 compatibility gate will be blocked and the team will not own the fix timeline.

**Required action — before Phase 1 begins**:
- Create `docs/compatibility/hermes-api-endpoint-inventory.md`
- List every route in the legacy PHP system: method, path, request shape, response shape, authentication requirement
- Cross-reference each route to its planned replacement in the Phase 1 task list
- Mark any routes with no planned replacement as explicitly deprecated with a removal timeline

---

### 2.6 `poc-hermes-app` Is a Dependency the Team Does Not Control — MEDIUM

**Problem**: Every phase milestone requires a "compatibility verification with `poc-hermes-app`" gate as a hard blocker before progression. `poc-hermes-app` is declared out of scope and lives on its own development timeline.

Scenarios that block a phase gate without a clear owner:
- `poc-hermes-app` WebSocket client does not negotiate the `hermes-v1` subprotocol
- `poc-hermes-app` does not handle `SESSION_INVALIDATED` and has no reconnect flow
- `poc-hermes-app` does not send `MESSAGE_ACK` frames
- `poc-hermes-app` is not updated on the same release cadence

**Required action — before Phase 1 begins**:
- Document explicitly who owns `poc-hermes-app` integration work and on what timeline
- Define what "compatibility verified" means at each gate: is it a passing automated test, a manual checklist, or a sign-off from the app team?
- Add a fallback: if `poc-hermes-app` is not updated in time, define which gate criteria can be verified with a test client instead

---

## 3. Missing Documentation

| Document | Impact | Urgency |
|---|---|---|
| `docs/compatibility/hermes-api-endpoint-inventory.md` | Phase 1 compatibility gate risk | Before Phase 1 |
| Memory/resource budget for RPi 4 full stack | Operational reliability in field | Before M-1 |
| ADR-008: Federation authentication model | Blocks Phase 4 schema design | Before Phase 4 |
| Deployment guide / field operations runbook | Phase 3 operability | Before M-3 |
| `poc-hermes-app` compatibility ownership and test strategy | Phase gate risk | Before Phase 1 |

---

## 4. Open Audit Findings Status

All findings from the 2026-05-20 adversarial audit were addressed in documentation on 2026-06-10. The following table tracks current status:

| ID | Finding | Severity | Doc Fix Status | Implementation Phase |
|---|---|---|---|---|
| C-1 | Message delivery crash window | CRITICAL | ✅ Fixed in docs | Phase 2 |
| C-2 | sync_queue stale payload | CRITICAL | ✅ Fixed in docs | Phase 3 |
| C-3 | Multi-device cursor broken for online clients | CRITICAL | ✅ Fixed in docs | Phase 3 |
| C-4 | JWT expiry during WebSocket session | CRITICAL | ✅ Fixed in docs | Phase 2 |
| C-5 | NATS migration "zero code change" claim | CRITICAL | ✅ Fixed in docs | Before Phase 4 |
| H-1 | `conversations.last_message_id` hot-row | HIGH | ✅ Fixed in docs | Phase 1 |
| H-2 | 30s ACK timeout TOCTOU | HIGH | ✅ Fixed in docs | Phase 2 |
| H-3 | Redis SPOF for PTT | HIGH | Documented in plan | Phase 1 |
| H-4 | Refresh token rotation optional | HIGH | ✅ Fixed in docs | Phase 1 |
| H-5 | No `IStorageBackend` abstraction | HIGH | Documented in plan | Phase 1 |
| H-6 | TURN credential RBAC gap | HIGH | ✅ Fixed in docs | Phase 2 |
| M-1 | sync_queue payload duplication | MEDIUM | ✅ Resolved by C-2 fix | Phase 3 |
| M-2 | No cleanup for never-reconnecting devices | MEDIUM | Open | Phase 3 |
| M-3 | TimescaleDB background job contention | MEDIUM | Documented in plan | Phase 1 |
| M-4 | WsEnvelope has no version field | MEDIUM | ✅ Fixed in docs | Phase 2 |
| M-5 | RFC process has no decision authority | MEDIUM | ✅ Fixed in docs | Complete |
| M-6 | mediasoup shares all CPU cores | MEDIUM | ✅ Fixed in docs | Phase 2 |
| M-7 | Pino redact misses nested fields | MEDIUM | Documented in plan | Phase 1 |
| L-1 | Federation auth model deferred | LOW | Open — ADR-008 needed | Before Phase 4 |
| L-2 | WebRTC phase assignment contradiction | LOW | Partially resolved | Resequencing recommended |
| L-3 | `localStorage` referenced for mobile arch | LOW | Open | Phase 3 |
| L-4 | Audit action strings not compile-time safe | LOW | Documented in plan | Phase 1 |
| L-5 | "poc-" prefix in all production artifacts | LOW | Open — rename required | Before first commit |

---

## 5. Recommended Roadmap Adjustments

### Current sequencing (problematic)
```
Phase 1 [W1–6]   Foundation (REST, DB, HAL, Auth)
Phase 2 [W7–12]  Real-Time (WebSocket, EventBus, WebRTC, BullMQ)
Phase 3 [W13–20] Messaging (Chat, Sync, Attachments, HMP/UUCP at W17-18)
Phase 4 [W21–28] Federation (NATS + radio bridge + group calls — all 8 weeks)
```

### Recommended sequencing
```
Phase 1 [W1–6]   Foundation (REST, DB, HAL, Auth)  ← no change
Phase 2 [W7–12]  Real-Time (WebSocket, EventBus, BullMQ, HMP/UUCP ingest + outbound)
Phase 3 [W13–20] Messaging (Chat, Sync, Attachments, WebRTC backend)
Phase 4a [W21–28] Federation (NATS migration, station-to-station, station discovery, ADR-008)
Phase 4b [W29–36] Advanced (radio bridge, group WebRTC, E2E encryption decision)
```

**Rationale**: Radio transport is the system's primary purpose. It should ship in Phase 2 alongside the WebSocket infrastructure that will deliver inbound HMP messages to connected clients. WebRTC is a secondary feature that belongs after the core messaging pipeline is proven.

---

## 6. Blocking Actions Before First Commit

These must be completed before any implementation code is written:

| # | Action | Owner | Urgency |
|---|---|---|---|
| 1 | Rename repository from `poc-hermes-server` to `hermes-server` | Engineering Lead | Now |
| 2 | Create `docs/compatibility/hermes-api-endpoint-inventory.md` with all legacy routes | Backend Engineer | Before Phase 1 starts |
| 3 | Document `poc-hermes-app` compatibility ownership and test strategy | Engineering Lead | Before Phase 1 starts |
| 4 | Validate memory budget for full stack on RPi 4 (or equivalent ARM hardware) | DevOps / Backend | Before M-1 |
| 5 | Open ADR-008 for federation authentication model | Engineering Lead | Before Phase 4 design begins |

---

## 7. Verdict Summary

| Area | Status | Action |
|---|---|---|
| Architecture fundamentals | ✅ Sound | No changes needed |
| Technology stack choices | ✅ Correct | No changes needed |
| ADRs (all 7) | ✅ Complete | No changes needed |
| Phase 1 task list | ✅ Ready to implement | Address blocking actions first |
| Phase 2 sequencing | ⚠️ Suboptimal | Move HMP/UUCP before WebRTC |
| Phase 3 task list | ✅ Well-specified | Minor WebRTC resequencing |
| Phase 4 scope | ⚠️ Unrealistic in 8 weeks | Split into Phase 4a and Phase 4b |
| Stack for RPi 4 | ⚠️ Unvalidated under full load | Memory budget required before M-1 |
| Project name | ❌ Must change | Rename before first commit |
| Legacy endpoint inventory | ❌ Missing | Required before Phase 1 |
| `poc-hermes-app` dependency ownership | ❌ Undefined | Required before Phase 1 |
