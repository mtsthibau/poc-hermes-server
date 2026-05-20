# System Architecture Overview

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Mission

The `poc-hermes-server` is the next-generation backend for HERMES stations — a platform that allows remote, rural, and disaster-affected communities to exchange messages, files, audio, and GPS coordinates over HF radio. The server must work reliably with intermittent connectivity, low-bandwidth radio links, and devices operating in harsh field environments.

The architecture is designed for **years of evolution**, not a temporary fix. Every decision prioritizes long-term maintainability, operational resilience, and extensibility over short-term convenience.

---

## 2. Architectural Style: Modular Monolith with Event-Driven Core

For the initial production phase, the system is designed as a **modular monolith** with a clearly defined internal event bus. Each module has explicit boundaries and communicates through typed interfaces and internal events — not direct function calls across modules.

This approach:

- Reduces operational complexity vs. premature microservices decomposition
- Enables straightforward local development and testing
- Allows future extraction of independent services (WebRTC, sync engine, HAL) without rewriting contracts
- Is appropriate for the scale of a single-station or small-cluster deployment

**Future evolution**: As stations federate and traffic grows, individual modules (particularly HAL, messaging, and WebRTC) can be extracted into independent processes or services communicating via NATS JetStream or gRPC.

---

## 3. System Boundary Diagram

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                            External Clients                               ║
║  ┌──────────────┐  ┌────────────────┐  ┌────────────┐  ┌──────────────┐  ║
║  │ hermes-chat  │  │  hermes-gps    │  │ Mobile App │  │  Radio Node  │  ║
║  │ (Next.js)    │  │  (Next.js)     │  │ (Capacitor)│  │  (UUCP/HMP) │  ║
║  └──────┬───────┘  └───────┬────────┘  └─────┬──────┘  └──────┬───────┘  ║
╚═════════╪══════════════════╪═════════════════╪════════════════╪══════════╝
          │ HTTPS/REST        │ HTTPS/REST       │ WSS            │ TCP/UUCP
