# Phase 1 — Foundation Task List

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 1–6  
**Prerequisites**: None  
**Generated**: 2026-06-10  
**Master Plan Reference**: [00-master-project-plan.md](../plan/00-master-project-plan.md)

---

## Phase Objectives (from roadmap)

1. Replace all existing `hermes-api` PHP endpoints with TypeScript equivalents under `/api/v1/`
2. Operate with PostgreSQL 17 + TimescaleDB + Redis 7 instead of SQLite
3. Have a fully functional HAL with both `SBitxCLIDriver` (production) and `SimulatedRadioDriver` (dev/test)
4. Have JWT RS256 authentication and RBAC on all endpoints
5. Auto-generate OpenAPI documentation from route schemas
6. Have Docker Compose for local development
7. Have a CI pipeline running lint, type-check, and tests

Phase 1 does **not** include WebSocket, WebRTC, or the new messaging model.

## Audit Findings Addressed This Phase

- **[AUDIT H-1]** Remove `conversations.last_message_id` denormalized field; compute on-read via index
- **[AUDIT H-3]** HAL circuit-breaker fallback for Redis unavailability (PTT safety)
- **[AUDIT H-4]** Mandatory refresh token rotation — "Optionally" removed from flow
- **[AUDIT H-5]** `IStorageBackend` abstraction defined from day one; `LocalFileStorageBackend` implemented
- **[AUDIT M-3]** TimescaleDB background worker configuration documented and applied
- **[AUDIT M-7]** Pino redact paths cover deeply nested token/password fields
- **[AUDIT L-4]** Audit log action strings defined as TypeScript `as const` record
- **[AUDIT TD-03]** RFC template updated with decision authority

---

## Week 1–2: Project Scaffold & Infrastructure (EP-01)

### [ ] Task 1.1.1: Initialize Node.js 22 + TypeScript 5 project
**Purpose**: Establish the baseline project with strict TypeScript configuration — the foundation every other task builds on.  
**Description**: Run `npm init`, install `typescript@5`, configure `tsconfig.json` with `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`. Set `rootDir: "src"`, `outDir: "dist"`. Add `npm run build`, `npm run dev` (tsx watch), `npm run type-check` scripts.  
**Acceptance Criteria**:
- [ ] `npm run build` produces zero TypeScript errors on an empty `src/index.ts`
- [ ] `tsc --noEmit` passes with strict mode in CI
- [ ] `npm run dev` starts and logs a startup message

**Files to Create/Edit**:
- `package.json` — project metadata, scripts
- `tsconfig.json` — strict TypeScript config
- `src/index.ts` — minimal entry point

**Stack Notes**: Use `tsx` for development watch mode; `tsc` for production build  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 1–2`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None at this step  
**Blocks**: All subsequent tasks

---

### [ ] Task 1.1.2: Configure ESLint + Prettier + commitlint
**Purpose**: Enforce code style and conventional commit discipline to keep the codebase consistent across the team.  
**Description**: Install `eslint`, `@typescript-eslint/eslint-plugin`, `prettier`, `eslint-config-prettier`, `@commitlint/cli`, `@commitlint/config-conventional`, `husky`. Configure `.eslintrc.cjs`, `.prettierrc`, `commitlint.config.cjs`. Add husky pre-commit (lint + format check) and commit-msg (commitlint) hooks.  
**Acceptance Criteria**:
- [ ] `npm run lint` exits 0 on clean code and non-zero on lint errors
- [ ] `npm run format:check` exits non-zero on unformatted code
- [ ] Commit with non-conventional message is rejected by commitlint hook

**Files to Create/Edit**:
- `.eslintrc.cjs`
- `.prettierrc`
- `commitlint.config.cjs`
- `.husky/pre-commit`
- `.husky/commit-msg`

**Stack Notes**: Use `@typescript-eslint/parser`; disable `eslint-plugin-prettier` (use format:check separately)  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 1–2`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: None (parallel with 1.1.1)

---

### [ ] Task 1.1.3: Create Docker Compose with PostgreSQL 17 + TimescaleDB + Redis 7
**Purpose**: Provide a reproducible local development environment matching production dependencies.  
**Description**: Create `docker-compose.yml` with services: `postgres` (image: `timescale/timescaledb:latest-pg17`), `redis` (image: `redis:7-alpine`). Configure health checks for both services. Expose PG on `5432`, Redis on `6379`. Add `.env.example` with all required variables. Configure `POSTGRES_DB=hermes`, `POSTGRES_USER=hermes`, `POSTGRES_PASSWORD` from env. **[AUDIT M-3]**: Set `TIMESCALEDB_TELEMETRY=off` and document `timescaledb.max_background_workers=2` in `infrastructure/docker/postgresql.conf`.  
**Acceptance Criteria**:
- [ ] `docker compose up -d` starts both services with health checks passing
- [ ] `psql -h localhost -U hermes hermes` connects successfully
- [ ] `redis-cli ping` returns PONG
- [ ] TimescaleDB extension is enabled: `SELECT installed_version FROM pg_available_extensions WHERE name = 'timescaledb';` returns a version

**Files to Create/Edit**:
- `docker-compose.yml`
- `.env.example`
- `infrastructure/docker/postgresql.conf`

