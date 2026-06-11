# ADR-003: Internal Event Broker Strategy

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [event-driven.md](../architecture/event-driven.md)

---

## Context

The Hermes server requires an internal event bus to decouple:

- HAL hardware events → WebSocket gateway → connected clients
- New message events → delivery workers → offline clients
- Presence changes → conversation subscribers
- Radio state changes → telemetry database writers

Without an event bus, these components would be tightly coupled, making the system harder to test, evolve, or partition into services later.

There are multiple options with very different complexity/capability tradeoffs.

---

## Decision

**Phase 1-3**: Redis Pub/Sub as the internal event bus  
**Phase 4**: Migrate to NATS JetStream for durable, federated messaging  
**Abstraction**: `IEventBus` interface from day one — with explicit fan-out and exclusive-consumer semantics

---

## Rationale

### Why Redis Pub/Sub for Phase 1-3

Redis is already required in the stack for BullMQ, session management, and rate limiting. Using it as an event bus adds zero new infrastructure:

- **No new operational dependency**: Redis is already running
- **Zero-latency fanout**: Events are delivered to all subscribers immediately
- **Pattern subscriptions**: `radio.*`, `message.*` — efficient filtering
- **Simplicity**: No durable consumer groups, no schema registry, no replication lag

The trade-offs of Redis Pub/Sub (no persistence, at-most-once delivery) are acceptable for Phase 1-3 because:
- All state changes are persisted to PostgreSQL **before** publishing
- Offline clients recover state via sync cursors, not event replay
- The system is single-station — no cross-station event routing needed yet

### Why Not Kafka for Phase 1

| Criteria | Redis Pub/Sub | Kafka | RabbitMQ | NATS JetStream |
|---|---|---|---|---|
| Additional infrastructure | None (already present) | Kafka + ZooKeeper/KRaft | RabbitMQ | NATS server |
| Message persistence | No | Yes | Yes | Yes |
| Consumer groups | No | Yes | Yes | Yes |
| At-least-once delivery | No (fire-and-forget) | Yes | Yes | Yes |
| Ordered delivery | Yes (per connection) | Yes (per partition) | Yes (per queue) | Yes (per subject) |
| Federation capability | No | Limited | No | Excellent (leaf nodes) |
| Operational complexity | Minimal | High | Medium | Low |
| Single-station justification | Full | Over-engineered | Over-engineered | Justified at Phase 4 |

Kafka was rejected because its operational complexity (broker replication, partition management, consumer group coordination) is not justified for a single-station deployment. A Kafka cluster for 10-100 concurrent users would be engineering for the sake of engineering.

### Why NATS JetStream for Phase 4

When multiple stations need to exchange events (station-to-station synchronization over radio or internet bridges), we need:

- Durable subscriptions that survive reconnection
- Consumer groups for load-balanced workers across station nodes
- Subject-based routing that maps naturally to station callsigns
- Leaf node topology: each station is a NATS leaf connected to a hub

NATS JetStream provides all of these with lower operational complexity than Kafka and explicit federation support via the hub-spoke leaf node topology.

### The IEventBus Abstraction

The abstraction preserves application structure across brokers, but the code must be explicit about fan-out versus exclusive-consumer behavior:

```typescript
interface IEventBus {
  publish<T>(topic: string, payload: T): Promise<void>
  subscribe<T>(topic: string, handler: EventHandler<T>): Unsubscribe
  subscribePattern<T>(pattern: string, handler: EventHandler<T>): Unsubscribe
  subscribeExclusive<T>(topic: string, group: string, handler: EventHandler<T>): Unsubscribe
}

// Phase 1-3:
const eventBus: IEventBus = new RedisEventBus(redis)

// Phase 4:
const eventBus: IEventBus = new NatsEventBus(natsConnection)
```

Publishers stay transport-agnostic, but consumers must choose the correct semantic API:

- `subscribe()` / `subscribePattern()` for fan-out subscribers such as the WebSocket gateway
- `subscribeExclusive()` for worker-style consumers such as delivery, sync, and telemetry ingest workers

---

## Consequences

### Positive

- Zero additional infrastructure in Phase 1-3
- Clean upgrade path to NATS without changing producer contracts
- Event handlers are testable in isolation (mock IEventBus)
- Pattern subscriptions enable efficient filtering

### Negative

- Redis Pub/Sub is at-most-once — if a subscriber is restarted, it misses events published during downtime
- No event replay — a new subscriber cannot catch up on historical events
- Redis fan-out and NATS consumer-group behavior are intentionally modeled as different APIs

### Mitigations

- All state mutations are written to PostgreSQL before publishing (DB is source of truth)
- Missed events recovered via sync cursor mechanism on reconnect
- Critical delivery operations use BullMQ (durable) not Redis Pub/Sub
- Worker restart recovery: workers query DB for unprocessed items on startup

---

## Alternatives Considered

- **NATS JetStream from day one**: Valid but premature — adds infrastructure without current justification
- **Kafka**: Powerful but operationally heavy; rejected for initial deployment
- **RabbitMQ**: Good option but NATS has better federation model for radio networks
- **EventEmitter (in-process)**: Cannot scale to multiple server processes
- **PostgreSQL LISTEN/NOTIFY**: Native but limited (no pattern subscriptions, 8KB payload limit)

---

## Related Documents

- [Event-Driven Architecture](../architecture/event-driven.md)
- [Offline-First Architecture](../architecture/offline-first.md)
- [ADR-002: Database Strategy](ADR-002-database-strategy.md)
