# HERMES Server — Master Software Development Plan

**Project**: poc-hermes-server  
**Document**: Master Project Plan (MPP-001)  
**Version**: 1.0  
**Date**: 2026-06-10  
**Status**: Active  

> **Mandatory cross-cutting constraint**: Continuous compatibility with **poc-hermes-app** (the existing frontend and radio-control application) is required throughout the entire project lifecycle. Every phase concludes with a dedicated "Compatibility Verification with poc-hermes-app" gate before progression is allowed.

---

## Table of Contents

1. [Project Objectives and Scope](#1-project-objectives-and-scope)
2. [Functional Requirements](#2-functional-requirements)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [System Architecture and Technical Design](#4-system-architecture-and-technical-design)
5. [User Stories and Use Cases](#5-user-stories-and-use-cases)
6. [Development Phases and Milestones](#6-development-phases-and-milestones)
7. [Detailed Task Breakdown — Epics and Features](#7-detailed-task-breakdown--epics-and-features)
8. [Dependency Map](#8-dependency-map)
9. [Testing Strategy](#9-testing-strategy)
10. [Risk Assessment and Mitigation](#10-risk-assessment-and-mitigation)
11. [Deployment and Release Strategy](#11-deployment-and-release-strategy)
12. [Documentation Requirements](#12-documentation-requirements)
13. [Maintenance and Future Enhancements](#13-maintenance-and-future-enhancements)
14. [poc-hermes-app Compatibility Matrix](#14-poc-hermes-app-compatibility-matrix)

---

## 1. Project Objectives and Scope

### 1.1 Mission Statement

HERMES (High-frequency Emergency and Rural Multimedia Exchange System) is a next-generation backend for HF radio communication stations that serve remote, rural, and disaster-affected communities. `poc-hermes-server` replaces the existing PHP-based `hermes-api` with a production-grade TypeScript server built for years of evolution, operational resilience, and extensibility.

### 1.2 Primary Objectives

| # | Objective | Target Phase |
|---|---|---|
| O-1 | Replace all `hermes-api` PHP endpoints with TypeScript/Fastify equivalents under `/api/v1/` | Phase 1 |
| O-2 | Migrate from SQLite to PostgreSQL 17 + TimescaleDB + Redis 7 | Phase 1 |
| O-3 | Implement a Hardware Abstraction Layer (HAL) decoupling business logic from sBitx CLI | Phase 1 |
| O-4 | Deploy a real-time WebSocket gateway with the `hermes-v1` typed protocol | Phase 2 |
| O-5 | Implement an event-driven architecture via Redis Pub/Sub (`IEventBus`) | Phase 2 |
| O-6 | Build a chat-native messaging model with delivery tracking and offline sync | Phase 3 |
| O-7 | Enable multi-station federation via NATS JetStream | Phase 4 |
| O-8 | Deliver a radio-to-WebRTC audio bridge for browser-based radio monitoring | Phase 4 |

### 1.3 Scope Boundary

**In scope:**
- All REST API endpoints replacing `hermes-api`
- JWT RS256 authentication and RBAC
- Hardware Abstraction Layer (sBitx, simulated driver)
- WebSocket gateway (`hermes-v1` protocol)
- Redis event bus and BullMQ job infrastructure
- Chat-native messaging, delivery tracking, offline sync engine
- WebRTC via mediasoup SFU and coturn STUN/TURN
- TimescaleDB telemetry ingestion
- Prometheus metrics and Pino structured logging
- Multi-station federation (Phase 4)
- Docker Compose development environment
- CI pipeline (lint, type-check, tests)

**Out of scope:**
- Frontend clients (`poc-hermes-app`, `hermes-chat`, `hermes-gps`) — compatibility is a constraint, not a deliverable
- Amateur radio licensing or regulatory compliance
- Cryptographic HF modem implementation (UUCP/HMP engine is consumed, not built)

### 1.4 Stakeholders

| Role | Responsibility |
|---|---|
| Engineering Lead | Architecture decisions, ADR ownership, phase gate approval |
| Backend Engineers | Feature implementation, unit and integration tests |
| QA / Test Engineer | Integration, system, and UAT test design and execution |
| DevOps | Docker Compose, CI/CD pipelines, deployment runbooks |
| Radio Operator SME | Acceptance criteria for HAL, PTT safety, frequency management |

---

## 2. Functional Requirements

### 2.1 Authentication and Authorization

| ID | Requirement | Priority |
|---|---|---|
| FR-A-01 | System shall authenticate users via callsign/password, returning a JWT RS256 access token (15 min TTL) and a refresh token (30 days) | Must |
| FR-A-02 | Refresh token rotation shall be mandatory on every `/auth/refresh` call — non-rotating tokens are a security defect (Audit H-4) | Must |
| FR-A-03 | System shall enforce RBAC with four roles: `admin`, `operator`, `user`, `readonly` | Must |
| FR-A-04 | All WebSocket connections shall re-authenticate by sending a valid JWT within 10 seconds of connect, or be closed with code 4001 | Must |
| FR-A-05 | WebSocket sessions shall be re-validated against `user_sessions` table periodically — expired or revoked sessions shall receive `SESSION_INVALIDATED` and be closed (Audit C-4) | Must |
| FR-A-06 | Rate limiting shall be applied to `/auth/login` and `/auth/refresh` (5 req / 15 min / IP) | Must |
| FR-A-07 | All authentication events shall be written to `audit_logs` | Must |

### 2.2 Hardware Abstraction Layer

| ID | Requirement | Priority |
|---|---|---|
| FR-H-01 | HAL shall expose an `IRadioDriver` interface; all hardware access must go through this interface — no direct CLI calls in domain services | Must |
| FR-H-02 | System shall support two driver implementations: `SBitxCLIDriver` (production) and `SimulatedRadioDriver` (dev/test/CI), selectable via `RADIO_DRIVER` env var | Must |
| FR-H-03 | HAL shall provide a `RadioCommandQueue` backed by BullMQ with concurrency=1 and priority levels | Must |
| FR-H-04 | If Redis is unavailable, the HAL shall fall back to synchronous in-process execution for critical commands (PTT, stop) using an async mutex (Audit H-3) | Must |
| FR-H-05 | All CLI arguments shall be passed as arrays to `execFile` — never string concatenation | Must |
| FR-H-06 | `TelemetryPoller` shall poll the driver at a configurable interval (default: 1000 ms) and publish `radio.telemetry` events on every poll and `radio.state_changed` events on significant state delta | Must |

### 2.3 REST API

| ID | Requirement | Priority |
|---|---|---|
| FR-R-01 | All endpoints shall be versioned under `/api/v1/` | Must |
| FR-R-02 | Every route shall have a JSON Schema declaration for Fastify's built-in AJV validation | Must |
| FR-R-03 | OpenAPI 3.1 documentation shall be auto-generated from route schemas via `@fastify/swagger` | Must |
| FR-R-04 | System shall implement all endpoint groups: auth, users, radio, file, system, geolocation, frequency, caller/schedules, wifi, customer errors, and a messaging compatibility shim | Must |
| FR-R-05 | Compatibility shim endpoints (`/api/v1/message/inbox`, `/api/v1/message/outbox`, `/api/v1/message/send`) shall be provided for `poc-hermes-app` and marked deprecated in OpenAPI docs | Must |

### 2.4 WebSocket Gateway

| ID | Requirement | Priority |
|---|---|---|
| FR-W-01 | Gateway shall accept connections at `wss://host/ws` with subprotocol `hermes-v1` | Must |
| FR-W-02 | All messages shall use the `WsEnvelope` format with `type`, `payload`, `requestId`, `timestamp`, and `version` fields (Audit M-4) | Must |
| FR-W-03 | Rate limiting shall be applied per connection: 60 messages/min, 10 RADIO_COMMAND/min | Must |
| FR-W-04 | Heartbeat shall be 30 s server PING / 10 s client PONG timeout | Must |
| FR-W-05 | Gateway shall support topic-based subscriptions — clients receive only subscribed topics | Must |

### 2.5 Event-Driven Architecture

| ID | Requirement | Priority |
|---|---|---|
| FR-E-01 | All inter-module events shall be published and consumed only through the `IEventBus` interface | Must |
| FR-E-02 | Phase 1–3: `RedisEventBus` (Redis Pub/Sub) shall implement `IEventBus` | Must |
| FR-E-03 | Phase 4: `NatsEventBus` (NATS JetStream) shall implement `IEventBus` with `subscribeExclusive()` for worker-style consumers (Audit C-5) | Must |
| FR-E-04 | All `EventEnvelope<T>` objects shall carry `id`, `topic`, `payload`, `timestamp`, and `version` fields | Must |

### 2.6 Messaging and Delivery

| ID | Requirement | Priority |
|---|---|---|
| FR-M-01 | System shall implement the chat-native conversation model with types: `direct`, `group`, `broadcast`, `radio` | Must |
| FR-M-02 | Message sending shall be idempotent via `clientMessageId` | Must |
| FR-M-03 | Delivery tracking shall be per-recipient, per-channel (websocket, push, radio, email) | Must |
| FR-M-04 | A missed-delivery recovery job shall periodically scan for `message_deliveries` with `status=pending` and `created_at < NOW() - INTERVAL '60 seconds'` and re-enqueue delivery (Audit C-1) | Must |
| FR-M-05 | `sync_queue` shall store entity references (`entity_id`, `operation`, `entity_type`) — not serialized payload snapshots; current state shall be fetched at sync time (Audit C-2) | Must |
| FR-M-06 | The sync cursor shall be advanced both after WebSocket push delivery and after sync queue replay, eliminating the duplicate-delivery bug (Audit C-3) | Must |
| FR-M-07 | System shall support message editing, soft deletion, reactions, and reply threading | Must |
| FR-M-08 | HMP/UUCP inbound messages shall be ingested and presented as chat messages in conversations | Must |
| FR-M-09 | Attachments shall be stored with UUID filenames; MIME type shall be detected from content bytes, not filename or HTTP header | Must |

### 2.7 WebRTC

| ID | Requirement | Priority |
|---|---|---|
| FR-V-01 | System shall deploy mediasoup v3 SFU with Opus codec (FEC, DTX enabled) | Must |
| FR-V-02 | coturn STUN/TURN shall be deployed with RFC 1918 denied-peer-ip and HMAC credentials | Must |
| FR-V-03 | TURN credential endpoint `GET /api/v1/webrtc/credentials` shall require at least `user` role and log every issuance (Audit H-6) | Must |
| FR-V-04 | mediasoup worker count shall default to `max(1, min(cpuCount - 2, 4))` and be configurable via `MEDIASOUP_WORKER_COUNT` (Audit M-6) | Must |

### 2.8 Observability

| ID | Requirement | Priority |
|---|---|---|
| FR-O-01 | All log output shall use Pino structured JSON; `console.log` is prohibited in production code | Must |
| FR-O-02 | Pino redact configuration shall cover deeply nested token/password fields (Audit M-7) | Must |
| FR-O-03 | Prometheus metrics shall be exposed at `/metrics` (internal only) for all key subsystems | Must |
| FR-O-04 | OpenTelemetry distributed tracing shall be initialized at startup | Must |
| FR-O-05 | A Grafana dashboard shall be provisioned at `infrastructure/observability/dashboards/` | Should |

---

## 3. Non-Functional Requirements

### 3.1 Performance

| ID | Requirement | Target |
|---|---|---|
| NFR-P-01 | HTTP API p99 latency for standard read endpoints | < 100 ms under normal load |
| NFR-P-02 | Unread count query on a 10,000-message conversation | < 20 ms |
| NFR-P-03 | Sync delta query for 500 queued events | < 100 ms |
| NFR-P-04 | 50 concurrent WebSocket clients at 10 messages/second | 0 frame drops |
| NFR-P-05 | `last_message_id` on `conversations` shall be computed on-read via index, not denormalized (Audit H-1) | < 5 ms on indexed column |

### 3.2 Reliability

| ID | Requirement |
|---|---|
| NFR-R-01 | No message delivery may be silently lost due to a process crash between DB write and event publish (Audit C-1 must be resolved) |
| NFR-R-02 | PTT and radio stop commands must always be executable even when Redis is unavailable |
| NFR-R-03 | All database operations must go through the repository layer — no direct SQL in route handlers |
| NFR-R-04 | All undelivered messages must be durably queued in BullMQ + PostgreSQL |

### 3.3 Security

| ID | Requirement |
|---|---|
| NFR-S-01 | Zero `npm audit` high/critical vulnerabilities at every phase gate |
| NFR-S-02 | JWT must use RS256 — HS256 is prohibited |
| NFR-S-03 | All refresh token operations shall be logged with `auth_refresh_token_reuse_detected_total` metric |
| NFR-S-04 | All CLI arguments to sBitx binaries shall be passed as arrays |
| NFR-S-05 | Stored filenames shall be UUIDs — client-provided filenames shall never reach the filesystem |
| NFR-S-06 | Audit log action strings shall be defined as a TypeScript `as const` record (Audit L-4) |

### 3.4 Maintainability

| ID | Requirement |
|---|---|
| NFR-M-01 | `tsc --strict` must produce zero errors at all times |
| NFR-M-02 | No `any` types — use proper TypeScript generics and discriminated unions |
| NFR-M-03 | All public interfaces shall have JSDoc comments documenting their contract |
| NFR-M-04 | Environment variables shall be accessed only through `src/shared/config.ts` |
| NFR-M-05 | All new modules shall be wired via dependency injection — no leaked singletons |
| NFR-M-06 | Attachment storage shall be abstracted behind `IStorageBackend`; `LocalFileStorageBackend` for Phase 1–3, `MinioStorageBackend` for Phase 4 (Audit H-5) |

### 3.5 Testability

| ID | Requirement |
|---|---|
| NFR-T-01 | `npm test` (Vitest) shall pass at every commit |
| NFR-T-02 | New modules shall have ≥ 80% test coverage |
| NFR-T-03 | `SimulatedRadioDriver` shall implement the full `IRadioDriver` contract test suite |
| NFR-T-04 | CI shall run lint, type-check, and tests on every PR |

### 3.6 Target Hardware

The system is designed to operate on a Raspberry Pi 4 (4 GB RAM) in field conditions. CPU and memory budgets are constrained. Architecture decisions (mediasoup worker count, TimescaleDB background workers) must reflect this constraint.

---

## 4. System Architecture and Technical Design

### 4.1 Architectural Style

**Modular monolith with event-driven core.** Modules have explicit boundaries and communicate through typed interfaces and internal events. The design enables future extraction of independent services (WebRTC SFU, sync engine, HAL daemon) without rewriting contracts.

### 4.2 Technology Stack (Locked)

| Layer | Technology | Version |
|---|---|---|
| Runtime | Node.js | 22 LTS |
| Language | TypeScript | 5 (strict mode) |
| HTTP Framework | Fastify | v5 |
| Database | PostgreSQL + TimescaleDB | 17 |
| Cache / Pub-Sub | Redis | 7 |
| Job Queue | BullMQ | latest |
| ORM / Migrations | Drizzle ORM + drizzle-kit | latest |
| Auth | JWT RS256 | — |
| WebSocket | `ws` library | — |
| SFU | mediasoup | v3 |
| STUN/TURN | coturn | latest |
| Event Bus (Ph 1–3) | Redis Pub/Sub via `IEventBus` | — |
| Event Bus (Ph 4) | NATS JetStream | — |
| Logger | Pino | latest |
| Observability | OpenTelemetry + Prometheus | — |
| Testing | Vitest + Supertest | — |
| Linting | ESLint + Prettier + commitlint | — |
| Containerization | Docker + Docker Compose | — |

### 4.3 Source Directory Layout

```
src/
  api/             Fastify routes, plugins, middleware
  modules/         Domain modules (radio, messaging, geolocation, ...)
  hal/             Hardware Abstraction Layer
  events/          IEventBus, RedisEventBus, NatsEventBus, topics.ts
  jobs/            BullMQ workers
  db/              Drizzle schema, migrations, repositories
  shared/          config.ts, logger.ts, utilities
infrastructure/
  docker/
  nginx/
  observability/   Grafana dashboards, Prometheus config
docs/
  architecture/
  adr/
  roadmap/
  plan/            This document and phase task lists
  tasks/           Per-phase task lists
```

### 4.4 Key Architectural Decisions

| ADR | Decision |
|---|---|
| ADR-001 | Fastify v5 as HTTP framework (type safety, schema-first, performance) |
| ADR-002 | PostgreSQL 17 + TimescaleDB (normalization, time-series, no SQLite) |
| ADR-003 | Redis Pub/Sub (Phase 1–3) → NATS JetStream (Phase 4); `IEventBus` abstraction with both `subscribe()` and `subscribeExclusive()` |
| ADR-004 | WebSocket `ws` library with `hermes-v1` subprotocol (full protocol control) |
| ADR-005 | Chat-native conversation model replacing email inbox/outbox |
| ADR-006 | Hardware Abstraction Layer with `IRadioDriver` interface |
| ADR-007 | mediasoup SFU + coturn for WebRTC |

---

## 5. User Stories and Use Cases

### Epic E-1: Authentication

| ID | Story | Role | Acceptance Criteria |
|---|---|---|---|
| US-A-01 | As a radio operator, I want to log in with my callsign and password so that I can access the system | operator | Returns JWT access token + refresh token; refresh token stored as SHA-256 hash; audit log entry created |
| US-A-02 | As a logged-in user, I want to refresh my access token without re-entering my password | user | New access token issued; old refresh token invalidated (mandatory rotation); reuse of revoked token is detected and alerted |
| US-A-03 | As an admin, I want to suspend a user so that their active sessions are immediately terminated | admin | User status set to `suspended`; all active WebSocket connections receive `SESSION_INVALIDATED` within next re-validation window |
| US-A-04 | As the system, I want to enforce RBAC on every API endpoint so that unauthorized actions are blocked | system | `403 FORBIDDEN` with audit log entry for every unauthorized access attempt |

### Epic E-2: Radio Control

| ID | Story | Role | Acceptance Criteria |
|---|---|---|---|
| US-R-01 | As an operator, I want to set the radio frequency so that I can tune to the correct band | operator | Frequency persisted to profile; HAL command executed within 2 s; audit log entry created |
| US-R-02 | As an operator, I want to activate PTT so that I can transmit audio | operator | PTT activated via HAL; TX state reflected in telemetry; HAL circuit-breaker allows PTT even if Redis is down |
| US-R-03 | As any user, I want to view real-time radio telemetry so that I can monitor station status | user | SNR, bitrate, TX/RX state updated at 1 s interval via WebSocket `RADIO_TELEMETRY` frames |
| US-R-04 | As an operator, I want to send a radio command via WebSocket so that I don't need to use REST | operator | `RADIO_COMMAND` accepted; correlated `RADIO_COMMAND_RESULT` returned with `requestId` |
| US-R-05 | As a developer, I want to run the server without real radio hardware so that I can develop and test | developer | `RADIO_DRIVER=simulated` starts `SimulatedRadioDriver`; all driver contract tests pass |

### Epic E-3: Messaging

| ID | Story | Role | Acceptance Criteria |
|---|---|---|---|
| US-M-01 | As a user, I want to start a direct conversation with another user so that we can chat privately | user | `POST /api/v1/conversations` creates `direct` conversation; both participants' unread count starts at 0 |
| US-M-02 | As a user, I want to send a message and know it was delivered to online recipients | user | `MESSAGE_NEW` WebSocket frame received by all online participants; `message_deliveries.status=delivered` after `MESSAGE_ACK` |
| US-M-03 | As an offline user who reconnects, I want to receive all messages I missed | user | `SYNC_REQUEST` → `SYNC_DELTA` frames deliver all missed events; no duplicates (cursor correctly advanced for both online push and sync replay) |
| US-M-04 | As a user, I want to react to a message with an emoji | user | Reaction persisted; `REACTION_ADDED` pushed to all conversation subscribers |
| US-M-05 | As a sender, I want to edit a message within the allowed time window | user | Message content updated; all online subscribers receive `MESSAGE_UPDATED`; offline clients receive current (not stale) content via sync |
| US-M-06 | As a remote station user, I want to receive messages from other stations via HF radio | user | HMP/UUCP inbound message ingested as a chat message; delivery tracking records created |

### Epic E-4: File Attachments

| ID | Story | Role | Acceptance Criteria |
|---|---|---|---|
| US-F-01 | As a user, I want to upload a file attachment to a message | user | File uploaded; MIME detected from bytes; UUID filename stored; preview generated by worker |
| US-F-02 | As a user, I want to download an attachment I have access to | user | File served with correct `Content-Type` from stored MIME; access check enforces conversation membership; no path traversal possible |

### Epic E-5: WebRTC Audio

| ID | Story | Role | Acceptance Criteria |
|---|---|---|---|
| US-V-01 | As an operator, I want to start a voice call over the LAN with another station user | operator | WebRTC session created via signaling; Opus audio flows bidirectionally; session linked to a `radio` conversation |
| US-V-02 | As an operator, I want to hear HF radio audio in my browser | operator | Radio-to-WebRTC bridge streams ALSA audio → Opus RTP → mediasoup → browser (Phase 4) |

### Epic E-6: Federation (Phase 4)

| ID | Story | Role | Acceptance Criteria |
|---|---|---|---|
| US-Fed-01 | As a user at station A, I want to send a message to a user at station B over the internet | user | Message routed via `FederationDeliveryWorker`; NATS JetStream delivers to station B; delivery acknowledgment returned |
| US-Fed-02 | As an operator, I want to see which remote stations are currently online | operator | Station heartbeats visible at `GET /api/v1/federation/stations` |

---

## 6. Development Phases and Milestones

```
Timeline (Weeks 1–28)
────────────────────────────────────────────────────────────────────
Phase 1  [W1 ──────────── W6]   Foundation (REST, DB, HAL, Auth)
Phase 2  [W7 ──────── W12]      Real-Time (WebSocket, EventBus, WebRTC, BullMQ)
Phase 3  [W13 ─────────── W20]  Messaging (Chat model, Sync engine, Attachments)
Phase 4  [W21 ─────────── W28]  Federation (NATS, multi-station, radio bridge)
────────────────────────────────────────────────────────────────────
         │                   │                  │                  │
     M-1 (W6)           M-2 (W12)          M-3 (W20)         M-4 (W28)
```

### Milestone Definitions

| ID | Milestone | Week | Criteria |
|---|---|---|---|
| M-1 | **Foundation Complete** | End of W6 | `npm run build` zero errors; `npm test` ≥ 80% coverage; `docker compose up` starts all services; all `hermes-api` endpoints replaced; HAL operational; JWT auth working; CI passing; poc-hermes-app compatibility gate passed |
| M-2 | **Real-Time Infrastructure Live** | End of W12 | WebSocket gateway handling `hermes-v1`; radio telemetry streaming; BullMQ workers running; mediasoup session create/destroy; coturn deployed; audit C-1 missed-delivery scanner running; audit C-4 WebSocket re-validation running; poc-hermes-app compatibility gate passed |
| M-3 | **Full Messaging Operational** | End of W20 | Chat conversations, delivery, reactions, threading, attachments, sync engine, HMP/UUCP ingest; audit C-2 sync_queue reference fix; audit C-3 cursor advancement fix; zero high/critical npm audit; poc-hermes-app compatibility gate passed |
| M-4 | **Federation Ready** | End of W28 | NATS JetStream migration; multi-station message delivery; radio-to-WebRTC bridge; group calls; station discovery; poc-hermes-app compatibility gate passed |

---

## 7. Detailed Task Breakdown — Epics and Features

> Full task-level detail is in the per-phase task lists:
> - [Phase 1 Task List](../tasks/phase-1-tasklist.md)
> - [Phase 2 Task List](../tasks/phase-2-tasklist.md)
> - [Phase 3 Task List](../tasks/phase-3-tasklist.md)
> - [Phase 4 Task List](../tasks/phase-4-tasklist.md)

### Epic Summary

| Epic ID | Name | Phase | Complexity | Priority |
|---|---|---|---|---|
| EP-01 | Project Scaffold & Infrastructure | 1 | Medium | Critical |
| EP-02 | Database Schema & Repository Layer | 1 | High | Critical |
| EP-03 | JWT Authentication & RBAC | 1 | High | Critical |
| EP-04 | Hardware Abstraction Layer (HAL) | 1 | High | Critical |
| EP-05 | REST API — All hermes-api Endpoints | 1 | High | Critical |
| EP-06 | IEventBus & RedisEventBus | 2 | Medium | Critical |
| EP-07 | WebSocket Gateway (hermes-v1) | 2 | High | Critical |
| EP-08 | Radio Telemetry Streaming | 2 | Medium | High |
| EP-09 | BullMQ Worker Infrastructure | 2 | Medium | High |
| EP-10 | WebRTC (LAN Audio — Phase 2) | 2 | High | High |
| EP-11 | Observability (Prometheus, OTel, Grafana) | 2 | Medium | High |
| EP-12 | Messaging Domain Model & REST API | 3 | High | Critical |
| EP-13 | Delivery Pipeline (Online + Offline) | 3 | High | Critical |
| EP-14 | Audit C-1: Missed Delivery Recovery Scanner | 3 | Medium | Critical |
| EP-15 | Audit C-2/C-3: Sync Engine Correctness | 3 | High | Critical |
| EP-16 | Attachment Pipeline | 3 | Medium | High |
| EP-17 | Reactions, Threading, Typing Indicators | 3 | Medium | High |
| EP-18 | HMP/UUCP Interoperability Shim | 3 | High | High |
| EP-19 | NATS JetStream Migration | 4 | High | Critical |
| EP-20 | Multi-Station Federation | 4 | High | Critical |
| EP-21 | Radio-to-WebRTC Bridge | 4 | Very High | High |
| EP-22 | Group WebRTC Calls | 4 | High | High |
| EP-23 | Security Hardening for Federation | 4 | High | Critical |

---

## 8. Dependency Map

```
EP-01 (Scaffold)
  └──► EP-02 (DB Schema)
        └──► EP-03 (Auth)         ─────────────────────────────────┐
        └──► EP-04 (HAL)          ─────────────────────────────────┤
        └──► EP-05 (REST API)     ← depends on EP-03, EP-04 ───────┤
              └──► M-1 (Gate) ◄──────────────────────────────────── ┘
                   └──► EP-06 (IEventBus)
                         └──► EP-07 (WebSocket Gateway) ← EP-06
                         └──► EP-08 (Telemetry) ← EP-06, EP-04
                         └──► EP-09 (BullMQ Workers) ← EP-06
                               └──► EP-10 (WebRTC) ← EP-07, EP-09
                               └──► EP-11 (Observability)
                                    └──► M-2 (Gate) ◄──────────────┘
                                         └──► EP-12 (Messaging Domain) ← EP-06
                                               └──► EP-13 (Delivery) ← EP-12, EP-09
                                               └──► EP-14 (C-1 Fix) ← EP-13
                                               └──► EP-15 (C-2/C-3 Sync) ← EP-12
                                               └──► EP-16 (Attachments) ← EP-12
                                               └──► EP-17 (Reactions) ← EP-12
                                               └──► EP-18 (HMP/UUCP) ← EP-12
                                                    └──► M-3 (Gate) ◄─┘
                                                         └──► EP-19 (NATS)
                                                               └──► EP-20 (Federation) ← EP-19
                                                               └──► EP-21 (Radio Bridge) ← EP-10
                                                               └──► EP-22 (Group WebRTC) ← EP-21
                                                               └──► EP-23 (Security) ← EP-20
                                                                    └──► M-4 (Gate)
```

**Critical Path**: EP-01 → EP-02 → EP-03 → EP-05 → M-1 → EP-06 → EP-07 → M-2 → EP-12 → EP-13 → EP-15 → M-3 → EP-19 → EP-20 → M-4

---

## 9. Testing Strategy

### 9.1 Testing Pyramid

```
           ┌───────────────────────────┐
           │    UAT / E2E (Cypress)    │  ← poc-hermes-app compatibility tests
           ├───────────────────────────┤
           │  System / Load Tests      │  ← k6 / autocannon; performance baselines
           ├───────────────────────────┤
           │  Integration Tests        │  ← Supertest; real PG + Redis in Docker
           ├───────────────────────────┤
           │  Unit Tests (Vitest)      │  ← Largest tier; mocked deps; fast
           └───────────────────────────┘
```

### 9.2 Unit Testing

**Framework**: Vitest  
**Coverage target**: ≥ 80% on all new modules  
**Scope**:
- All repository query functions
- Auth middleware (JWT verify, RBAC enforce)
- HAL driver contract tests — `SimulatedRadioDriver` must pass all tests that `SBitxCLIDriver` is expected to pass
- EventBus publish/subscribe semantics
- `SyncService` cursor logic
- `MessagingService` idempotency via `clientMessageId`
- Pino redact coverage: automated assertion that sensitive fields are redacted

**Pattern**: Every new domain module ships with a `*.test.ts` co-located or in `__tests__/`.

### 9.3 Integration Testing

**Framework**: Supertest (HTTP) + native WebSocket client  
**Environment**: Docker Compose — real PostgreSQL, TimescaleDB, Redis  
**Scope**:
- Full authentication flow: login → refresh (rotation validated) → logout
- HAL command via REST and WebSocket, verified in DB and telemetry
- Message send → WebSocket delivery → `MESSAGE_ACK` → delivery status `delivered`
- Offline message scenario: send → disconnect client → reconnect → `SYNC_REQUEST` → `SYNC_DELTA` → no duplicates
- Attachment upload → MIME detection → thumbnail generation → access-controlled download
- HMP ingest → message visible in conversation
- Missed-delivery recovery scanner: verify it re-enqueues orphaned deliveries after crash simulation
- WebRTC session lifecycle: create, transport, produce, consume, end

### 9.4 System / Performance Testing

**Framework**: k6 (load), autocannon (HTTP throughput)  
**Baselines** (validated per NFR section):
- 50 concurrent WebSocket clients, 10 msg/s: 0 frame drops
- Sync delta query with 500 queued events: < 100 ms
- Unread count on 10,000-message conversation: < 20 ms
- HTTP p99 latency under 100 RPS: < 100 ms

### 9.5 Security Testing

- `npm audit --audit-level=high` run in CI; gate fails on any high/critical finding
- OWASP checklist applied to auth endpoints at Phase 1 gate
- Rate limit enforcement tested: auth endpoints (5 req/15 min/IP), WebSocket (60 msg/min)
- JWT algorithm enforcement: server must reject tokens signed with HS256
- CLI injection test: numeric and string inputs to HAL driver validated against allowlist before `execFile`

### 9.6 Compatibility Testing with poc-hermes-app

> This is a **mandatory gate** at the end of every phase.

**Scope per phase**:

| Phase | API endpoints to verify | Protocol compatibility |
|---|---|---|
| 1 | All `/api/v1/` REST endpoints used by poc-hermes-app; legacy shim (`/api/v1/message/`) | N/A — no WebSocket in Phase 1 |
| 2 | Telemetry WebSocket topics; `RADIO_COMMAND` flow | `hermes-v1` subprotocol; `WsEnvelope` format |
| 3 | Message delivery, sync, attachment download | Full `hermes-v1` protocol including SYNC_REQUEST/DELTA |
| 4 | Federation routing; NATS migration transparent to app | No API changes visible to app |

**Method**: Run poc-hermes-app against the new server in Docker Compose. Execute the poc-hermes-app existing test suite (or manual smoke test if no automated suite exists). Document any failures and required modifications to either system.

### 9.7 User Acceptance Testing (UAT)

**Owner**: Radio Operator SME + QA  
**Entry criteria**: All integration tests passing; performance baselines met  
**Scenarios**:
1. Full field simulation: operator login, tune frequency, activate PTT, send message to offline station, reconnect and receive
2. Multi-device: same user logged in on two devices; message delivered to both; read receipt propagated
3. Admin user management: create user, assign role, suspend user, verify active session terminated
4. Attachment workflow: upload image, wait for thumbnail, share in conversation, verify download from second device

---

## 10. Risk Assessment and Mitigation

### 10.1 Risk Register

| ID | Risk | Probability | Impact | Severity | Mitigation |
|---|---|---|---|---|---|
| R-01 | **[AUDIT C-1] Process crash orphans message_deliveries** | High (any deployment restart) | Critical (silent message loss) | Critical | Implement missed-delivery recovery scanner in Phase 2; alert on `missed_delivery_recovery_total` > 0 |
| R-02 | **[AUDIT C-2] Edited messages deliver stale content to offline clients** | Medium | High (incorrect UX) | High | Phase 3: redesign sync_queue to store entity references, not payload snapshots |
| R-03 | **[AUDIT C-3] Online clients receive duplicate messages on reconnect** | High (every reconnect) | High (duplicate UX) | High | Phase 3: advance sync cursor on every WebSocket push delivery |
| R-04 | **[AUDIT C-4] JWT-expired WebSocket sessions remain active** | Medium | Critical (security — suspended user keeps access) | Critical | Phase 2: implement periodic session re-validation; `SESSION_INVALIDATED` message type |
| R-05 | **[AUDIT C-5] NATS migration breaks fan-out semantics** | High (incorrect IEventBus design) | High (triple delivery, worker starvation) | High | Fix IEventBus before Phase 4: add `subscribeExclusive()`; document semantic difference |
| R-06 | **[AUDIT H-1] conversations.last_message_id hot-row contention** | Medium (concurrent messages) | High (latency spikes) | High | Phase 1 schema: compute on-read via index; remove denormalized field |
| R-07 | **[AUDIT H-3] Redis unavailability blocks PTT** | Medium (field hardware failures) | Critical (safety: cannot deactivate TX) | Critical | Phase 1 HAL: circuit-breaker fallback with in-process async mutex |
| R-08 | **[AUDIT H-4] Non-rotating refresh tokens — security defect** | Certain (never rotated without fix) | High (30-day stolen token undetectable) | High | Phase 1 auth: mandatory rotation; `auth_refresh_token_reuse_detected_total` metric |
| R-09 | **[AUDIT H-5] No IStorageBackend abstraction** | Certain (not built) | High (Phase 4 federation requires object storage refactor) | High | Phase 1: define `IStorageBackend`; implement `LocalFileStorageBackend` |
| R-10 | **[AUDIT H-6] TURN credential endpoint missing RBAC** | Certain (not defined) | High (any readonly user can use coturn as relay) | High | Phase 2: add to RBAC matrix; embed userId in HMAC; audit log every issuance |
| R-11 | **Raspberry Pi 4 CPU contention** (mediasoup + PG + Redis + Node.js) | High (target hardware) | Medium (degraded performance) | Medium | Phase 2: `MEDIASOUP_WORKER_COUNT = max(1, min(cpus-2, 4))`; TimescaleDB background worker limit |
| R-12 | **poc-hermes-app API contract drift** | Medium (iterative dev) | High (broken client) | High | Compatibility gate at every phase; OpenAPI diff in CI |
| R-13 | **Intermittent HF link loses in-flight messages** | Certain (field conditions) | Medium (expected — handled by sync) | Medium | Offline-first sync engine; `sync_queue` durability; BullMQ retry with exponential backoff |
| R-14 | **TimescaleDB maintenance jobs cause I/O spikes** | Medium (background jobs) | Medium (increased p99 latency) | Medium | Configure `max_background_workers=2`; schedule maintenance windows via pg_cron |

---

## 11. Deployment and Release Strategy

### 11.1 Environments

| Environment | Purpose | Infrastructure |
|---|---|---|
| **Development** | Local development | `docker compose up` (docker-compose.yml) |
| **CI** | Automated test runs | Docker Compose in GitHub Actions |
| **Staging** | Pre-release validation | Raspberry Pi 4 or equivalent VM; `docker-compose.prod.yml` |
| **Production** | Live station deployment | Raspberry Pi 4; `docker-compose.prod.yml` with resource limits |

### 11.2 Release Cadence

Each phase milestone (M-1 through M-4) constitutes a versioned release:

| Release | Version | Deliverable |
|---|---|---|
| Phase 1 gate | v1.0.0 | Foundation; replaces hermes-api |
| Phase 2 gate | v1.1.0 | Real-time WebSocket; WebRTC LAN audio |
| Phase 3 gate | v1.2.0 | Full messaging; offline sync |
| Phase 4 gate | v2.0.0 | Federation; NATS migration (breaking API change to event bus) |

### 11.3 Deployment Procedure

1. All CI checks pass (`build`, `lint`, `test`)
2. Phase compatibility gate passed (poc-hermes-app smoke test)
3. `docker compose pull` on target system
4. `npm run db:migrate` (Drizzle migrations applied)
5. `docker compose up -d` with zero-downtime rolling restart
6. Health check: `GET /health/ready` returns 200
7. Smoke test: login → radio status → send message

### 11.4 Rollback Procedure

1. `docker compose down`
2. Restore previous container image tag via `docker compose up -d`
3. If schema migration was applied: execute rollback migration (`drizzle-kit drop`)
4. Verify health check passes

### 11.5 Database Migration Strategy

- All schema changes via Drizzle ORM migrations in `src/db/migrations/`
- Migrations are additive in Phase 1–3 (no destructive changes)
- No migration may remove a column used by `poc-hermes-app` without a deprecation period
- Migration CI job: fresh PostgreSQL + TimescaleDB, apply all migrations, verify seed data

---

## 12. Documentation Requirements

| Document | Owner | Phase | Location |
|---|---|---|---|
| Architecture Overview | Engineering Lead | Phase 1 | `docs/architecture/overview.md` (update as built) |
| API Reference (OpenAPI) | Auto-generated | All | Served at `/docs` via `@fastify/swagger-ui` |
| Database Schema | Engineering Lead | Phase 1 | `docs/architecture/database-schema.md` (update) |
| WebSocket Protocol Spec | Engineering Lead | Phase 2 | `docs/architecture/websocket-protocol.md` (update) |
| HAL Integration Guide | Backend Engineer | Phase 1 | `docs/architecture/hal.md` (update) |
| Deployment Runbook | DevOps | Phase 1 | `docs/deployment/runbook.md` |
| ADR-008: Federation Auth Model | Engineering Lead | Before Phase 4 | `docs/adr/ADR-008-federation-auth.md` |
| Contributing Guide | Engineering Lead | Phase 1 | `docs/governance/CONTRIBUTING.md` |
| RFC Template (updated with decision authority) | Engineering Lead | Immediately | `docs/governance/RFC-TEMPLATE.md` |
| Performance Baseline Report | QA | Each phase gate | `docs/qa/performance-phase-N.md` |
| poc-hermes-app Compatibility Report | QA | Each phase gate | `docs/qa/compat-phase-N.md` |

---

## 13. Maintenance and Future Enhancement Considerations

### 13.1 Phase 4 Prerequisites to Address Before Design

- **ADR-008** (Federation Authentication): Define station-to-station auth model (shared secrets vs. PKI) before Phase 4 schema design. The `stations` table needs credential fields whose type depends on this decision (Audit L-1).
- **IEventBus `subscribeExclusive()`**: Must be added and tested before NATS migration begins (Audit C-5).
- **WebRTC phase assignment correction**: "Radio audio streamed to browser (receive)" is Phase 4, not Phase 3. Reconcile `docs/architecture/webrtc.md §2` with the Phase 4 roadmap (Audit L-2).

### 13.2 Post-Phase 4 Roadmap Candidates

| Feature | Description | Dependency |
|---|---|---|
| End-to-End Encryption | Optional E2EE via Insertable Streams; radio bridge breaks with E2EE (documented tradeoff) | Phase 4 complete |
| MinIO Object Storage | Federation-ready attachment storage via `MinioStorageBackend` | `IStorageBackend` (Phase 1) |
| CRDT-based Conflict Resolution | Replace last-write-wins for complex entities | Phase 4 sync engine |
| Mobile Push Notifications | FCM/APNs delivery via `PushNotificationWorker` | Phase 3 delivery pipeline |
| Capacitor Mobile Client | Native iOS/Android wrapper for hermes-chat | Phase 3 messaging |
| pg_cron Maintenance Window | Scheduled TimescaleDB compression/retention to off-peak hours | Phase 1 TimescaleDB config |

### 13.3 Technical Debt Tracker

| ID | Item | Created | Target |
|---|---|---|---|
| TD-01 | Legacy API shim (`/api/v1/message/`) marked deprecated — remove in Phase 4 | Phase 1 | Phase 4 |
| TD-02 | `localStorage` referenced in offline-first.md — change to storage-agnostic language | Audit L-3 | Phase 3 docs update |
| TD-03 | RFC template lacks decision authority definition | Audit M-5 | Immediately |
| TD-04 | Audit log action strings must be `as const` record — not loose strings | Audit L-4 | Phase 1 |

---

## 14. poc-hermes-app Compatibility Matrix

> **This section defines the mandatory compatibility contract between `poc-hermes-server` and `poc-hermes-app`.**

### 14.1 API Endpoints Used by poc-hermes-app

The following endpoint groups are known to be consumed by `poc-hermes-app` and must remain compatible:

| Endpoint Group | Status in hermes-api | Status in poc-hermes-server |
|---|---|---|
| `POST /api/v1/auth/login` | Existing | Must be wire-compatible (same request/response shape) |
| `GET /api/v1/radio/status` | Existing | Must return same field names; new fields allowed |
| `POST /api/v1/radio/profiles/:id/frequency` | Existing | Must be wire-compatible |
| `POST /api/v1/radio/ptt` | Existing | Must be wire-compatible |
| `GET /api/v1/message/inbox` | Existing (PHP) | Compatibility shim provided; response shape preserved |
| `POST /api/v1/message/send` | Existing (PHP) | Compatibility shim provided; routes through messaging service |
| `GET /api/v1/geolocation/current` | Existing | Must be wire-compatible |
| `GET /api/v1/sys/stats` | Existing | Must return same field names; new fields allowed |
| `GET /api/v1/caller` | Existing | Must be wire-compatible |
| `GET /api/v1/frequency` | Existing | Must be wire-compatible |

### 14.2 Breaking Change Policy

- **Additive changes** (new fields in response, new endpoints): Always allowed.
- **Renaming or removing response fields**: Requires compatibility shim or deprecation notice ≥ one full phase before removal.
- **Auth format changes**: JWT claim names and token structure must remain identical across all phases.
- **HTTP status codes**: Must not change for any endpoint currently consumed by poc-hermes-app.

### 14.3 Phase Gate — Compatibility Checkpoint Template

At the end of every phase, a compatibility report must be produced and approved before progression:

```
## Compatibility Report — Phase [N] Gate

**Date**: ...
**Server version**: ...
**poc-hermes-app version tested**: ...

### Test Results

| Endpoint | Expected | Actual | Pass/Fail |
|---|---|---|---|
| ...       | ...      | ...    | ...       |

### WebSocket Compatibility (Phase 2+)

| Topic | Expected frames | Actual frames | Pass/Fail |
|---|---|---|---|
| ...    | ...             | ...           | ...       |

### Required poc-hermes-app Modifications

| Change | Reason | Urgency |
|---|---|---|
| ...    | ...    | ...     |

### Sign-off

- [ ] Engineering Lead
- [ ] QA
- [ ] Radio Operator SME (for HAL/radio tests)
```

---

*This document is the authoritative master plan. All per-phase task lists are derived from it and must remain aligned. Any deviation from this plan requires an Engineering Lead decision and an updated ADR or RFC.*