**Stack Notes**: Do not use `postgres:17-alpine` — must use TimescaleDB image  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 1–2`, `docs/architecture/database-schema.md §1`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: Tasks 1.2.x (DB schema)

---

### [ ] Task 1.1.4: Create source directory structure and Fastify application skeleton
**Purpose**: Establish the module directory layout and a runnable minimal server with a health check.  
**Description**: Create the full `src/` directory tree. Initialize Fastify v5 app in `src/api/app.ts`. Register `@fastify/sensible`. Add `GET /health/live` and `GET /health/ready` routes (ready checks DB + Redis connectivity). Wire startup and graceful shutdown in `src/index.ts`.  
**Acceptance Criteria**:
- [ ] `npm run dev` starts a Fastify server on configured port
- [ ] `GET /health/live` returns `200 { status: "ok" }`
- [ ] `GET /health/ready` returns `200` when PG and Redis are reachable, `503` otherwise
- [ ] Graceful shutdown handler closes DB pool and Redis on `SIGTERM`/`SIGINT`

**Files to Create/Edit**:
- `src/api/app.ts`
- `src/api/routes/health.ts`
- `src/index.ts`
- `src/shared/config.ts` (env var accessor — all env access goes here)
- `src/shared/logger.ts` (Pino instance — see Task 1.1.5)

**Stack Notes**: Fastify v5 uses `fastify()` factory; register plugins before routes; `app.ready()` before server listen  
**Doc Reference**: `docs/architecture/overview.md §4.1`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: All route tasks (1.5.x)

---

### [ ] Task 1.1.5: Configure Pino logger with security redaction
**Purpose**: Establish structured logging that prevents credential leakage in log output.  
**Description**: Create `src/shared/logger.ts`. Configure Pino with: `level` from `LOG_LEVEL` env, `base` containing service/version/environment, redact paths covering deep nesting **[AUDIT M-7]**: `['password', '*.password', '**.password', 'token', '*.token', '**.token', 'refreshToken', '*.refreshToken', 'authorization', '**.authorization', 'payload.token', 'body.password', 'body.token']`. Add `pino-pretty` for development. Register Fastify logger plugin using the shared logger.  
**Acceptance Criteria**:
- [ ] Log entries are JSON in production mode
- [ ] Automated test asserts that `logger.info({ body: { password: 'secret' } })` outputs `[REDACTED]`
- [ ] Automated test asserts that `logger.info({ payload: { token: 'eyJ...' } })` outputs `[REDACTED]`
- [ ] Request logs include `requestId` and `userId` fields

**Files to Create/Edit**:
- `src/shared/logger.ts`
- `src/shared/logger.test.ts`

**Stack Notes**: Use `pino.child()` for per-request loggers inside Fastify hooks  
**Doc Reference**: `docs/architecture/observability.md §2`, Audit M-7  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: All other tasks (logger is shared)

---

### [ ] Task 1.1.6: Set up Vitest and CI pipeline
**Purpose**: Enable automated test execution and enforce quality gates on every pull request.  
**Description**: Install `vitest`, `@vitest/coverage-v8`, `supertest`. Configure `vitest.config.ts` with coverage threshold 80%. Create GitHub Actions workflow `.github/workflows/ci.yml` with jobs: `lint`, `type-check`, `test`, `build`. CI must start Docker Compose services (PG + Redis) for integration tests. Add `npm run db:migrate` step in CI before tests.  
**Acceptance Criteria**:
- [ ] `npm test` runs all `*.test.ts` files
- [ ] `npm run test:coverage` fails if any module < 80% coverage
- [ ] CI workflow passes on a clean repository
- [ ] CI fails on lint errors, type errors, or test failures

**Files to Create/Edit**:
- `vitest.config.ts`
- `.github/workflows/ci.yml`

**Stack Notes**: Use `vitest`'s built-in mocking — no jest-mock imports  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 1–2`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: None (can run in parallel with other scaffold tasks)

---

## Week 2–3: Database Schema & Repository Layer (EP-02)

### [ ] Task 1.2.1: Implement Drizzle ORM configuration and migration tooling
**Purpose**: Establish the ORM connection and migration workflow that all schema tasks depend on.  
**Description**: Install `drizzle-orm`, `drizzle-kit`, `pg`. Create `src/db/index.ts` (drizzle connection pool, exported as `db`). Create `src/db/migrate.ts` (runs migrations via `drizzle-kit migrate`). Add `npm run db:migrate` and `npm run db:generate` scripts. Configure `drizzle.config.ts`.  
**Acceptance Criteria**:
- [ ] `npm run db:migrate` applies migrations to a fresh PostgreSQL instance
- [ ] `db.execute(sql`SELECT 1`)` succeeds from application code
- [ ] Connection pool is closed on graceful shutdown

**Files to Create/Edit**:
- `src/db/index.ts`
- `src/db/migrate.ts`
- `drizzle.config.ts`

