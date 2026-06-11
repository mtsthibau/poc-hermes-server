# WebSocket Protocol Specification

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-06-10

---

## 1. Problems with the Current Implementation

The existing `sbitx_websocket.c` implementation has the following critical architectural flaws:

| Problem | Description |
|---|---|
| No authentication | Any client connecting to the socket receives all telemetry |
| Single broadcast | All data sent to ALL connected clients — no filtering |
| No event typing | Flat JSON blob with mixed concerns (telemetry + message notification) |
| No subscription model | Clients cannot subscribe to specific event streams |
| No reconnect/resume | Lost connections lose all events — no catch-up mechanism |
| Coupled to C daemon | WebSocket is embedded in hardware process — cannot scale independently |
| No backpressure | Slow clients block the event loop |
| 100ms polling | Fixed poll interval regardless of whether state changed |
| sprintf buffer building | Fragile, vulnerable to buffer issues |
| Mixed domains | Radio telemetry, message availability, and datetime in one payload |

---

## 2. New Protocol Design

### 2.1 Transport

- **URL**: `wss://station-host/ws`
- **Protocol**: WebSocket (RFC 6455) over TLS
- **Subprotocol**: `hermes-v1` (negotiated via `Sec-WebSocket-Protocol` header)
- **Encoding**: JSON text frames (binary frames reserved for future audio streaming)
- **Connection flow**: Connect → Authenticate → Subscribe → Receive events → Disconnect

### 2.2 Message Envelope

Every message (client → server and server → client) follows this envelope:

```typescript
interface WsEnvelope {
  type: WsMessageType       // Event type identifier
  requestId?: string        // Client-provided correlation ID (echoed in response)
  payload: unknown          // Type-specific payload
  timestamp: string         // ISO 8601 UTC timestamp
  version: number           // Protocol envelope version
}
```

Example:
```json
{
  "type": "SUBSCRIBE",
  "requestId": "req-abc-123",
  "payload": { "topics": ["radio.telemetry", "message.new"] },
  "timestamp": "2026-05-20T14:32:00.000Z",
  "version": 1
}
```

### 2.3 Message Types

#### Client → Server

| Type | Description |
|---|---|
| `AUTHENTICATE` | Initial authentication handshake (JWT token) |
| `SUBSCRIBE` | Subscribe to one or more topics |
| `UNSUBSCRIBE` | Unsubscribe from topics |
| `PING` | Heartbeat (client-initiated) |
| `PRESENCE` | Report presence/activity state |
| `TYPING_START` | User started typing in a conversation |
| `TYPING_STOP` | User stopped typing |
| `MESSAGE_ACK` | Acknowledge message delivery |
| `SYNC_REQUEST` | Request delta sync from a cursor position |
| `RADIO_COMMAND` | Send a command to the radio HAL (if authorized) |

#### Server → Client

| Type | Description |
|---|---|
| `AUTHENTICATED` | Auth successful, session established |
| `AUTH_ERROR` | Authentication failed |
| `SESSION_INVALIDATED` | Session revoked, expired, or suspended during re-validation |
| `SUBSCRIBED` | Subscription confirmation |
| `UNSUBSCRIBED` | Unsubscription confirmation |
| `PONG` | Heartbeat response |
| `ERROR` | Protocol or application error |
| `RADIO_TELEMETRY` | Periodic radio state snapshot |
| `RADIO_STATE_CHANGE` | Significant radio state change (PTT, connection, protection) |
| `MESSAGE_NEW` | New message received in a subscribed conversation |
| `MESSAGE_UPDATED` | Message content edited |
| `MESSAGE_DELETED` | Message soft-deleted |
| `MESSAGE_DELIVERED` | Delivery receipt for a sent message |
| `MESSAGE_READ` | Read receipt from a recipient |
| `REACTION_ADDED` | Emoji reaction added to a message |
| `REACTION_REMOVED` | Emoji reaction removed |
| `TYPING_START` | A participant is typing in a conversation |
| `TYPING_STOP` | A participant stopped typing |
| `PRESENCE_UPDATE` | User presence changed (online/offline/away) |
| `SYNC_DELTA` | Batch of missed events (replay on reconnect) |
| `SYNC_COMPLETE` | Delta sync complete, cursor updated |
| `WEBRTC_OFFER` | WebRTC offer signal (from another peer) |
| `WEBRTC_ANSWER` | WebRTC answer signal |
| `WEBRTC_ICE_CANDIDATE` | ICE candidate for WebRTC negotiation |
| `WEBRTC_SESSION_ENDED` | WebRTC session terminated |
| `SYSTEM_NOTIFICATION` | System-level notification (maintenance, etc.) |

