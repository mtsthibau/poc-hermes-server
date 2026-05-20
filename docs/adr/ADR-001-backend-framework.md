# ADR-001: Backend Runtime and Framework Selection

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [overview.md](../architecture/overview.md)

---

## Context

The existing Hermes API backend is a PHP Lumen application using SQLite. This creates several limitations:

- No native real-time/WebSocket capabilities
- Synchronous PHP execution model unsuitable for event-driven communication
- SQLite does not scale for concurrent writes or time-series workloads
- No structured observability or type safety
- CLI calls to `sbitx_client` scattered across controllers with no abstraction

We need to select a new runtime, language, and web framework that supports:

- High-throughput WebSocket connections
- Native async I/O without blocking
- TypeScript for type safety across all layers
- Strong ecosystem for the libraries we need (WebRTC, queues, observability)
- Sufficient performance for radio telemetry streaming
- Developer familiarity across the team

---

## Decision

**Runtime**: Node.js 22 LTS  
**Language**: TypeScript 5.x (strict mode)  
**Framework**: Fastify v5

---

## Rationale

### Node.js 22 LTS over Go, Python, or PHP

| Criteria | Node.js 22 | Go 1.22 | Python 3.12 | PHP 8 |
|---|---|---|---|---|
| TypeScript/type safety | Excellent | Built-in (Go types) | Partial (mypy) | Partial |
| Async I/O model | Event loop, excellent | Goroutines, excellent | Asyncio, good | Poor |
| WebSocket library maturity | Excellent | Good | Good | Poor |
| WebRTC library availability | mediasoup (Node-native) | No production options | No production options | None |
| BullMQ / Redis queue | Excellent | Different ecosystem | Different ecosystem | None |
| Team familiarity | High (hermes-chat is TS) | Low | Medium | Legacy only |
| Shared types with frontend | Yes (TS) | No | No | No |
| Event loop blocking risk | Yes (requires care) | No | No | N/A |

The critical factor: **mediasoup v3** — the only SFU library suitable for radio bridging — is a native Node.js library. There is no equivalent for Go or Python. Using Node.js also enables sharing TypeScript type definitions between the server and the existing `hermes-chat` frontend monorepo.

### Fastify v5 over Express, NestJS, or Hono

| Criteria | Fastify v5 | Express 4 | NestJS | Hono |
|---|---|---|---|---|
| Performance | Excellent (fastest) | Good | Good | Excellent |
| Native JSON Schema validation | Built-in (AJV) | Third-party | Third-party | Third-party |
| Native OpenAPI generation | Built-in | Manual | Swagger module | Third-party |
| TypeScript support | First-class | Types via @types | First-class | First-class |
| Plugin ecosystem | Mature | Massive | Angular-style | Small |
| WebSocket integration | Excellent | Good | Good | Good |
| Opinionated structure | Low | Low | High (modules) | Low |

NestJS was rejected because:
- Its Angular-inspired module system adds significant complexity overhead
- Decorator-based DI introduces runtime coupling that is harder to test and debug
- Fastify achieves the same structure without framework lock-in
- NestJS is a framework that "owns" your architecture; we prefer Fastify as a transport layer we own

Express was rejected for performance and the lack of first-class schema validation.

---

## Consequences

### Positive

- Shared TypeScript types between server and hermes-chat frontend
- mediasoup WebRTC becomes available
- Native event loop handles concurrent WebSocket connections efficiently
- AJV schema validation prevents invalid inputs at the framework boundary
- Auto-generated OpenAPI spec from route schemas

### Negative

- Node.js event loop must not be blocked — CPU-intensive operations must be offloaded to worker threads or external processes (e.g., attachment processing via BullMQ workers)
- Node.js has higher memory overhead per connection than Go
- Single-threaded event loop is a reliability risk if improperly used

### Mitigations

- Telemetry polling runs outside the critical request path (EventBus)
- CLI invocations use `execFile` (non-blocking subprocess)
- BullMQ workers run in separate Node.js processes when needed
- Memory limits monitored via Prometheus; alert at >80% usage

---

## Alternatives Considered

- **Go + Gin**: Excellent performance, but no mediasoup, different ecosystem from frontend
- **Python + FastAPI**: Good async support, but no production WebRTC SFU, slower than Node.js
- **PHP + Laravel**: Legacy ecosystem, poor async/WebSocket support
- **Deno 2**: Promising but immature ecosystem for BullMQ and mediasoup
- **Bun**: Immature for production; not yet stable for all required libraries

---

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [WebRTC Architecture](../architecture/webrtc.md) — mediasoup dependency
- [ADR-007: WebRTC Stack](ADR-007-webrtc-stack.md)