**Stack Notes**: Use `drizzle-orm/pg-core`; connection pool via `pg.Pool`; env vars from `src/shared/config.ts` only  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 2–3`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: 1.2.2–1.2.8

---

### [ ] Task 1.2.2: Implement Drizzle schema — Identity & Authentication tables
**Purpose**: Define the users, sessions, and devices schema that backs all authentication and RBAC.  
**Description**: Implement `src/db/schema/users.ts` with tables `users`, `user_sessions`, `user_devices` matching `docs/architecture/database-schema.md §3.1`. Use UUIDs, `TIMESTAMPTZ`, JSONB, appropriate `CHECK` constraints, and named indexes. **[AUDIT H-1]**: Do NOT include `last_message_id` as a column on `conversations` — it will be computed on-read.  
**Acceptance Criteria**:
- [ ] `npm run db:generate` produces a valid migration SQL
- [ ] `npm run db:migrate` applies the migration without error
- [ ] TypeScript types are exported and usable (`User`, `UserSession`, `UserDevice`)
- [ ] `users.callsign` has a `UNIQUE` constraint

**Files to Create/Edit**:
- `src/db/schema/users.ts`
- `src/db/migrations/0001_identity.sql` (generated)

**Stack Notes**: Use `pgTable`, `uuid`, `text`, `timestamptz`, `jsonb` from drizzle-orm/pg-core  
**Doc Reference**: `docs/architecture/database-schema.md §3.1`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: This is the authoritative user store; callsign must match hermes-api user identifiers  
**Blocks**: 1.3.x (Auth module)

---

### [ ] Task 1.2.3: Implement Drizzle schema — Conversations & Messaging tables
**Purpose**: Define the chat-native messaging schema that will replace the email inbox/outbox model.  
**Description**: Implement `src/db/schema/conversations.ts` with tables: `conversations` (without `last_message_id` column — **[AUDIT H-1]**), `conversation_participants`, `messages`, `message_deliveries`, `message_reactions`, `attachments`, `message_envelopes`. Add `idx_messages_conversation_created` composite index on `(conversation_id, created_at DESC)` to support on-read `last_message_id` computation. **[AUDIT C-2]**: `sync_queue` stores `entity_id UUID`, `operation TEXT`, `entity_type TEXT` — no `payload` JSONB column.  
**Acceptance Criteria**:
- [ ] Migration applies cleanly
- [ ] `messages` table has `client_message_id` with `UNIQUE` constraint for idempotency
- [ ] `sync_queue` does NOT have a `payload` column — only `entity_id`, `operation`, `entity_type`, `sequence`, `target_user_id`, `target_device_id`
- [ ] `conversation_participants` has unique constraint on `(conversation_id, user_id)`

**Files to Create/Edit**:
- `src/db/schema/conversations.ts`
- `src/db/migrations/0002_messaging.sql` (generated)

**Doc Reference**: `docs/architecture/database-schema.md §3.2`, Audit C-2, Audit H-1  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Legacy message shim must map from new schema to old inbox/outbox format  
**Blocks**: 1.5.x (messaging API routes), EP-12 (Phase 3)

---

### [ ] Task 1.2.4: Implement Drizzle schema — Radio, Geolocation, Frequency, Schedule tables
**Purpose**: Persist radio operational data and configuration.  
**Description**: Implement `src/db/schema/radio.ts` and `src/db/schema/geolocation.ts`: `radio_profiles`, `radio_sessions`, `radio_telemetry`, `frequencies`, `connection_schedules`, `gps_readings`. Configure TimescaleDB hypertables for `radio_telemetry` (`time_column: created_at`) and `gps_readings`. Apply retention policy: 90 days hot, compress after 7 days. **[AUDIT M-3]**: Add `timescaledb.max_background_workers = 2` to PostgreSQL config.  
**Acceptance Criteria**:
- [ ] `SELECT create_hypertable('radio_telemetry', 'created_at')` executes without error after migration
- [ ] `SELECT create_hypertable('gps_readings', 'created_at')` executes without error
- [ ] Retention policy job created for 90-day window
- [ ] Migration applies to fresh DB in CI

**Files to Create/Edit**:
- `src/db/schema/radio.ts`
- `src/db/schema/geolocation.ts`
- `src/db/migrations/0003_radio_geo.sql` (generated)
- `infrastructure/docker/postgresql.conf` (add `timescaledb.max_background_workers=2`)

**Doc Reference**: `docs/architecture/database-schema.md §3.4`, Audit M-3  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Radio profiles must map to existing frequency/mode profile IDs used by poc-hermes-app  
**Blocks**: 1.4.x (HAL), 1.5.x (radio API)

---

### [ ] Task 1.2.5: Implement Drizzle schema — Audit & Sync tables
**Purpose**: Persist audit trail and device sync state.  
**Description**: Implement `src/db/schema/audit.ts` with `audit_logs` and `src/db/schema/sync.ts` with `sync_cursors`, `sync_queue`. **[AUDIT L-4]**: Define `AuditAction` as `as const` record in `src/shared/audit-actions.ts` with all known action strings (`auth.login`, `auth.refresh`, `radio.ptt_activated`, etc.).  
**Acceptance Criteria**:
- [ ] `audit_logs` has `actor_id`, `action`, `entity_type`, `entity_id`, `metadata`, `created_at`
- [ ] `AuditAction` type is imported across auth, radio, messaging modules — no loose string literals
- [ ] TypeScript errors if an undefined action string is used

**Files to Create/Edit**:
- `src/db/schema/audit.ts`
- `src/db/schema/sync.ts`
- `src/shared/audit-actions.ts`
- `src/db/migrations/0004_audit_sync.sql` (generated)

**Doc Reference**: `docs/architecture/security.md §9`, Audit L-4  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None directly  
**Blocks**: 1.3.x (auth audit logging), 1.5.x (radio audit logging)

---

### [ ] Task 1.2.6: Implement repository layer — UserRepository and SessionRepository
**Purpose**: Encapsulate all user/session database queries; no raw SQL in route handlers.  
**Description**: Create `src/modules/auth/repositories/UserRepository.ts` and `SessionRepository.ts`. Implement: `UserRepository.findByCallsign()`, `findById()`, `create()`, `updateStatus()`. `SessionRepository.create()`, `findByTokenHash()`, `revoke()`, `revokeAllForUser()`, `deleteExpired()`.  
**Acceptance Criteria**:
- [ ] All methods have JSDoc documenting their contract
- [ ] `findByCallsign` uses constant-time comparison path (no early return that leaks existence)
- [ ] `findByTokenHash` accepts SHA-256 hash, not plaintext
- [ ] Unit tests mock `db` and verify all query branches

**Files to Create/Edit**:
- `src/modules/auth/repositories/UserRepository.ts`
- `src/modules/auth/repositories/SessionRepository.ts`
- `src/modules/auth/repositories/UserRepository.test.ts`
- `src/modules/auth/repositories/SessionRepository.test.ts`

**Stack Notes**: Use Drizzle `db.select().from().where()` — no raw SQL  
**Doc Reference**: `docs/architecture/database-schema.md §3.1`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Callsign field must match poc-hermes-app user login flow  
**Blocks**: 1.3.x (Auth module)

---

### [ ] Task 1.2.7: Implement repository layer — RadioRepository and GeoRepository
**Purpose**: Typed access to radio profiles, sessions, and telemetry.  
**Description**: Create `src/modules/radio/repositories/RadioRepository.ts` with: `getStatus()`, `getActiveSession()`, `getProfiles()`, `createProfile()`, `updateProfile()`, `getLatestTelemetry()`. Create `src/modules/geolocation/repositories/GeoRepository.ts` with: `getCurrentPosition()`, `insertReading()`.  
**Acceptance Criteria**:
- [ ] All methods return typed objects (no `any`)
- [ ] `getStatus()` returns `RadioStatus` type exactly matching `IRadioDriver` output shape
- [ ] Unit tests for both repositories

**Files to Create/Edit**:
- `src/modules/radio/repositories/RadioRepository.ts`
- `src/modules/geolocation/repositories/GeoRepository.ts`

**Doc Reference**: `docs/architecture/hal.md §3`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Radio status response shape must be wire-compatible with poc-hermes-app  
**Blocks**: 1.4.x (HAL), 1.5.x (radio routes)

---

### [ ] Task 1.2.8: Database seed script
**Purpose**: Provide a reproducible starting state for development and CI.  
**Description**: Create `src/db/seed.ts`. Seed: one admin user (callsign from env, password hashed with bcrypt cost 12), two default frequency profiles, one test `readonly` user. Seed script is idempotent (upsert, not insert).  
**Acceptance Criteria**:
- [ ] `npm run db:seed` runs without error on a fresh or already-seeded DB
- [ ] Admin user can log in via `POST /api/v1/auth/login` after seed
- [ ] No plaintext passwords in seed script or logs

**Files to Create/Edit**:
- `src/db/seed.ts`

**Stack Notes**: bcrypt cost factor must be 12 minimum  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 2–3`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: Admin callsign must match expected poc-hermes-app default credentials  
**Blocks**: None

