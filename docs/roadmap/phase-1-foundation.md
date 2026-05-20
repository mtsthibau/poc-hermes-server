# Phase 1 — Foundation

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 1–6  
**Goal**: Production-ready server with all existing hermes-api functionality, proper infrastructure, auth, and the Hardware Abstraction Layer

---

## Objectives

By the end of Phase 1, the system must:

1. Replace all existing `hermes-api` PHP endpoints with TypeScript equivalents under `/api/v1/`
2. Operate with PostgreSQL + TimescaleDB + Redis instead of SQLite
3. Have a fully functional HAL with both `SBitxCLIDriver` (production) and `SimulatedRadioDriver` (dev/test)
4. Have JWT RS256 authentication and RBAC on all endpoints
5. Auto-generate OpenAPI documentation from route schemas
6. Have Docker Compose for local development
7. Have a CI pipeline running lint, type-check, and tests

Phase 1 does **not** include WebSocket, WebRTC, or the new messaging model — those are Phase 2 and 3.

---

## Week 1–2: Project Scaffold

### Tasks

- [ ] Initialize Node.js 22 + TypeScript 5 project with strict config
- [ ] Configure ESLint + Prettier + commitlint (conventional commits)
- [ ] Create Docker Compose with PostgreSQL 17 + TimescaleDB + Redis 7
- [ ] Create `src/` directory structure:
  ```
  src/
    api/           — Fastify routes + plugins
    modules/       — Domain modules (radio, messaging, geolocation, etc.)
    hal/           — Hardware Abstraction Layer
    events/        — Event bus
    jobs/          — BullMQ workers
    db/            — Drizzle schema + migrations
    shared/        — Logger, config, utilities
  ```
- [ ] Configure Drizzle ORM + `drizzle-kit` for migrations
- [ ] Set up Vitest for unit + integration tests
- [ ] Create `infrastructure/` directory with Docker, nginx configs

### Deliverables

- Runnable `npm run dev` that starts a Fastify server with health check
- `docker-compose.yml` with all infrastructure services
- `npm test` running with at least one test passing

---

## Week 2–3: Database Schema + Migrations

### Tasks

- [ ] Implement all Drizzle schema files (matching `docs/architecture/database-schema.md`)
- [ ] Create initial migration (`0001_initial_schema.sql`)
- [ ] Run `SELECT create_hypertable(...)` for `radio_telemetry` and `gps_readings`
- [ ] Configure TimescaleDB retention policies
- [ ] Seed script: create default admin user, default frequency profiles
- [ ] Repository layer: typed query functions for all entities

### Key Schema Files

```
src/db/schema/
  users.ts
  conversations.ts
  messages.ts
  attachments.ts
  radio.ts
  geolocation.ts
  sync.ts
  audit.ts
```

### Deliverables

- `npm run db:migrate` applies schema to fresh PostgreSQL
- `npm run db:seed` creates default data
- All schema types exported and usable from TypeScript

---

## Week 3: Authentication Module

### Tasks

- [ ] Generate RSA keypair for JWT RS256 signing
- [ ] Implement `POST /api/v1/auth/login` — bcrypt verify, issue JWT + refresh token
- [ ] Implement `POST /api/v1/auth/refresh` — verify hashed refresh token, issue new JWT
- [ ] Implement `POST /api/v1/auth/logout` — revoke session
- [ ] Implement `GET /api/v1/auth/me` — return current user
- [ ] Fastify `preHandler` JWT verification middleware
- [ ] RBAC `requireRole()` middleware
- [ ] Rate limiting on auth endpoints (5 req/15min per IP)
- [ ] Audit logging for all auth events

### Security Checklist

- [ ] Passwords hashed with bcrypt (cost factor 12)
- [ ] Refresh tokens stored as SHA-256 hash (not plaintext)
- [ ] JWT RS256 (asymmetric) — no HS256
- [ ] Access token expiry: 15 minutes
- [ ] Refresh token expiry: 30 days

---

## Week 4: Hardware Abstraction Layer

### Tasks

- [ ] Define `IRadioDriver` interface (all methods)
- [ ] Implement `SBitxCLIDriver` using `execFile` with array args
- [ ] Implement `SimulatedRadioDriver` with configurable state
- [ ] Implement `RadioCommandQueue` (BullMQ, concurrency=1, priorities)
- [ ] Implement `TelemetryPoller` (configurable interval, change detection)
- [ ] Integration test: `SimulatedRadioDriver` passes all driver contract tests
- [ ] Driver selection via `RADIO_DRIVER` env var (`sbitx` | `simulated`)

### Security Checklist

