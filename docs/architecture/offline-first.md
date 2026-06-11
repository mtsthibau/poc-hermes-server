# Offline-First & Synchronization Architecture

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-06-10

---

## 1. Why Offline-First Is Non-Negotiable

Hermes stations operate in conditions where connectivity is unreliable by design:

- HF radio links are intermittent — they depend on ionospheric propagation
- Rural and disaster-affected deployments may have no internet at all
- UUCP connections to gateway stations are scheduled, not always-on
- Mobile clients may lose connectivity while composing or sending messages
- Network partitions between stations can last hours or days

A system that assumes persistent connectivity will fail its users in the exact moments it is most needed. Offline-first is not a feature — it is a foundational design constraint.

---

## 2. Offline-First Principles Applied

| Principle | Implementation |
|---|---|
| Local-write-first | All client writes are acknowledged locally before network delivery |
| Sync on reconnect | On connection restore, client and server exchange only what changed |
| Idempotent operations | Every operation can be safely replayed without duplicate effects |
| Conflict awareness | Last-write-wins with timestamp; future CRDTs for complex entities |
| No data loss | All undelivered messages are durably queued in BullMQ + PostgreSQL |
| State convergence | Eventually consistent — all nodes converge to same state |
| Ordered delivery | Sync uses sequence numbers to preserve causal ordering |

---

## 3. Synchronization Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Client Device                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Local State (IndexedDB / SQLite on mobile)              │    │
│  │  - messages, conversations, reactions, deliveries       │    │
│  │  - sync_cursors: { messages: seq, reactions: seq, ... } │    │
│  └────────────────────────┬──────────────────────────────── ┘    │
│                           │                                      │
│  [Reconnect event]        │                                      │
│  cursor = localStorage.get('sync_cursor')                        │
└───────────────────────────╪──────────────────────────────────────┘
                            │ WebSocket: SYNC_REQUEST { cursors }
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                      poc-hermes-server                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ SyncService                                             │    │
│  │  1. Validate user/device cursors                        │    │
│  │  2. Query sync_queue for events since cursor            │    │
│  │  3. Fetch current entity state for each queued ref      │    │
│  │  4. Batch into SYNC_DELTA frames (100 events/batch)     │    │
│  │  5. Stream frames to client                             │    │
│  │  6. Send SYNC_COMPLETE with new cursor values           │    │
│  └────────────────────────┬──────────────────────────────── ┘    │
│                           │                                      │
│  ┌────────────────────────▼──────────────────────────────── ┐    │
│  │ sync_queue (PostgreSQL)                                 │    │
│  │  Append-only queue of entity mutations per user/device  │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
                            │ WebSocket: SYNC_DELTA batches
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Client Device                               │
│  [Apply each SYNC_DELTA event to local state]                    │
│  [Update local sync_cursor after SYNC_COMPLETE]                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Sync Cursor Model

Each client device maintains cursor positions per entity type. The server uses these cursors to compute exactly what changed since the client's last known state.

```typescript
// Client stores this locally (IndexedDB / AsyncStorage)
interface SyncCursors {
  messages: number        // Sequence number of last received message event
  reactions: number       // Sequence number of last received reaction event
  deliveries: number      // Sequence number of last received delivery update
  conversations: number   // Sequence number of last conversation update
  presences: number       // Sequence number of last presence update
  updatedAt: string       // ISO timestamp of last successful sync
}
```

**Server-side cursor tracking** (`sync_cursors` table):
- Server records `last_event_sequence` per user/device/entity_type
- On sync request, server computes delta: `WHERE sequence > cursor AND target_user_id = userId`
- Cursor advancement happens on both successful WebSocket push delivery and sync replay completion
- Monotonic sequence is ensured by PostgreSQL sequence (not timestamp — clocks can skew)

---

## 5. Outbound Message Queue (Client Offline)

When a client sends a message while offline or during a connection gap:

```
[Client action: user hits send]
  │
  ├─► Store message in local DB with status='sending', clientMessageId=UUID
  ├─► Add to local outbox queue
  │
  [When connectivity restored]
  │
  └─► POST /api/v1/conversations/:id/messages
        body: { clientMessageId: UUID, content: '...' }
        │
        ├─► [Server: idempotency check]
        │    SELECT id FROM messages WHERE client_message_id = UUID
        │    → if found: return existing message (idempotent)
        │    → if not found: create new message
        │
        └─► Client updates local message status to 'sent'
```

The `clientMessageId` (UUID generated by client) is the idempotency key. The server will never create duplicate messages for the same `clientMessageId`, even if the client retries the same request multiple times.

---

## 6. Inbound Delivery Queue (Server → Offline Client)

When a message is sent to a recipient who is offline:

```
[MessagingService sends message to user X]
  │
  ├─► Check: is user X online (any WebSocket connection)?
  │    [Check Redis: SET hermes:presence:userId with TTL]
  │
  ├─► [ONLINE] → Push MESSAGE_NEW via WebSocket immediately
  │     ├─► [ws.send() error] → Insert sync_queue reference immediately
  │     └─► [send success] → Wait for MESSAGE_ACK (30s timeout)
  │           ├─► [ACK received] → mark delivery + advance sync cursor
  │           └─► [No ACK] → Insert sync_queue reference for that device
  │
  └─► [OFFLINE] → SyncQueueWorker:
        ├─► INSERT INTO sync_queue (target_user_id, entity_type='message',
        │     entity_id=messageId, operation='create')
        │
        └─► [Optional] Enqueue PushNotificationJob in BullMQ
              └─► Send FCM/APNs push to device
```

`message_deliveries` is the durable source of truth. A `MissedDeliveryRecoveryWorker` periodically scans for `status='pending'` rows older than 60 seconds and re-enqueues delivery attempts so a crash between database commit and real-time notification cannot silently lose a delivery.

---

## 7. Radio Station Synchronization

For offline radio stations (not connected to internet), synchronization happens via the existing radio transport:

```
Station A (internet-connected)
  │ Composes message for Station B
  ├─► Message stored in DB (status=sending)
  ├─► DeliveryService creates message_delivery (channel=radio, status=pending)
  │
  ├─► RadioDeliveryWorker (BullMQ)
  │     ├─► Packages message as HMP (Hermes Message Pack)
  │     ├─► Enqueues for UUCP delivery to Station B's callsign
  │     └─► Submits to connection_schedules or immediate UUCP call
  │
  └─► [When UUCP connection to Station B established]
        └─► HMP transferred → Station B's API ingests
              └─► Creates message in Station B's conversation
              └─► Emits message.new event → Station B's clients notified
```

This flow preserves the existing UUCP/HMP transport while integrating it into the new messaging domain model via the `message_envelopes` table (transport=`radio`).

---

## 8. Conflict Resolution Strategy

### Last-Write-Wins (Phase 1-3)

For the initial implementation, conflicts are resolved by timestamp:

- `updated_at` is always set server-side at write time (not client-provided)
- Client-side timestamp is never trusted for ordering
- Edit conflicts (two clients edit same message simultaneously): the later write wins
- This is acceptable for the current use case where simultaneous edits are rare

### Future: CRDTs (Phase 4+)

For distributed/federated deployments where network partitions can cause write conflicts:

- Message content: `LWW-Register` (last-write-wins, sufficient for most cases)
- Reactions: `OR-Set` (observed-remove set — concurrent add/remove handled correctly)
- Conversation participants: `OR-Set` (add/remove operations)
- Read positions: `Max-Register` (read position only moves forward)

CRDTs eliminate the need for central coordination and work naturally with the eventual consistency model of federated HF radio networks.

---

## 9. Presence System

User presence (online/offline/away) is tracked in Redis with TTL:

```typescript
// When user authenticates on WebSocket:
await redis.setex(`presence:${userId}`, 90, JSON.stringify({
  status: 'online',
  deviceId,
  connectedAt: new Date().toISOString(),
}))

// Heartbeat refreshes TTL every 30s
// TTL expiry = user went offline (no explicit offline event needed)

// Presence check:
const raw = await redis.get(`presence:${userId}`)
const isOnline = raw !== null
```

Presence events are published to `presence.online` / `presence.offline` topics and delivered to WebSocket subscribers.

**Privacy**: Presence is only broadcast to users who share a conversation with the target user. Global presence broadcast is disabled.

---

## 10. Synchronization Security

- Sync cursors are per-user-device — one device cannot advance another device's cursor
- Sync delta queries enforce `WHERE target_user_id = authenticatedUserId` — no cross-user leakage
- `entity_type` and `operation` are validated before insertion; payloads are resolved from current database state at sync time
- Sync requests are rate-limited: max 10 sync requests per minute per device
- Large sync payloads (>1000 events) are flagged and reviewed — may indicate sync cursor reset attack

---

## 11. Retry Policies

| Operation | Initial Delay | Max Retries | Backoff |
|---|---|---|---|
| WebSocket delivery | 5s | 10 | Exponential (max 5 min) |
| Push notification | 10s | 5 | Exponential |
| UUCP radio delivery | 1 hour | 24 | Fixed (scheduled windows) |
| SMTP email delivery | 5 min | 5 | Exponential |
| HAL command | 500ms | 3 | Exponential |
| Attachment processing | 1 min | 2 | Fixed |

After max retries exhausted:
- `message_deliveries.status` → `failed`
- `message_deliveries.error` → reason
- Dead letter queue review (BullMQ failed jobs retained for 24h)
- Optional: alert operator for radio delivery failures

---

## 12. Sync Queue Cleanup

The sync_queue table is append-only during normal operation. It is periodically cleaned:

```sql
-- CleanupWorker runs daily
-- Keep processed items for 7 days (for debugging)
-- Delete older processed items
DELETE FROM sync_queue
WHERE processed_at IS NOT NULL
  AND processed_at < NOW() - INTERVAL '7 days';

-- Delete unprocessed items for users that no longer exist
DELETE FROM sync_queue sq
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.id = sq.target_user_id);
```