---

## 3. Connection Lifecycle

```
Client                                              Server
  │                                                   │
  │──── TCP/TLS handshake ────────────────────────────│
  │──── WebSocket Upgrade (Sec-WebSocket-Protocol: hermes-v1) ─│
  │                                                   │
  │  [Server starts 10s auth timeout]                 │
  │                                                   │
  │──── AUTHENTICATE ─────────────────────────────────│
  │     { token: "eyJhb...", deviceId: "uuid" }       │
  │                                                   │
  │◄─── AUTHENTICATED ────────────────────────────────│
  │     { userId, callsign, role, sessionId }         │
  │                                                   │
  │──── SUBSCRIBE ─────────────────────────────────────│
  │     { topics: ["radio.telemetry", "message.new",  │
  │                "conversation.uuid-xyz"] }         │
  │                                                   │
  │◄─── SUBSCRIBED ───────────────────────────────────│
  │     { topics: [...], grantedTopics: [...] }       │
  │                                                   │
  │  [Normal operation]                               │
  │◄─── RADIO_TELEMETRY (every ~1s) ──────────────────│
  │◄─── MESSAGE_NEW ──────────────────────────────────│
  │◄─── TYPING_START ─────────────────────────────────│
  │                                                   │
  │  [Server heartbeat every 30s]                     │
  │◄─── PING ─────────────────────────────────────────│
  │──── PONG ─────────────────────────────────────────│
  │                                                   │
  │  [Periodic session re-validation]                │
  │◄─── SESSION_INVALIDATED ─────────────────────────│
  │     { reason: "revoked" | "expired" | "suspended" } │
  │──── WebSocket Close ───────────────────────────────│
  │                                                   │
  │  [Client disconnects or times out]                │
  │──── WebSocket Close ───────────────────────────────│
  │                                                   │
  │  [Client reconnects]                              │
  │──── AUTHENTICATE ─────────────────────────────────│
  │──── SYNC_REQUEST ─────────────────────────────────│
  │     { cursors: { messages: "seq-1234", ... } }    │
  │                                                   │
  │◄─── SYNC_DELTA (batch) ───────────────────────────│
  │◄─── SYNC_DELTA (batch) ───────────────────────────│
  │◄─── SYNC_COMPLETE ────────────────────────────────│
```

---

## 4. Topic Subscription System

Topics follow a hierarchical dot-notation naming scheme. Authorization is enforced at subscription time — clients cannot subscribe to topics they lack permission for.

### 4.1 Topic Definitions

| Topic | Description | Required Role |
|---|---|---|
| `radio.telemetry` | Radio telemetry stream | user |
| `radio.state` | Significant radio state changes | user |
| `radio.commands` | Radio command results | operator |
| `message.new` | New messages in any subscribed conversation | user |
| `conversation.<id>` | All events for a specific conversation | member |
| `conversation.<id>.typing` | Typing indicators for a conversation | member |
| `presence.<userId>` | Presence updates for a user | user |
| `sync.<deviceId>` | Sync events for this device | self |
| `system` | System-level notifications | user |
| `webrtc.<sessionId>` | WebRTC signaling for a session | participant |
| `geolocation` | Real-time GPS updates | user |
| `admin.*` | Administrative events | admin |

