# ADR-004: WebSocket Protocol Design

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [websocket-protocol.md](../architecture/websocket-protocol.md)

---

## Context

The current Hermes system uses a C daemon (`sbitx_websocket.c`) with a single WebSocket endpoint that:

1. Broadcasts the same JSON blob to ALL connected clients with no filtering
2. Has no authentication or authorization
3. Has no message type system — clients receive everything
4. Has no subscription model — clients cannot express interest
5. Uses a 100ms polling loop (CPU waste when nothing changes)
6. Has no reconnection/resume protocol
7. Has no error message format
8. Has no request-response correlation (no requestId)
9. Has no heartbeat/health check mechanism
10. Hardcodes buffer sizes in C `sprintf` calls

A new WebSocket protocol must replace this with a typed, authenticated, subscription-based, bidirectional protocol.

---

## Decision

**Transport**: Native `ws` library (not Socket.IO)  
**Subprotocol**: `hermes-v1` (negotiated via `Sec-WebSocket-Protocol` header)  
**Message format**: JSON with typed, versioned `WsEnvelope` wrapper  
**Authentication**: JWT in first message within 10-second window, followed by periodic session re-validation  
**Model**: Topic-based subscriptions with request-response correlation

---

## Rationale

### Custom ws Protocol over Socket.IO

| Criteria | Custom ws (hermes-v1) | Socket.IO |
|---|---|---|
| Client library required | No (native WebSocket API) | Yes (Socket.IO client) |
| Protocol transparency | Full (inspect with any WS tool) | Opaque (Socket.IO framing) |
| Binary message support | Yes | Yes |
| Room/namespace model | Custom (subscription topics) | Built-in namespaces/rooms |
| Reconnection logic | Custom (sync cursor) | Built-in (stateless) |
| Fallback to polling | No (WebSocket only) | Yes (long-polling) |
| Type safety | Yes (TypeScript generics) | Partial |
| Radio-client compatibility | Any WebSocket client | Requires Socket.IO client |
| Debugging | WS inspector tools | Limited to Socket.IO tools |
| Protocol evolution | Full control | Tied to Socket.IO versioning |

Socket.IO's automatic reconnection and room model are convenient but not sufficient justification because:

- Our sync cursor protocol for reconnection provides stronger guarantees than Socket.IO's built-in
- Socket.IO long-polling fallback is unnecessary for our target environments (all support WebSocket)
- The Socket.IO client library adds weight to embedded or constrained clients
- Future radio clients (embedded systems, CLI tools) can implement raw WebSocket more easily

### The WsEnvelope Design

```typescript
interface WsEnvelope<T = unknown> {
  type: string         // Message type identifier (e.g., 'MESSAGE_NEW', 'RADIO_TELEMETRY')
  requestId?: string   // Optional: for request-response correlation
  payload: T           // Type-safe payload
  timestamp: string    // ISO 8601 — for client-side ordering
  version: number      // Protocol envelope version
}
```

The `requestId` enables:
- Client sends `RADIO_COMMAND` with `requestId: 'abc'`
- Server responds with `RADIO_COMMAND_RESULT` with `requestId: 'abc'`
- Client matches response to the original request without global state

The `type` field enables:
- Client-side routing to typed handlers
- Server-side middleware dispatch
- Protocol evolution (add new types without breaking old clients)

The protocol also includes `SESSION_INVALIDATED` so suspended, revoked, or expired sessions can be terminated during active WebSocket use.

### Topic-Based Subscriptions

Clients subscribe to topics they need, reducing unnecessary traffic:

```
radio.telemetry          — Radio status updates (operators only)
conversation.<id>        — New messages in a specific conversation
sync.<deviceId>          — Sync events for this device
presence.<userId>        — Presence updates for a specific user
webrtc.<sessionId>       — WebRTC signaling events
```

This replaces the current "broadcast everything to everyone" model with targeted delivery. A `readonly` role user interested only in GPS tracking does not receive conversation events.

---

## Consequences

### Positive

- Any WebSocket client (browser, mobile, embedded) can connect
- Protocol inspectable with standard WebSocket debugging tools
- Typed event system prevents silent protocol drift
- Subscription model reduces bandwidth (no unnecessary data)
- Authentication from first message (no unauthenticated window beyond 10s)
- Envelope versioning prevents silent protocol drift

### Negative

- Custom protocol means custom documentation and client SDKs
- No automatic reconnection (clients must implement reconnect + SYNC_REQUEST)
- No fallback for environments that block WebSocket (rare for HF radio stations)

### Mitigations

- Client SDK (TypeScript) provided in the hermes-chat monorepo
- Reconnection + sync pattern documented with reference implementation
- WebSocket-blocking environments: use HTTPS long-polling fallback endpoint (future work)

---

## Alternatives Considered

- **Socket.IO**: Rejected (see comparison table above)
- **GraphQL Subscriptions (over WebSocket)**: Heavy for telemetry; adds GraphQL layer complexity
- **gRPC streaming**: Excellent for service-to-service, complex for browser clients
- **SSE (Server-Sent Events)**: Unidirectional — cannot handle client commands
- **MQTT**: Good pub/sub model but requires separate MQTT broker; not browser-native
- **Keep C daemon**: Rejected — 10 identified architectural flaws, no auth, no filtering

---

## Related Documents

- [WebSocket Protocol Specification](../architecture/websocket-protocol.md)
- [Security Architecture](../architecture/security.md) — WebSocket auth details
- [Offline-First Architecture](../architecture/offline-first.md) — Sync on reconnect
