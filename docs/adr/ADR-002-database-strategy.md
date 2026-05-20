# ADR-002: Database Strategy

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [database-schema.md](../architecture/database-schema.md)

---

## Context

The existing system uses **SQLite** for all persistent storage. This creates hard ceilings:

- Single-writer limitation blocks concurrent API requests
- No native support for time-series data (radio telemetry, GPS readings)
- No native JSON/JSONB querying for extensible metadata
- No row-level security or schema-level access control
- Not suitable for multi-device synchronization workloads
- Full-text search is limited

We need a persistence strategy that:

- Handles concurrent writes from API + WebSocket gateway + background workers
- Efficiently stores and queries time-series telemetry data
- Supports normalized relational schema with foreign key constraints
- Enables cursor-based synchronization queries
- Provides caching and ephemeral state (sessions, presence, rate limiting)
- Scales from a single Raspberry Pi station to a multi-server cluster

---

## Decision

**Primary Database**: PostgreSQL 17  
**Time-Series Extension**: TimescaleDB (on same PostgreSQL instance)  
**Cache / Pub-Sub / Queue Backend**: Redis 7  
**ORM**: Drizzle ORM  

---

## Rationale

### PostgreSQL 17 over MongoDB, MySQL, or SQLite

| Criteria | PostgreSQL 17 | MongoDB | MySQL 8 | SQLite |
|---|---|---|---|---|
| ACID transactions | Full | Multi-doc (limited) | Full | Full |
| Foreign key constraints | Yes | No | Yes | Yes |
| JSONB native querying | Excellent | Native (document) | JSON (limited) | No |
| Full-text search | Built-in | Built-in | Limited | No |
| TimescaleDB extension | Yes | No | No | No |
| Row-level security | Yes | Collection-level | Limited | No |
| Concurrent writes | Excellent | Good | Good | Poor |
| Schema migrations | Standard SQL | Schema-less | Standard SQL | Limited |
| Normalization support | Full | Not applicable | Full | Full |

MongoDB was considered for its document model flexibility. Rejected because:
- Hermes data is relational (users → conversations → messages → deliveries) — a document model adds JOIN complexity without benefit
- No foreign key enforcement means referential integrity must be maintained in application code
- TimescaleDB is not available for MongoDB
- Our schema is well-defined, not schema-less

### TimescaleDB over InfluxDB or VictoriaMetrics

TimescaleDB runs as a PostgreSQL extension, which means:
- No additional database process to manage
- Time-series data queries can JOIN with relational tables (e.g., GPS reading JOIN with users)
- Same connection pool, same ORM, same backup strategy
- Compression, retention policies, and hypertable chunking all built-in

If TimescaleDB becomes a bottleneck (high-frequency telemetry at scale), the migration path to a dedicated TSDB is straightforward: stop writing to TimescaleDB, write to external InfluxDB, update `radio_telemetry` reads to query InfluxDB. The abstraction layer (TimescaleDB tables) would remain in place for historical data.

### Redis 7 over Memcached or in-process cache

Redis is required for:
- **BullMQ**: Persistent job queues (requires Redis)
- **Pub/Sub**: Internal event bus
- **Session store**: Refresh token storage (hashed)
- **Presence state**: TTL-based online/offline
- **Rate limiting**: Sliding window counters
- **WebSocket routing**: Track which user/device is connected to which process

Memcached lacks pub/sub and sorted sets (needed for rate limiting). An in-process cache cannot be shared across multiple server processes.

### Drizzle ORM over Prisma or TypeORM

| Criteria | Drizzle ORM | Prisma 5 | TypeORM |
|---|---|---|---|
| Type safety | Excellent (SQL-like) | Excellent (schema-first) | Good |
| Code generation | No (schema in TS) | Yes (requires generate step) | No |
| Raw SQL escape | Easy | Complex | Medium |
| Bundle size | Small | Large (query engine) | Medium |
| Migration tooling | drizzle-kit | prisma migrate | typeorm CLI |
| PostgreSQL features | Full access | Partial (abstracted) | Good |
| TimescaleDB queries | Full (raw SQL escape) | Limited | Partial |

Prisma was rejected because its Rust-based query engine adds deployment complexity and bundle weight. Drizzle's schema-as-TypeScript model means the schema definition IS the type definition — no code generation step, no drift between schema and types.

---

## Consequences

### Positive

- PostgreSQL MVCC enables concurrent reads without blocking writes
- TimescaleDB handles telemetry retention, compression, and chunk pruning automatically
- Redis Pub/Sub enables zero-latency internal event bus
- Drizzle types are inferred from schema — no type drift

### Negative

- More infrastructure components to manage (PostgreSQL + Redis)
- TimescaleDB requires the extension to be installed on PostgreSQL
- Redis is a single point of failure for queues/pub-sub (mitigated by persistence mode)

### Mitigations

- Redis configured with `appendonly yes` (AOF persistence) for queue durability
- Redis Sentinel or Cluster for high-availability in Phase 4
- PostgreSQL replication (streaming) for read scaling and failover in Phase 4
- Docker Compose setup handles all infrastructure components in development

---

## Alternatives Considered

- **Pure MongoDB**: Document flexibility without relational structure — rejected (see above)
- **CockroachDB**: Distributed PostgreSQL-compatible — too complex for current scale
- **PlanetScale (MySQL)**: Cloud-only — field deployments have no internet
- **SQLite + WAL mode**: Concurrent reads, single writer — insufficient for multi-process server
- **In-memory SQLite for testing**: Retained — test environment uses SQLite in-memory via Drizzle adapters

---

## Related Documents

- [Database Schema Design](../architecture/database-schema.md)
- [Offline-First Architecture](../architecture/offline-first.md)
- [Event-Driven Backend](../architecture/event-driven.md)