---

## Week 3: Authentication Module (EP-03)

### [ ] Task 1.3.1: Generate RSA keypair and JWT infrastructure
**Purpose**: Establish the RS256 signing infrastructure.  
**Description**: Generate 4096-bit RSA keypair via `openssl` (documented in README). Store private key path in `JWT_PRIVATE_KEY_PATH` env var; public key in `JWT_PUBLIC_KEY_PATH`. Create `src/modules/auth/jwt.ts` with `signAccessToken(payload): string` and `verifyAccessToken(token): AccessTokenPayload`. Access token expiry: 15 minutes. Refresh token: `crypto.randomBytes(48).toString('hex')`, stored as SHA-256 hash.  
**Acceptance Criteria**:
- [ ] Generated JWT is signed with RS256 (not HS256) — verified with `jose` or `jsonwebtoken` decode
- [ ] `verifyAccessToken` throws on HS256-signed tokens
- [ ] `verifyAccessToken` throws on expired tokens
- [ ] Unit tests cover sign → verify roundtrip, expiry, wrong algorithm

**Files to Create/Edit**:
- `src/modules/auth/jwt.ts`
- `src/modules/auth/jwt.test.ts`

**Stack Notes**: Use `jsonwebtoken` with explicit `algorithms: ['RS256']` in verify options  
**Doc Reference**: `docs/architecture/security.md §2.1`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Token payload claims (`sub`, `callsign`, `role`, `sessionId`) must match what poc-hermes-app expects to parse  
**Blocks**: 1.3.2–1.3.5

---