╔═════════╪══════════════════╪═════════════════╪════════════════╪══════════╗
║         ▼                  ▼                 ▼                ▼          ║
║  ┌──────────────────────────────────────────────────────────────────────┐ ║
║  │                      API Gateway / Load Balancer                     │ ║
║  │                     (nginx / caddy — TLS termination)               │ ║
║  └────────┬─────────────────────────┬────────────────────────┬──────────┘ ║
║           │                         │                        │            ║
║  ┌────────▼──────────┐  ┌───────────▼──────────┐  ┌─────────▼──────────┐ ║
║  │   REST API        │  │  WebSocket Gateway   │  │  WebRTC Signaling  │ ║
║  │   (Fastify v5)    │  │  (typed events)      │  │  + mediasoup SFU   │ ║
║  │   /api/v1/*       │  │  wss://host/ws       │  │  (port 40000-49999)│ ║
║  └────────┬──────────┘  └───────────┬──────────┘  └─────────┬──────────┘ ║
║           │                         │                        │            ║
║  ╔════════╪═════════════════════════╪════════════════════════╪══════════╗ ║
║  ║        ▼    Internal Event Bus   ▼    (Redis Pub/Sub)     ▼          ║ ║
║  ║  ┌───────────────────────────────────────────────────────────────┐   ║ ║
║  ║  │              EventBus (topics: radio.*, message.*, sync.*)    │   ║ ║
║  ║  └───┬───────────────┬───────────────────┬──────────────────┬────┘   ║ ║
║  ║      │               │                   │                  │         ║ ║
║  ║  ┌───▼────┐  ┌───────▼──────┐  ┌─────────▼──────┐  ┌───────▼──────┐ ║ ║
║  ║  │Messaging│  │Radio/Telemetry│  │ Sync Engine   │  │Geolocation   │ ║ ║
║  ║  │Service │  │Service        │  │ (BullMQ)      │  │Service       │ ║ ║
║  ║  └───┬────┘  └───────┬───────┘  └─────────┬──────┘  └──────────────┘ ║ ║
║  ║      │               │                     │                           ║ ║
║  ║  ┌───▼───────────────▼─────────────────────▼──────────────────────┐   ║ ║
║  ║  │                  Hardware Abstraction Layer (HAL)               │   ║ ║
║  ║  │     IRadioDriver { getStatus, setFreq, setMode, setProfile...}  │   ║ ║
║  ║  │     ┌──────────────────────┐   ┌──────────────────────────┐    │   ║ ║
║  ║  │     │  SBitxCLIDriver      │   │  SimulatedRadioDriver    │    │   ║ ║
║  ║  │     │  (production)        │   │  (dev/test)              │    │   ║ ║
║  ║  │     └──────────────────────┘   └──────────────────────────┘    │   ║ ║
║  ║  └────────────────────────────┬───────────────────────────────────┘   ║ ║
║  ╚═══════════════════════════════╪══════════════════════════════════════╝ ║
║                                  │ sbitx_client / ubitx_client CLI       ║
║  ╔═══════════════════════════════╪══════════════════════════════════════╗ ║
║  ║         sBitx Hardware        ▼   (HF Radio + Modem)                ║ ║
║  ╚══════════════════════════════════════════════════════════════════════╝ ║
║                                                                           ║
║  ┌─────────────────────────────────────────────────────────────────────┐ ║
║  │                         Persistence Layer                           │ ║
║  │  ┌──────────────────┐   ┌──────────────┐   ┌──────────────────────┐│ ║
║  │  │  PostgreSQL 17   │   │  TimescaleDB │   │       Redis 7        ││ ║
║  │  │  (Drizzle ORM)   │   │  (telemetry) │   │  (cache, pub/sub,    ││ ║
║  │  │  primary store   │   │  hypertables │   │   queues, sessions)  ││ ║
║  │  └──────────────────┘   └──────────────┘   └──────────────────────┘│ ║
║  └─────────────────────────────────────────────────────────────────────┘ ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## 4. Module Responsibilities

### 4.1 REST API (`src/api/`)

Fastify-based HTTP API serving all synchronous operations.

**Responsibilities:**
- Authentication and authorization
- Radio control and configuration
- Conversation and message management
- User and device management
- File/attachment handling
- System configuration and diagnostics
- Geolocation data
- Frequency management
- Health and metrics endpoints

**Key design decisions:**
- All routes versioned under `/api/v1/`
- JSON Schema validation at request boundary via Fastify's built-in AJV integration
- OpenAPI 3.1 spec auto-generated from route schemas (`@fastify/swagger`)
- RBAC enforced via middleware before reaching controllers
- Controllers are thin — they delegate to domain services
- No business logic in route handlers

### 4.2 WebSocket Gateway (`src/gateway/`)

Typed, event-driven WebSocket server for real-time communication.

**Responsibilities:**
- Real-time radio telemetry streaming
- Real-time message event delivery
- Presence and typing indicators
- Device synchronization events
- Authentication via initial handshake
- Topic-based subscription management
- Connection lifecycle (connect, authenticate, subscribe, disconnect, resume)
- Backpressure handling

**Key design decisions:**
- Uses the `ws` library — not Socket.IO — for full protocol control
- Custom typed protocol with `type`, `payload`, `requestId`, `timestamp` envelope
- Clients subscribe to specific topics — no broadcast to all
- Every connection must authenticate within 10 seconds or is dropped
- Heartbeat/ping-pong with 30s interval, 10s timeout
- Redis Pub/Sub bridges the HAL event stream to connected WebSocket clients
- See [WebSocket Protocol Specification](websocket-protocol.md) for full protocol

### 4.3 Hardware Abstraction Layer (`src/hal/`)

Isolates all hardware-specific communication behind a typed interface.

**Responsibilities:**
- Execute sBitx CLI commands (typed, async, queued)
- Normalize hardware responses into typed domain objects
- Emit internal events when hardware state changes
- Provide telemetry streaming to internal subscribers
- Support command queues with retry and timeout
- Support future alternative radio drivers

**Key design decisions:**
- `IRadioDriver` interface is the only contract upper layers use
- `SBitxCLIDriver` implements it using `sbitx_client` / `ubitx_client` CLI
- `SimulatedRadioDriver` enables full development and testing without hardware
- Commands go through a BullMQ queue for ordering and retry guarantees
- Telemetry is polled at configurable interval (default: 1s) and emitted as normalized events
- See [Hardware Abstraction Layer](hal.md) for full design

### 4.4 Messaging Domain (`src/messaging/`)

Chat-native messaging system with email interoperability.

**Responsibilities:**
- Conversation lifecycle management
- Message creation, editing, deletion
- Delivery status tracking (sent → delivered → read)
- Reactions and replies
- Attachment coordination
- Transport-layer abstraction (chat, email, radio/HMP)
- Synchronization support

**Key design decisions:**
- Messages are NOT inbox/outbox — they belong to conversations
- Delivery receipts are tracked per-recipient per-channel
- Transport envelopes allow email-compatible metadata without polluting the message model
- See [Messaging Domain Model](messaging-domain.md) for full design

### 4.5 WebRTC Module (`src/webrtc/`)

Real-time audio (and future video) communication using mediasoup SFU.

**Responsibilities:**
- WebRTC signaling (offer/answer/ICE via WebSocket)
- mediasoup router and transport management
- Session lifecycle (create, join, leave)
- Radio-to-WebRTC audio bridging
- STUN/TURN coordination

**Key design decisions:**
- SFU model (not mesh) for scalability beyond 2 participants
- Signaling delivered over the existing WebSocket gateway (new message types)
- coturn handles STUN/TURN for NAT traversal
- See [WebRTC Architecture](webrtc.md) for full design

### 4.6 Event Bus (`src/events/`)

Internal pub/sub backbone for decoupled module communication.

**Responsibilities:**
- Publish normalized events from HAL, messaging, sync
- Fan out events to WebSocket gateway, jobs, other services
- Provide type-safe event publishing and subscription API

**Implementation:** Redis Pub/Sub with typed topic channels. See [Event-Driven Backend](event-driven.md).

### 4.7 Sync Engine (`src/jobs/`)

Offline-first synchronization and background processing.

**Responsibilities:**
- Process outgoing message delivery queues
- Retry failed radio transmissions
- Synchronize device state for reconnecting clients
- Manage UUCP job scheduling
- Clean expired data, attachment lifecycle, telemetry retention

**Implementation:** BullMQ workers backed by Redis. See [Offline-First & Synchronization](offline-first.md).

---

## 5. Data Flow Examples

### 5.1 Radio Telemetry → Client

```
sBitx Hardware
  → [SBitxCLIDriver polls sbitx_client every 1s]
  → HAL normalizes response into RadioTelemetryEvent
  → EventBus publishes to topic: radio.telemetry
  → WebSocket Gateway subscriber receives event
  → Gateway sends RADIO_TELEMETRY frame to all subscribed connections
  → Client updates UI
```

### 5.2 Client Sends Message

```
Client (REST POST /api/v1/conversations/:id/messages)
  → Fastify validates request schema
  → Auth middleware verifies JWT, checks RBAC
  → MessageController delegates to MessagingService
  → MessagingService persists message (status: sending)
  → MessagingService publishes message.new event to EventBus
  → EventBus fans out:
      → WebSocket Gateway → pushes MESSAGE_NEW to subscribed recipients
      → DeliveryWorker (BullMQ) → enqueues delivery jobs per transport
      → (if offline) → SyncQueue stores for later delivery
  → API returns 201 with message object
```

### 5.3 Radio Command from Client

```
Client (REST POST /api/v1/radio/profile/1/frequency)
  → Fastify validates schema
  → RadioController delegates to RadioService
  → RadioService calls HAL.setFrequency(freq, profileId)
  → HAL enqueues command in BullMQ RadioCommandQueue
  → CommandWorker dequeues, executes: sbitx_client set_frequency <freq> <profile>
  → Response normalized, HAL emits radio.state_changed event
  → EventBus → WebSocket Gateway → pushes RADIO_STATE_CHANGE to subscribers
  → API returns 200 with updated radio state
```

### 5.4 Offline Message Sync on Reconnect

```
Client reconnects (WebSocket AUTHENTICATE + SYNC_REQUEST with cursor)
  → Gateway authenticates connection
  → Gateway sends SYNC_REQUEST to SyncService
  → SyncService queries sync_cursors for last known position
  → SyncService fetches delta: messages, reactions, delivery updates since cursor
  → Gateway streams SYNC_DELTA frames to client
  → Client applies delta to local state
  → Gateway sends SYNC_COMPLETE
  → Client updates cursor
```

---

## 6. Cross-Cutting Concerns

### 6.1 Authentication & Authorization

- JWT RS256 tokens with 15-minute expiry
- Refresh tokens stored in Redis (hashed), 30-day expiry
- All REST endpoints protected (except `/health`, `/version`, `/auth/login`)
- WebSocket connections must authenticate within 10 seconds via `AUTHENTICATE` frame
- RBAC roles: `admin`, `operator`, `user`, `readonly`
- See [Security Architecture](security.md)

### 6.2 Validation

- All REST request/response schemas defined as JSON Schema in route definitions
- Fastify's AJV integration validates at request boundary — no manual validation in controllers
- Unknown fields are stripped (Fastify's `additionalProperties: false` globally)
- WebSocket message frames validated against typed schemas at gateway boundary

### 6.3 Error Handling

- All errors derive from a base `HermesError` class with `code`, `message`, `details`
- Fastify error handler serializes all errors to RFC 7807 Problem Detail format
- Business logic errors use specific error codes (e.g., `RADIO_COMMAND_TIMEOUT`, `CONVERSATION_NOT_FOUND`)
- HAL errors are wrapped and never expose raw CLI stderr to clients

### 6.4 Observability

- **Logging**: Pino structured JSON with request-id correlation
- **Tracing**: OpenTelemetry auto-instrumentation (Fastify, PostgreSQL, Redis, BullMQ)
- **Metrics**: Prometheus metrics endpoint at `/metrics`
- See [Observability & Reliability](observability.md)

---

## 7. Deployment Architecture

### Development

```
docker compose up
  → postgres (port 5432)
  → redis (port 6379)
  → coturn (ports 3478, 5349)
  → poc-hermes-server (port 3000)
  → SimulatedRadioDriver (no hardware needed)
```

### Production (Single Station)

```
nginx (TLS termination, reverse proxy)
  → poc-hermes-server (systemd service or Docker)
  → PostgreSQL (local or remote)
  → Redis (local, with persistence enabled)
  → coturn (STUN/TURN server)
  → sBitx hardware (via local CLI)
```

### Future: Multi-Station / Federated

```
NATS JetStream cluster (message broker)
  → Station A: poc-hermes-server (publishes/subscribes)
  → Station B: poc-hermes-server (publishes/subscribes)
  → Station C: poc-hermes-server (publishes/subscribes)
  → Shared PostgreSQL (or station-local with replication)
```

---

## 8. Legacy Compatibility

The system provides a compatibility shim during migration:

- All existing `hermes-api` REST endpoints preserved under `/api/v1/` with identical responses
- The `sbitx_websocket.c` C daemon can be gradually replaced — the new WebSocket gateway initially proxies its telemetry stream while the HAL is being hardened
- UUCP / HMP message flows are preserved through the sync engine and transport adapters

---

## 9. Related Documents

| Document | Description |
|---|---|
| [Database Schema](database-schema.md) | Full normalized schema with indexes and rationale |
| [WebSocket Protocol](websocket-protocol.md) | Complete protocol specification |
| [HAL Design](hal.md) | Hardware abstraction layer architecture |
| [Messaging Domain](messaging-domain.md) | Chat-native messaging with email interoperability |
| [WebRTC Architecture](webrtc.md) | Audio communication architecture |
| [Event-Driven Backend](event-driven.md) | Event bus, topics, worker architecture |
| [Security Architecture](security.md) | Auth, RBAC, transport security, audit |
| [Observability](observability.md) | Logging, tracing, metrics, alerting |
| [Offline-First](offline-first.md) | Sync engine, queues, conflict resolution |