### 4.2 Subscription Authorization Rules

- A client can only subscribe to `conversation.<id>` if they are a participant in that conversation
- A client can only subscribe to `sync.<deviceId>` for their own device ID
- `admin.*` topics require the `admin` role
- `radio.commands` requires the `operator` role
- A client's topic grants are re-evaluated on each subscription request

---

## 5. Payload Schemas

### 5.1 AUTHENTICATE

```typescript
// Client → Server
interface AuthenticatePayload {
  token: string       // JWT access token
  deviceId?: string   // UUID of user's device (for sync targeting)
  resumeSessionId?: string  // Previous session ID for event replay
  version: number     // Must be 1 for hermes-v1
}

// Server → Client (success)
interface AuthenticatedPayload {
  userId: string
  callsign: string
  role: 'admin' | 'operator' | 'user' | 'readonly'
  sessionId: string   // New WebSocket session ID (use for resume)
  serverTime: string  // Server timestamp for clock sync
}

// Server → Client (failure)
interface AuthErrorPayload {
  code: 'TOKEN_EXPIRED' | 'TOKEN_INVALID' | 'USER_SUSPENDED' | 'TIMEOUT'
  message: string
}
```

### 5.2 RADIO_TELEMETRY

```typescript
// Server → Client
// Replaces the flat sprintf JSON from sbitx_websocket.c
// Structured, typed, versioned
interface RadioTelemetryPayload {
  stationId: string
  timestamp: string
  radio: {
    frequencyHz: number       // was: flat "freq" across multiple profiles
    mode: 'USB' | 'LSB' | 'CW' | 'AM' | 'FM' | 'DIGITAL'
    tx: boolean
    rx: boolean
    snr: number               // dB
    bitrate: number           // bps
    bytesTransmitted: number
    bytesReceived: number
    powerLevel: number | null
  }
  system: {
    ok: boolean               // was: "led"
    connected: boolean
    protection: boolean       // SWR protection
    profileActiveIdx: number
    timeoutCounter: number
    digitalVoice: boolean
  }
  profiles: Array<{
    idx: number
    frequencyHz: number
    mode: string
    volume: number
    digitalVoice: boolean
  }>
}
```

### 5.3 MESSAGE_NEW

```typescript
// Server → Client
interface MessageNewPayload {
  message: {
    id: string
    conversationId: string
    senderId: string
    senderCallsign: string
    content: string | null
    contentType: string
    replyToMessageId: string | null
    subject: string | null
    attachments: AttachmentSummary[]
    createdAt: string
  }
  conversation: {
    id: string
    type: string
    title: string | null
  }
}
```

### 5.4 SYNC_REQUEST / SYNC_DELTA

```typescript
// Client → Server
interface SyncRequestPayload {
  cursors: {
    messages?: number           // Last known message sequence
    reactions?: number
    deliveries?: number
    conversations?: number
    presences?: number
  }
  conversationIds?: string[]    // Limit sync to specific conversations
  limit?: number                // Max events per delta batch (default: 100)
}

// Server → Client (sent in batches)
interface SyncDeltaPayload {
  batchId: string
  batchIndex: number
  totalBatches: number
  events: Array<{
    sequence: number
    type: string                // Same type values as WsMessageType
    payload: unknown
    timestamp: string
  }>
}

// Server → Client (final)
interface SyncCompletePayload {
  cursors: {                    // Updated cursors client should persist
    messages: number
    reactions: number
    deliveries: number
    conversations: number
  }
  missedEvents: number          // Total events delivered
}
```

### 5.5 TYPING_START / TYPING_STOP

```typescript
// Client → Server
interface TypingClientPayload {
  conversationId: string
}

// Server → Client (broadcast to conversation subscribers)
interface TypingServerPayload {
  conversationId: string
  userId: string
  callsign: string
  expiresAt: string             // Auto-stops after 5 seconds if no TYPING_STOP
}
```