### [ ] Task 1.3.2: Implement authentication endpoints
**Purpose**: Provide login, refresh, logout, and me endpoints for all clients.  
**Description**: Implement `src/modules/auth/routes.ts`. `POST /api/v1/auth/login`: bcrypt verify (cost≥12), issue tokens, create `user_sessions` row, write audit log. `POST /api/v1/auth/refresh`: lookup `SHA-256(token)` in `user_sessions`, verify not revoked/expired, **[AUDIT H-4]** MANDATORY rotation — issue new refresh token, revoke old. Detect reuse of revoked token, increment `auth_refresh_token_reuse_detected_total` metric. `POST /api/v1/auth/logout`: revoke session. `GET /api/v1/auth/me`: return current user from JWT.  
**Acceptance Criteria**:
- [ ] `POST /auth/login` returns `{ accessToken, refreshToken, expiresIn: 900 }`
- [ ] `POST /auth/refresh` with used refresh token returns `403` and increments reuse metric
- [ ] `POST /auth/refresh` issues new refresh token (old is revoked in DB)
- [ ] `POST /auth/logout` revokes the session
- [ ] All auth events appear in `audit_logs`
- [ ] Integration test covers full login → refresh → logout cycle

**Files to Create/Edit**:
- `src/modules/auth/routes.ts`
- `src/modules/auth/routes.test.ts` (integration)

**Stack Notes**: Use `@fastify/rate-limit` for auth endpoints (5 req/15 min/IP)  
**Doc Reference**: `docs/architecture/security.md §2.2`, `§2.3`, Audit H-4  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Login response shape must be wire-compatible; callsign field (not username/email) is the login identifier  
**Blocks**: 1.3.3 (middleware), all route tasks

---

### [ ] Task 1.3.3: Implement JWT verification middleware and RBAC
**Purpose**: Enforce authentication and authorization on every protected route.  
**Description**: Create `src/api/middleware/authenticate.ts` (Fastify preHandler): verify JWT, set `request.user`. Create `src/api/middleware/rbac.ts`: `requireRole(...roles: UserRole[])` — returns `403` with audit log entry on unauthorized access. **[AUDIT C-4]**: Add `src/api/middleware/sessionRevalidation.ts` — every N minutes per connection, check `user_sessions` for `revoked_at` and `expires_at`; if invalid, return `401`. (Full WebSocket re-validation is Phase 2; REST re-validation is here.)  
**Acceptance Criteria**:
- [ ] Requests without JWT return `401 UNAUTHENTICATED`
- [ ] Requests with wrong role return `403 FORBIDDEN` and write audit log
- [ ] Requests with revoked session return `401`
- [ ] Unit tests for both middleware functions covering all branches

**Files to Create/Edit**:
- `src/api/middleware/authenticate.ts`
- `src/api/middleware/rbac.ts`
- `src/api/middleware/authenticate.test.ts`

**Doc Reference**: `docs/architecture/security.md §3`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Bearer token format must be `Authorization: Bearer <jwt>` — no API key header variants  
**Blocks**: All authenticated route tasks

---

## Week 4: Hardware Abstraction Layer (EP-04)

### [ ] Task 1.4.1: Define IRadioDriver interface and HAL types
**Purpose**: Establish the central contract that all radio-hardware interactions must implement.  
**Description**: Create `src/hal/types.ts` with all types from `docs/architecture/hal.md §3`: `IRadioDriver`, `RadioStatus`, `RadioProfile`, `RadioMode`, `RadioCommandResult`, `RadioCommandError`, `RadioErrorCode`. Add JSDoc to every interface member.  
**Acceptance Criteria**:
- [ ] `IRadioDriver` interface compiles under strict TypeScript with no `any`
- [ ] All types exported from `src/hal/index.ts`
- [ ] No service file imports from `src/hal/drivers/` directly — only from `src/hal/` (enforced via ESLint `no-restricted-imports`)

**Files to Create/Edit**:
- `src/hal/types.ts`
- `src/hal/index.ts`

**Doc Reference**: `docs/architecture/hal.md §3`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None directly (internal abstraction)  
**Blocks**: 1.4.2, 1.4.3, 1.4.4

---

### [ ] Task 1.4.2: Implement SimulatedRadioDriver
**Purpose**: Enable development and CI testing without physical radio hardware.  
**Description**: Create `src/hal/drivers/SimulatedRadioDriver.ts`. Implements `IRadioDriver` with in-memory state machine. Configurable initial state via constructor options. Deterministic responses. Emits events on state changes. Supports injecting error scenarios (`simulateError(errorCode)`) for testing.  
**Acceptance Criteria**:
- [ ] `SimulatedRadioDriver` passes the full IRadioDriver contract test suite (see Task 1.4.5)
- [ ] `setFrequency(invalidHz)` returns `RadioCommandError { code: 'INVALID_PARAMETER' }`
- [ ] `simulateError('HARDWARE_UNAVAILABLE')` causes next command to fail with correct error

**Files to Create/Edit**:
- `src/hal/drivers/SimulatedRadioDriver.ts`

**Doc Reference**: `docs/architecture/hal.md §2`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: None directly — test/CI usage  
**Blocks**: 1.4.5 (contract tests), CI integration

---

