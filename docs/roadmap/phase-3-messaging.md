# Phase 3 — Chat-Native Messaging & Synchronization

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 13–20  
**Prerequisites**: Phase 2 complete and deployed  
**Goal**: Full chat-native messaging model, delivery tracking, offline sync engine, and email interoperability

---

## Objectives

By the end of Phase 3:

1. Conversations, messages, reactions, and delivery tracking fully operational
2. WebSocket push delivery of new messages to online clients
3. Offline sync engine with cursor-based reconnect
4. Attachment pipeline with MIME validation, dedup, signed URLs
5. HMP/UUCP email interoperability preserved via `message_envelopes`
6. Message editing, deletion, and reply threading
7. Typing indicators

---

## Week 13: Messaging Domain Model

### Core Entities Implementation

- [ ] `ConversationRepository`:
  - `create(type, participantIds, metadata)` → creates conversation + participants
  - `getById(id, userId)` → verifies membership
  - `listForUser(userId, cursor, limit)` → paginated, ordered by last message
  - `archive(id, userId)` → soft archive
  - `addParticipant(id, userId)` → add member
  - `removeParticipant(id, userId)` → remove member

- [ ] `MessageRepository`:
  - `create(conversationId, senderId, content, clientMessageId)` → idempotent
  - `getById(id, userId)` → fetch with access check
  - `listByConversation(conversationId, cursor, limit)` → paginated
  - `update(id, userId, content)` → edit (sender only, within time window)
  - `softDelete(id, userId)` → sets `deleted_at`

- [ ] `DeliveryRepository`:
  - `createBatch(messageId, recipientIds)` → one row per recipient per channel
  - `markDelivered(messageId, recipientId, channel)` → status transition
  - `markRead(messageId, recipientId)` → status transition

### REST API — Conversations

- [ ] `POST /api/v1/conversations` — create conversation
- [ ] `GET /api/v1/conversations` — list user's conversations (unread count included)
- [ ] `GET /api/v1/conversations/:id` — get conversation + participants
- [ ] `POST /api/v1/conversations/:id/participants` — add participant (operator/admin)
- [ ] `DELETE /api/v1/conversations/:id/participants/:userId` — remove participant

### REST API — Messages

- [ ] `POST /api/v1/conversations/:id/messages` — send message (idempotent via clientMessageId)
- [ ] `GET /api/v1/conversations/:id/messages` — list messages (cursor pagination)
- [ ] `PATCH /api/v1/conversations/:id/messages/:messageId` — edit message
- [ ] `DELETE /api/v1/conversations/:id/messages/:messageId` — delete message

---

## Week 14: Delivery Pipeline

### Online Delivery (WebSocket)

- [ ] `MessagingService.sendMessage()` publishes `message.new` event after DB write
- [ ] WebSocket gateway subscribes to `message.*` topics
- [ ] For each participant subscribed to `conversation.<id>`: push `MESSAGE_NEW` frame
- [ ] Client sends `MESSAGE_ACK` → gateway marks delivery `delivered` (WebSocket channel)

### Offline Delivery (Sync Queue)

- [ ] `DeliveryWorker` (BullMQ) triggered by `message.new` event for offline recipients
- [ ] Inserts into `sync_queue` per offline recipient device
- [ ] `MessageDeliveryWorker` (BullMQ): per-recipient delivery attempts
- [ ] Retry policy: 5x exponential backoff (see offline-first.md)

### Read Status

- [ ] Client sends `MESSAGE_READ { conversationId, messageId }` via WebSocket
- [ ] Server updates `conversation_participants.last_read_message_id`
- [ ] Publishes `message.read` event → broadcast `MESSAGE_READ` to other participants

### Typing Indicators

- [ ] Client sends `TYPING_START { conversationId }` / `TYPING_STOP { conversationId }`
- [ ] Server publishes `typing.started` / `typing.stopped` topic
- [ ] Gateway fans out to conversation subscribers (excluding sender)
- [ ] Auto-stop typing indicator after 5 seconds of no TYPING_START

---

## Week 15: Reactions & Threading

### Reactions

- [ ] `POST /api/v1/conversations/:id/messages/:messageId/reactions` — add reaction
- [ ] `DELETE /api/v1/conversations/:id/messages/:messageId/reactions/:emoji` — remove reaction
- [ ] Publish `reaction.added` / `reaction.removed` events
- [ ] WebSocket: push `REACTION_ADDED` / `REACTION_REMOVED` to conversation subscribers

### Reply Threading (Flat Model)

- [ ] `POST /api/v1/conversations/:id/messages` accepts `replyToMessageId`
- [ ] Messages store `reply_to_message_id` (FK to messages)
- [ ] GET messages returns `replyToMessage` as embedded object (single JOIN, no N+1)
- [ ] WebSocket: `MESSAGE_NEW` includes `replyToMessage` object

---

## Week 15–16: Attachment Pipeline

### Upload Flow