- [ ] All CLI args passed as array (never string concatenation)
- [ ] Numeric args validated as integer/float before CLI call
- [ ] Mode/profile string args validated against allowlist
- [ ] CLI binary path from env config only

---

## Week 4–5: REST API — All hermes-api Endpoints

### Endpoint Groups

#### User Management (`/api/v1/user`)
- [ ] `GET /` — list users
- [ ] `POST /` — create user
- [ ] `GET /:id` — get user
- [ ] `PATCH /:id` — update user
- [ ] `DELETE /:id` — delete user (admin only)

#### Radio (`/api/v1/radio`)
- [ ] `GET /status` — current radio status (via HAL)
- [ ] `POST /profiles/:id/frequency` — set frequency
- [ ] `POST /profiles/:id/mode` — set mode
- [ ] `POST /profiles/:id/bfo` — set BFO
- [ ] `POST /volume` — set volume
- [ ] `POST /ptt` — toggle PTT (operator only)
- [ ] `POST /tone` — send tone
- [ ] `POST /protection/reset` — reset SWR protection
- [ ] `GET /profiles` — list radio profiles
- [ ] `POST /profiles` — create radio profile
- [ ] `GET /snr` — current SNR
- [ ] `GET /bitrate` — current bitrate
- [ ] `POST /power` — set power level
- [ ] `POST /step` — set frequency step

#### File (`/api/v1/file`)
- [ ] `POST /upload` — upload file (MIME validation, UUID filename)
- [ ] `GET /:id` — download file (signed URL check)

#### System (`/api/v1/sys`)
- [ ] `GET /logs` — view system logs (tail)
- [ ] `GET /stats` — system statistics
- [ ] `POST /uucp` — trigger UUCP connection
- [ ] `POST /shutdown` — shutdown (admin only)
- [ ] `POST /reboot` — reboot (admin only)

#### Geolocation (`/api/v1/geolocation`)
- [ ] `GET /current` — current GPS position
- [ ] `POST /calibrate` — manual calibration
- [ ] `POST /sos` — trigger SOS alert

#### Frequency (`/api/v1/frequency`)
- [ ] `GET /` — list saved frequencies
- [ ] `POST /` — add frequency
- [ ] `DELETE /:id` — remove frequency

#### Caller / Schedules (`/api/v1/caller`)
- [ ] `GET /` — list schedules
- [ ] `POST /` — create schedule
- [ ] `DELETE /:id` — delete schedule

#### WiFi (`/api/v1/wifi`)
- [ ] `GET /` — current WiFi config
- [ ] `POST /` — update WiFi config

#### Customer Errors (`/api/v1/customerrors`)
- [ ] `POST /` — log client-side error
- [ ] `GET /` — retrieve error logs (admin)

#### Message (compatibility shim) (`/api/v1/message`)
- [ ] `GET /inbox` — list inbox (shim → conversation queries)
- [ ] `GET /outbox` — list outbox (shim → conversation queries)
- [ ] `POST /send` — send message (shim → messaging service)

### Validation Checklist

- [ ] Every route has JSON Schema with `additionalProperties: false`
- [ ] All routes require authentication except `/api/v1/health`
- [ ] Route schemas generate OpenAPI spec via `@fastify/swagger`

---

## Week 5–6: Testing + CI

### Tests Required

- [ ] Auth: login, refresh, logout, invalid credentials, expired token
- [ ] RBAC: all role combinations for protected endpoints
- [ ] HAL: all `SimulatedRadioDriver` operations (contract tests)
- [ ] Command queue: serial execution, priority ordering, retry on failure
- [ ] Radio endpoints: full stack with simulated driver
- [ ] File upload: MIME validation, UUID naming, size limit
- [ ] Health checks: all three endpoints

### CI Pipeline (`github/workflows/ci.yml`)

- [ ] `npm run lint` — ESLint
- [ ] `npm run typecheck` — tsc --noEmit
- [ ] `npm run test` — Vitest (integration + unit)
- [ ] `npm audit` — security scan
- [ ] Build Docker image

---

## Definition of Done — Phase 1

- [ ] All existing hermes-api endpoints functional under `/api/v1/`
- [ ] `RADIO_DRIVER=simulated` passes all endpoint tests in CI
- [ ] `RADIO_DRIVER=sbitx` tested manually on station hardware
- [ ] OpenAPI docs available at `/api/docs`
- [ ] Prometheus metrics at `/metrics`
- [ ] Health check at `/health/ready` returns 200
- [ ] Zero `npm audit` high/critical vulnerabilities
- [ ] All TypeScript strict-mode errors resolved
- [ ] CONTRIBUTING.md followed for all PRs (conventional commits, PR template)