### [ ] Task 1.4.3: Implement SBitxCLIDriver
**Purpose**: Wrap the sBitx CLI binary for production radio control.  
**Description**: Create `src/hal/drivers/SBitxCLIDriver.ts`. All CLI calls via `child_process.execFile` with array args (never string concatenation). Binary path from env config only. Numeric args validated as integer/float before call. Mode/profile strings validated against allowlist. Per-command timeout (5000 ms default, configurable). Parses CLI stdout/stderr into `RadioCommandResult<T>`.  
**Acceptance Criteria**:
- [ ] `execFile` is always called with array args — ESLint rule enforces this
- [ ] Passing a non-integer to `setFrequency` returns `INVALID_PARAMETER` before any CLI call
- [ ] Mode string not in `['USB', 'LSB', 'CW', 'AM', 'FM', 'DIGITAL']` returns `INVALID_PARAMETER`
- [ ] CLI timeout produces `COMMAND_TIMEOUT` error code

**Files to Create/Edit**:
- `src/hal/drivers/SBitxCLIDriver.ts`

**Stack Notes**: `child_process.execFile(binaryPath, ['--freq', String(hz)], { timeout: 5000 })`  
**Doc Reference**: `docs/architecture/hal.md §3`, `docs/architecture/security.md §6`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Frequency values and mode strings must produce same hardware effect as existing hermes-api PHP CLI wrappers  
**Blocks**: Production deployment

---

### [ ] Task 1.4.4: Implement RadioCommandQueue and TelemetryPoller
**Purpose**: Serialize hardware commands and stream telemetry to the event bus.  
**Description**: Create `src/hal/RadioCommandQueue.ts` using BullMQ `Queue` + `Worker` with `concurrency: 1`. Priority levels: `PTT=10`, `standard=5`, `background=1`. Per-job timeout. **[AUDIT H-3]**: Implement circuit-breaker fallback — when Redis connection fails, fall back to in-process `AsyncMutex` for serialization; log `warn` + alert via metric `radio.command_queue.degraded`. PTT and stop commands must always be executable. Create `src/hal/TelemetryPoller.ts`: configurable interval, change detection, publishes to `IEventBus` (injected).  
**Acceptance Criteria**:
- [ ] Commands execute in order (concurrency=1)
- [ ] Circuit-breaker test: disconnect Redis mock → PTT command still executes via fallback
- [ ] `TelemetryPoller` emits `radio.telemetry` event every poll interval
- [ ] `TelemetryPoller` emits `radio.state_changed` only when state delta is significant

**Files to Create/Edit**:
- `src/hal/RadioCommandQueue.ts`
- `src/hal/TelemetryPoller.ts`
- `src/hal/AsyncMutex.ts`

**Doc Reference**: `docs/architecture/hal.md §2`, Audit H-3  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: None directly; protects hardware from concurrent commands from any client  
**Blocks**: EP-06 (IEventBus must exist first — use constructor injection)

---

### [ ] Task 1.4.5: IRadioDriver contract test suite
**Purpose**: Ensure both drivers behave identically under the same inputs.  
**Description**: Create `src/hal/drivers/radioDriver.contract.test.ts`. Parameterized test suite that runs against both `SimulatedRadioDriver` and (when `RADIO_DRIVER=sbitx` is set) `SBitxCLIDriver`. Tests: `initialize()`, `getStatus()`, `setFrequency()`, `setMode()`, `setPTT()`, `shutdown()`, error scenarios.  
**Acceptance Criteria**:
- [ ] `SimulatedRadioDriver` passes 100% of contract tests in CI
- [ ] Contract tests cover both success and error paths for every `IRadioDriver` method
- [ ] Contract tests document which behaviors are hardware-only (skipped for simulated)

**Files to Create/Edit**:
- `src/hal/drivers/radioDriver.contract.test.ts`

**Doc Reference**: `docs/architecture/hal.md §3`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Ensures production driver produces same output shape as simulated — protecting poc-hermes-app consumers  
**Blocks**: None (can run in parallel with 1.4.2)

---

## Week 4–5: REST API (EP-05)

### [ ] Task 1.5.1: REST API — Radio endpoints (`/api/v1/radio`)
**Purpose**: Expose all radio control operations via HTTP for poc-hermes-app.  
**Description**: Implement all radio routes from `docs/roadmap/phase-1-foundation.md §Week 4–5`: `GET /status`, `POST /profiles/:id/frequency`, `POST /profiles/:id/mode`, `POST /profiles/:id/bfo`, `POST /volume`, `POST /ptt`, `POST /tone`, `POST /protection/reset`, `GET /profiles`, `POST /profiles`, `GET /snr`, `GET /bitrate`, `POST /power`, `POST /step`. All routes delegate to `RadioService` → `RadioCommandQueue` → `IRadioDriver`. JSON Schema on all routes. RBAC: PTT requires `operator`|`admin`.  
**Acceptance Criteria**:
- [ ] All radio endpoints return correct HTTP status codes and response shapes
- [ ] `POST /ptt` with `user` role returns `403`
- [ ] Invalid frequency (e.g., negative number) returns `400` with validation error
- [ ] All radio commands written to `audit_logs`
- [ ] Integration test: login as operator → set frequency → GET status shows new frequency

