# poc-hermes-server

> **HERMES — High-frequency Emergency and Rural Multimedia Exchange System**
> Next-generation communication platform server

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE)
[![Node.js](https://img.shields.io/badge/Node.js-22%20LTS-green.svg)](https://nodejs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg)](https://www.typescriptlang.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-blue.svg)](https://www.postgresql.org)

---

## Overview

`poc-hermes-server` is a ground-up redesign of the backend infrastructure powering Hermes communication stations. The platform enables communities in remote or disaster-affected areas to exchange messages, files, audio, and coordinates over HF radio (3–30 MHz shortwave), where no internet, cell towers, or satellites are available.

This project replaces the legacy PHP/Lumen `hermes-api` and the C-embedded `sbitx_websocket` server with a production-grade, event-driven, realtime-capable, offline-first backend architecture.

---

## What Changed & Why

| Concern | Legacy System | New Architecture |
|---|---|---|
| Framework | PHP Lumen | Node.js 22 + TypeScript + Fastify |
| Database | SQLite | PostgreSQL 17 + TimescaleDB + Redis |
| Messaging model | Email-oriented (inbox/outbox, HMP, UUCP) | Chat-native with email interoperability |
| WebSocket | C daemon (Mongoose), flat poll broadcast | Typed event-driven gateway, subscriptions |
| Hardware access | Direct CLI calls in controllers | Hardware Abstraction Layer (HAL) |
| Real-time | None | WebSocket gateway + Redis Pub/Sub |
| Authentication | Minimal/none on WebSocket | JWT RS256, per-connection WS auth |
| Observability | None | Pino + OpenTelemetry + Prometheus |
| Offline-first | None | Sync engine with queues and cursors |
| WebRTC | None | mediasoup SFU + coturn STUN/TURN |
| Testing | Minimal | Vitest + Supertest integration tests |
| Schema | Flat, denormalized | Fully normalized, migration-managed |

---

## Architecture Summary

```
┌───────────────────────────────────────────────────────────────────────┐
│                  Clients (hermes-chat, hermes-gps, mobile)            │
└────────────┬──────────────────────┬──────────────────┬───────────────┘
             │ HTTPS REST           │ WSS              │ WebRTC (SRTP)
┌────────────▼──────────────────────▼──────────────────▼───────────────┐
│                        poc-hermes-server                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────────┐ │
│  │   REST API       │  │  WebSocket       │  │  WebRTC Signaling   │ │
│  │   (Fastify)      │  │  Gateway         │  │  + mediasoup SFU    │ │
│  └────────┬─────────┘  └────────┬─────────┘  └──────────┬──────────┘ │
│           │                     │                        │            │
│  ┌────────▼─────────────────────▼────────────────────────▼──────────┐ │
│  │                Internal Event Bus (Redis Pub/Sub)                 │ │
│  └────────┬─────────────────────┬────────────────────────┬──────────┘ │
│           │                     │                        │            │
│  ┌────────▼────────┐  ┌─────────▼──────────┐  ┌─────────▼─────────┐ │
│  │ Messaging       │  │ Radio/Telemetry     │  │ Sync Engine       │ │
│  │ Domain          │  │ Service             │  │ (BullMQ)          │ │
│  └────────┬────────┘  └─────────┬──────────┘  └─────────┬─────────┘ │
│           │                     │                        │            │
│  ┌────────▼─────────────────────▼────────────────────────▼──────────┐ │
│  │              Hardware Abstraction Layer (HAL)                     │ │
│  │         IRadioDriver → SBitxCLIDriver / SimulatedDriver          │ │
│  └────────────────────────────┬──────────────────────────────────────┘ │
└───────────────────────────────┼───────────────────────────────────────┘
                                │ sBitx CLI
┌───────────────────────────────▼───────────────────────────────────────┐
│                         sBitx Hardware                                │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Runtime | Node.js 22 LTS | Async I/O, TypeScript ecosystem, WebSocket/WebRTC native |
| Language | TypeScript 5.x strict | Type safety across entire stack |
| REST API | Fastify v5 | Performance, native OpenAPI, JSON Schema validation |
| WebSocket | ws + custom protocol | Full control, typed events, subscription model |
| Database | PostgreSQL 17 | Normalized schema, JSONB, mature ecosystem |
| Time-series | TimescaleDB | Radio telemetry, hypertables, retention policies |
| Cache | Redis 7 | Pub/Sub, session store, rate limiting |
| Queues | BullMQ (Redis) | Background jobs, sync queues, HAL commands |
| ORM | Drizzle ORM | Type-safe SQL, migration-managed, no codegen overhead |
| WebRTC | mediasoup v3 | SFU, Node.js native, excellent TypeScript support |
| TURN/STUN | coturn | NAT traversal for WebRTC |
| Auth | JWT (RS256) + RBAC | Stateless, asymmetric signing, role-based access |
| Logging | Pino | Structured JSON, minimal overhead |
| Tracing | OpenTelemetry | Standards-based distributed tracing |
| Metrics | Prometheus + Grafana | Platform observability |
| Testing | Vitest + Supertest | Fast unit/integration tests |
| CI/CD | GitHub Actions | Automated test, lint, build, deploy pipeline |
| Containers | Docker + Compose | Reproducible environments |

---

## Documentation Index

### Architecture
- [System Architecture Overview](docs/architecture/overview.md)
- [Database Schema Design](docs/architecture/database-schema.md)
- [WebSocket Protocol Specification](docs/architecture/websocket-protocol.md)
- [Hardware Abstraction Layer (HAL)](docs/architecture/hal.md)
- [Messaging Domain Model](docs/architecture/messaging-domain.md)
- [WebRTC Architecture](docs/architecture/webrtc.md)
- [Event-Driven Backend](docs/architecture/event-driven.md)
- [Security Architecture](docs/architecture/security.md)
- [Observability & Reliability](docs/architecture/observability.md)
- [Offline-First & Synchronization](docs/architecture/offline-first.md)

### Architecture Decision Records (ADRs)
- [ADR-001: Backend Framework Selection](docs/adr/ADR-001-backend-framework.md)
- [ADR-002: Database Strategy](docs/adr/ADR-002-database-strategy.md)
- [ADR-003: Event Broker Selection](docs/adr/ADR-003-event-broker.md)
- [ADR-004: WebSocket Protocol Design](docs/adr/ADR-004-websocket-protocol.md)
- [ADR-005: Messaging Domain Model](docs/adr/ADR-005-messaging-model.md)
- [ADR-006: Hardware Abstraction Layer Design](docs/adr/ADR-006-hal-design.md)
- [ADR-007: WebRTC Stack Selection](docs/adr/ADR-007-webrtc-stack.md)

### Implementation Roadmap
- [Phase 1 — Foundation](docs/roadmap/phase-1-foundation.md)
- [Phase 2 — Realtime Infrastructure](docs/roadmap/phase-2-realtime.md)
- [Phase 3 — Messaging & Synchronization](docs/roadmap/phase-3-messaging.md)
- [Phase 4 — Federation & Distribution](docs/roadmap/phase-4-federation.md)

### Governance
- [Contributing Guide](docs/governance/CONTRIBUTING.md)
- [ADR Template](docs/governance/ADR-TEMPLATE.md)
- [RFC Template](docs/governance/RFC-TEMPLATE.md)

### Audits
- [2026-05-20 — Adversarial Architecture Audit](docs/audit/2026-05-20-architecture-audit.md)
- [2026-06-11 — Project Review (readiness, sequencing, open gaps)](docs/audit/2026-06-11-project-review.md)

---

## Project Structure

```
poc-hermes-server/
├── src/
│   ├── api/                    # Fastify HTTP routes, controllers, middleware
│   │   ├── v1/
│   │   │   ├── auth/
│   │   │   ├── radio/
│   │   │   ├── conversations/
│   │   │   ├── messages/
│   │   │   ├── users/
│   │   │   ├── system/
│   │   │   ├── geolocation/
│   │   │   ├── frequencies/
│   │   │   └── attachments/
│   │   └── openapi/            # OpenAPI/Swagger spec generation
│   ├── gateway/                # WebSocket gateway
│   │   ├── handlers/           # Event handlers per domain
│   │   ├── middleware/         # Auth, rate limit, logging
│   │   └── protocol/           # Message framing, routing
│   ├── hal/                    # Hardware Abstraction Layer
│   │   ├── drivers/
│   │   │   ├── sbitx/          # sBitx CLI driver (production)
│   │   │   └── simulated/      # Simulated driver (dev/test)
│   │   ├── commands/           # Typed command definitions
│   │   └── telemetry/          # Telemetry stream normalization
│   ├── messaging/              # Messaging domain
│   │   ├── conversations/
│   │   ├── messages/
│   │   ├── delivery/
│   │   └── transport/          # Email/radio transport adapters
│   ├── webrtc/                 # WebRTC / mediasoup
│   │   ├── signaling/
│   │   └── sessions/
│   ├── events/                 # Internal event bus (Redis Pub/Sub)
│   ├── jobs/                   # Background workers (BullMQ)
│   ├── db/                     # Database layer (Drizzle ORM)
│   │   ├── schema/
│   │   ├── migrations/
│   │   └── repositories/
│   ├── cache/                  # Redis client and cache utilities
│   ├── auth/                   # JWT, RBAC, session management
│   └── shared/                 # Shared types, errors, utilities
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
│   ├── architecture/
│   ├── adr/
│   ├── roadmap/
│   ├── governance/
│   └── audit/
├── infrastructure/
│   ├── docker/
│   ├── scripts/
│   └── observability/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   └── workflows/
├── package.json
├── tsconfig.json
├── drizzle.config.ts
├── docker-compose.yml
└── .env.example
```

---

## Quick Start (Development)

```bash
# Clone and install
git clone https://github.com/your-org/poc-hermes-server
cd poc-hermes-server
npm install

# Configure environment
cp .env.example .env
# Edit .env with your settings

# Start infrastructure (PostgreSQL, Redis, TimescaleDB)
docker compose up -d postgres redis

# Run database migrations
npm run db:migrate

# Start development server
npm run dev
```

---

## Contributing

See [CONTRIBUTING.md](docs/governance/CONTRIBUTING.md) for development workflow, commit conventions, and PR guidelines.

---

## License

GNU General Public License v3.0 — see [LICENSE](LICENSE)

---

> HERMES is a project by [Rhizomatica](https://www.rhizomatica.org/hermes/) enabling communities in remote or disaster-affected areas to communicate when infrastructure fails.