### 5.6 RADIO_COMMAND

```typescript
// Client → Server (operator+ role required)
interface RadioCommandPayload {
  command: 'set_frequency' | 'set_mode' | 'set_profile' | 'set_volume' |
           'set_ptt' | 'set_protection' | 'reset_defaults' | 'restart_voice_timeout'
  params: Record<string, unknown>  // Command-specific parameters
}

// Server → Client (command result)
interface RadioCommandResultPayload {
  requestId: string
  success: boolean
  result?: unknown
  error?: { code: string; message: string }
}
```

---

## 6. Error Handling

All protocol errors follow this schema:

```typescript
interface ErrorPayload {
  code: string        // Machine-readable error code
  message: string     // Human-readable description
  requestId?: string  // Echo of client requestId if applicable
  details?: unknown   // Optional additional context
}
```

Common error codes:

| Code | Description |
|---|---|
| `UNAUTHENTICATED` | Message received before authentication |
| `AUTH_TIMEOUT` | Authentication not completed in 10 seconds |
| `UNAUTHORIZED` | Insufficient role for requested operation |
| `SUBSCRIPTION_DENIED` | Topic subscription denied |
| `INVALID_PAYLOAD` | Message payload failed schema validation |
| `RATE_LIMITED` | Too many messages from this connection |
| `COMMAND_TIMEOUT` | Radio command did not complete in time |
| `COMMAND_FAILED` | Radio command execution failed |
| `UNKNOWN_TYPE` | Unrecognized message type |

---

## 7. Heartbeat & Connection Health

- Server sends `PING` every **30 seconds** if no data received
- Client must respond with `PONG` within **10 seconds**
- If no `PONG` received within 10 seconds, connection is terminated with code `1001`
- Clients should also send periodic `PING` frames as an application-level check
- Connection state tracked per session in Redis with TTL

---

## 8. Rate Limiting

| Connection state | Limit |
|---|---|
| Pre-authentication | 3 messages total, then close |
| Authenticated (default) | 60 messages/minute |
| Authenticated (operator) | 300 messages/minute |
| `TYPING_START` events | 1 per 2 seconds per conversation |
| `RADIO_COMMAND` events | 10 per minute |

Exceeding limits results in an `ERROR` frame with code `RATE_LIMITED`. Repeated violations result in connection termination.

---

## 9. Server Implementation Notes

```typescript
// src/gateway/protocol/router.ts — simplified routing

class WsMessageRouter {
  private handlers = new Map<WsMessageType, WsHandler>()

  register(type: WsMessageType, handler: WsHandler): void {
    this.handlers.set(type, handler)
  }

  async route(connection: WsConnection, envelope: WsEnvelope): Promise<void> {
    // Schema validation
    const validated = this.validateEnvelope(envelope)
    
    // Auth check (all non-AUTHENTICATE messages require auth)
    if (envelope.type !== 'AUTHENTICATE' && !connection.isAuthenticated) {
      return connection.send({ type: 'ERROR', payload: { code: 'UNAUTHENTICATED' } })
    }

    // Rate limiting
    await this.rateLimiter.check(connection)

    // Route to handler
    const handler = this.handlers.get(envelope.type)
    if (!handler) {
      return connection.send({ type: 'ERROR', payload: { code: 'UNKNOWN_TYPE' } })
    }

    await handler.handle(connection, validated)
  }
}
```

---

## 10. Migration from sbitx_websocket.c

During transition, the new gateway can bridge the legacy C websocket:

```
Legacy C daemon (port 8443 internal)
  → New Gateway subscribes as a client to legacy telemetry
  → Gateway normalizes and re-emits as typed RADIO_TELEMETRY events
  → Clients connect to new gateway only

This allows incremental migration without disrupting existing hardware.
```

Once the HAL is fully implemented and validated, the legacy bridge is removed.