**Files to Create/Edit**:
- `src/api/routes/radio.ts`
- `src/modules/radio/RadioService.ts`
- `src/api/routes/radio.test.ts`

**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 4–5`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: **Critical** — these are the primary endpoints used by poc-hermes-app; response shapes must be wire-compatible  
**Blocks**: Compatibility gate

---

### [ ] Task 1.5.2: REST API — User management endpoints (`/api/v1/user`)
**Purpose**: Allow admin management of user accounts.  
**Description**: `GET /`, `POST /`, `GET /:id`, `PATCH /:id`, `DELETE /:id` (admin only). All input validated against JSON Schema. `DELETE` requires `admin` role. `PATCH` cannot change `role` unless caller is `admin`.  
**Acceptance Criteria**:
- [ ] `DELETE /:id` with `operator` role returns `403`
- [ ] `PATCH /:id` with `role` field and non-admin caller returns `403`
- [ ] `POST /` with duplicate callsign returns `409 CONFLICT`
- [ ] Integration tests cover all CRUD operations

**Files to Create/Edit**:
- `src/api/routes/users.ts`
- `src/modules/users/UserService.ts`

**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 4–5`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: User list response must include `callsign` field  
**Blocks**: None

---

### [ ] Task 1.5.3: REST API — File upload/download with IStorageBackend
**Purpose**: Provide attachment storage with security controls and the backend abstraction required for Phase 4.  
**Description**: **[AUDIT H-5]**: Define `IStorageBackend` interface in `src/modules/storage/IStorageBackend.ts`. Implement `LocalFileStorageBackend`. `POST /api/v1/file/upload`: stream to temp → detect MIME from content bytes (`file-type` library) → validate against allowlist → compute SHA-256 → dedup → store with UUID filename → insert `attachments` row. `GET /api/v1/file/:id`: verify conversation membership → serve file with stored MIME `Content-Type`.  
**Acceptance Criteria**:
- [ ] Upload with a renamed `.exe` file (content-type spoofed) is rejected if MIME not in allowlist
- [ ] Stored filename is always a UUID — original filename never reaches filesystem path
- [ ] Path traversal test: `../../../etc/passwd` as filename returns `400` before any filesystem call
- [ ] Download without conversation membership returns `403`
- [ ] `IStorageBackend` is injected into upload/download handlers — not instantiated inside them

**Files to Create/Edit**:
- `src/modules/storage/IStorageBackend.ts`
- `src/modules/storage/LocalFileStorageBackend.ts`
- `src/api/routes/files.ts`

**Stack Notes**: Use `file-type` npm package for content-byte MIME detection  
**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 4–5`, Audit H-5  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: File download endpoint must return same `Content-Disposition` behavior  
**Blocks**: EP-16 (Phase 3 attachment pipeline)

---

### [ ] Task 1.5.4: REST API — System, Geolocation, Frequency, Schedule, WiFi endpoints
**Purpose**: Complete the hermes-api endpoint surface for full poc-hermes-app compatibility.  
**Description**: Implement: `GET /api/v1/sys/logs`, `GET /api/v1/sys/stats`, `POST /api/v1/sys/uucp`, `POST /api/v1/sys/shutdown` (admin), `POST /api/v1/sys/reboot` (admin). `GET /api/v1/geolocation/current`, `POST /api/v1/geolocation/calibrate`, `POST /api/v1/geolocation/sos`. `GET|POST|DELETE /api/v1/frequency`. `GET|POST|DELETE /api/v1/caller`. `GET|POST /api/v1/wifi`. `POST|GET /api/v1/customerrors`.  
**Acceptance Criteria**:
- [ ] `POST /sys/shutdown` with `operator` role returns `403`
- [ ] `POST /geolocation/sos` writes to `audit_logs`
- [ ] All endpoints have JSON Schema annotation and appear in OpenAPI docs
- [ ] Integration tests for each endpoint group

**Files to Create/Edit**:
- `src/api/routes/system.ts`
- `src/api/routes/geolocation.ts`
- `src/api/routes/frequency.ts`
- `src/api/routes/schedules.ts`
- `src/api/routes/wifi.ts`
- `src/api/routes/customerErrors.ts`

**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 4–5`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: **Critical** — all of these are used by poc-hermes-app; wire-compatibility required  
**Blocks**: Compatibility gate

---

### [ ] Task 1.5.5: REST API — Legacy messaging compatibility shim
**Purpose**: Ensure poc-hermes-app continues to work with the new server before Phase 3 messaging is complete.  
**Description**: Implement `src/api/routes/messagingShim.ts`. `GET /api/v1/message/inbox` → queries `messages` + `conversations` → maps to legacy inbox format (array of `{ from, to, subject, body, date, id }`). `GET /api/v1/message/outbox` → same for sent messages. `POST /api/v1/message/send` → creates conversation (if needed) + message via `MessagingService`. Mark all shim routes as deprecated in OpenAPI spec with `deprecated: true`.  
**Acceptance Criteria**:
- [ ] poc-hermes-app receives the same JSON shape it expects from hermes-api for inbox/outbox
- [ ] `POST /message/send` creates a message visible in conversations (Phase 3 model)
- [ ] Shim endpoints appear in OpenAPI docs with `deprecated: true` and a note referencing the replacement endpoints
- [ ] Integration test: send via shim → message appears in conversations