- [ ] `POST /api/v1/file/upload` — multipart form
  - Stream to temporary storage
  - Detect MIME type from content bytes (not header)
  - Validate against allowed MIME allowlist
  - Compute SHA-256 hash
  - Dedup: if hash exists, return existing attachment ID
  - Store with UUID filename in configured storage path
  - Insert `attachments` row with all metadata
  - Publish `attachment.uploaded` event

- [ ] `AttachmentProcessingWorker` (BullMQ):
  - Generate thumbnail for images (sharp library)
  - Generate preview for documents (pdf-parse for page count, first page preview)
  - Update `attachments.preview_url`, `attachments.width`, `attachments.height`
  - Publish `attachment.ready` event

### Download Flow

- [ ] `GET /api/v1/file/:id` — download
  - Verify requester has access to a conversation containing this attachment
  - Return file with correct `Content-Type` header (from stored MIME, not filename)
  - Set `Content-Disposition: attachment; filename="<originalFilename>"`

### Security Checklist

- [ ] MIME from content bytes (file-type library)
- [ ] Stored filename is UUID (never client-provided)
- [ ] Access check before serving (conversation membership)
- [ ] No path traversal possible (absolute path computed server-side)
- [ ] Max upload size enforced at streaming level

---

## Week 16–17: Sync Engine

### Sync Queue Population

- [ ] Every message create, update, delete → inserts `sync_queue` row for each offline device
- [ ] Every reaction add/remove → inserts `sync_queue` row
- [ ] Every delivery status change → inserts `sync_queue` row
- [ ] `sync_queue.sequence` is a monotonic PostgreSQL BIGSERIAL per-user

### SYNC_REQUEST Handling

- [ ] Client sends `SYNC_REQUEST { cursors: { messages: N, reactions: N, ... } }`
- [ ] `SyncService` queries `sync_queue WHERE sequence > cursor AND target_user_id = userId`
- [ ] Results batched into 100-event groups
- [ ] Each batch sent as `SYNC_DELTA` frame
- [ ] After all batches: `SYNC_COMPLETE { newCursors: { ... } }`

### Cursor Persistence

- [ ] `sync_cursors` table updated after SYNC_COMPLETE acknowledged
- [ ] Client stores new cursors in local storage
- [ ] On reconnect: client reads cursors from storage → sends SYNC_REQUEST

### Tests

- [ ] Sync correctly delivers messages sent while client was offline
- [ ] Idempotent: replaying SYNC_REQUEST with same cursor returns same delta
- [ ] Sequence numbers are monotonically increasing per user

---

## Week 17–18: Email Interoperability

### Inbound HMP/UUCP → Chat Message

- [ ] `HmpIngestService`: parses incoming HMP packets (existing format)
- [ ] Find or create conversation for sender callsign
- [ ] Create message in conversation with `source=radio`
- [ ] Insert `message_envelopes` row (transport=`uucp`, raw headers preserved)
- [ ] Publish `message.new` event → normal delivery pipeline handles the rest

### Outbound Chat → UUCP

- [ ] `RadioDeliveryWorker` (BullMQ): triggered when delivery channel=`radio`
- [ ] Packages message content as HMP packet
- [ ] Submits to UUCP queue for target station callsign
- [ ] Updates `message_deliveries.status` to `sent` after UUCP acceptance
- [ ] Marks `delivered` when acknowledgment received

### Legacy API Compatibility Shim

- [ ] `GET /api/v1/message/inbox` → queries conversations for inbound messages → returns in legacy format
- [ ] `POST /api/v1/message/send` → creates message in conversation → normal delivery pipeline
- [ ] Shim marked deprecated in OpenAPI docs
- [ ] Remove date scheduled for Phase 4

---

## Week 18–20: Integration, Testing, and Polish

### Integration Tests

- [ ] Full message flow: send → deliver via WebSocket → read receipt
- [ ] Offline scenario: message sent → client reconnects → sync delivers message
- [ ] Attachment: upload → processing → download with access check
- [ ] Reaction: add → pushed to other participants
- [ ] Edit/delete: message updated → all subscribed clients receive update
- [ ] HMP ingest: fake HMP packet → message appears in conversation

### Performance Tests

- [ ] 50 concurrent WebSocket clients, 10 messages/second → no frame drops
- [ ] Unread count query on 10,000 message conversation: <20ms
- [ ] Sync delta query with 500 queued events: <100ms

---

## Definition of Done — Phase 3

- [ ] Hermes-chat frontend showing conversations, messages, reactions, threading
- [ ] Messages delivered in real-time to online clients via WebSocket
- [ ] Offline clients receive all missed messages on reconnect via sync engine
- [ ] Attachments uploadable, processed, and downloadable with access control
- [ ] HMP/UUCP inbound messages appear as chat messages in conversations
- [ ] All Phase 1 and Phase 2 functionality still passing CI
- [ ] Zero `npm audit` high/critical vulnerabilities