**Files to Create/Edit**:
- `src/api/routes/messagingShim.ts`

**Doc Reference**: `docs/roadmap/phase-1-foundation.md §Week 4–5`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: **Critical** — this is the backward-compatibility bridge  
**Blocks**: Compatibility gate

---

### [ ] Task 1.5.6: OpenAPI documentation and @fastify/swagger setup
**Purpose**: Auto-generate API documentation from route schemas for integration partners and developers.  
**Description**: Install `@fastify/swagger`, `@fastify/swagger-ui`. Register plugin in `src/api/app.ts`. Configure OpenAPI 3.1 metadata (title, version, license). Serve spec at `GET /docs/openapi.json`; UI at `GET /docs`. Add CI step that validates generated spec is valid OpenAPI 3.1.  
**Acceptance Criteria**:
- [ ] `GET /docs` serves Swagger UI
- [ ] `GET /docs/openapi.json` returns valid OpenAPI 3.1 JSON (validated by `openapi-schema-validator`)
- [ ] All routes appear in spec with request/response schemas
- [ ] Deprecated shim routes marked as `deprecated: true` in spec

**Files to Create/Edit**:
- `src/api/app.ts` (register swagger plugins)

**Doc Reference**: `docs/architecture/overview.md §4.1`  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: Enables API contract verification and automated client generation  
**Blocks**: None

---

## Week 6: CI Polish & Phase Gate Preparation

### [ ] Task 1.6.1: Production Docker Compose and nginx config
**Purpose**: Prepare a production-like deployment that can be smoke-tested before the phase gate.  
**Description**: Create `docker-compose.prod.yml` with resource limits (`mem_limit`, `cpus`), `restart: unless-stopped`, health check configs. Create `infrastructure/nginx/nginx.conf` for TLS termination and reverse proxy to Fastify. Create deployment README at `docs/deployment/runbook.md`.  
**Acceptance Criteria**:
- [ ] `docker compose -f docker-compose.prod.yml up -d` starts all services
- [ ] `GET /health/ready` returns 200 through nginx
- [ ] Runbook documents startup, shutdown, migration, and rollback procedures

**Files to Create/Edit**:
- `docker-compose.prod.yml`
- `infrastructure/nginx/nginx.conf`
- `docs/deployment/runbook.md`

**Doc Reference**: `docs/roadmap/phase-2-realtime.md §Week 11–12`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app must be able to reach the server through nginx  
**Blocks**: Compatibility gate

---

### [ ] Task 1.6.2: RFC template update — decision authority
**Purpose**: Address the governance gap identified in Audit M-5 before any new ADRs or RFCs are drafted.  
**Description**: Update `docs/governance/RFC-TEMPLATE.md` to include: Decision Owner (role definition), acceptance criteria (explicit approval from Engineering Lead + 1 reviewer), objection handling (documented veto → Engineering Lead resolution), fallback behavior after 2-week window with no responses.  
**Acceptance Criteria**:
- [ ] RFC template has a "Decision Owner" field with role definition
- [ ] Template specifies minimum reviewers required for approval
- [ ] Template specifies what happens if objections are unresolved

**Files to Create/Edit**:
- `docs/governance/RFC-TEMPLATE.md`

**Doc Reference**: Audit M-5  
**Estimated Complexity**: Low  
**poc-hermes-app Integration Impact**: None  
**Blocks**: None

---

## Phase 1 Compatibility Verification Gate — Checkpoint

> **This checkpoint must pass before Phase 2 begins.**

### [ ] Task 1.7.1: Run poc-hermes-app against Phase 1 server
**Purpose**: Verify zero regressions for existing poc-hermes-app functionality.  
**Description**: Deploy Phase 1 server via `docker compose up`. Connect poc-hermes-app to the new server. Execute full smoke test covering: login, radio status, PTT, frequency change, file upload/download, system stats, schedule management, inbox/outbox via shim. Produce compatibility report at `docs/qa/compat-phase-1.md`.  
**Acceptance Criteria**:
- [ ] All poc-hermes-app smoke tests pass (zero failures)
- [ ] No HTTP 500 errors in server logs during smoke test
- [ ] `npm audit --audit-level=high` returns zero vulnerabilities
- [ ] `tsc --noEmit` passes
- [ ] `npm test` coverage ≥ 80% on all new modules
- [ ] `docker compose up` (fresh) starts all services with health checks passing
- [ ] Compatibility report signed off by Engineering Lead and Radio Operator SME
- [ ] All Phase 1 audit findings marked resolved in `docs/audit/2026-05-20-architecture-audit.md`

**Files to Create/Edit**:
- `docs/qa/compat-phase-1.md`

**Blocks**: Phase 2 start

---

## Phase 1 Quality Gates

- [ ] `npm run build` passes with zero TypeScript errors
- [ ] `npm test` passes with ≥ 80% coverage on all new modules
- [ ] `docker compose up` starts all services cleanly
- [ ] All new routes have OpenAPI schema annotations
- [ ] All new Drizzle schema changes have a migration file
- [ ] Pino logger is used — no `console.log` in production code
- [ ] All environment variables declared in `src/shared/config.ts`
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] poc-hermes-app compatibility report produced and approved
