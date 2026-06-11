# Messaging Domain Model

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Why the Current Model Must Be Replaced

The current `hermes-api` messaging model is email-oriented:

- Messages are inbox/outbox items, not conversation members
- HMP (Hermes Message Pack) is a tar.gz bundle — not a streaming format
- UUCP is the transport — not an abstraction
- There is no conversation entity
- Message status is binary (sent/received) — no delivery tracking per recipient
- No threading, no reactions, no editing, no real-time delivery
- File attachments are handled via a completely separate controller with no message linkage
- The model cannot support multi-device synchronization

The new model makes the **conversation** the central entity, with **messages** as members of conversations, and a separate **delivery tracking** layer for multi-channel, multi-recipient delivery.

---

## 2. Domain Model

```
┌─────────────────────────────────────────────────────────────────┐
│                       Communication Domain                      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      Conversation                        │  │
│  │  type: direct | group | broadcast | radio                │  │
│  │  participants: User[] (with roles)                       │  │
│  │  lastMessageId, lastActivityAt                           │  │
│  └───────────────────────┬──────────────────────────────────┘  │
│                           │ 1:N                                 │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │                       Message                            │  │
│  │  content, contentType                                    │  │
│  │  senderId, conversationId                                │  │
│  │  replyToMessageId (threading)                            │  │
│  │  subject (email compat, optional)                        │  │
│  │  status: draft | sending | sent | failed                 │  │
│  │  clientMessageId (idempotency key)                       │  │
│  └──┬──────────────────┬─────────────────────┬─────────────┘  │
│     │ 1:N              │ 1:N                 │ 0:1            │
│  ┌──▼──────────────┐  ┌▼──────────────────┐  ┌▼─────────────┐  │
│  │ MessageDelivery │  │   Attachment      │  │  Envelope    │  │
│  │ (per recipient) │  │ (file metadata)   │  │ (email compat│  │
│  │ per channel     │  │ storage path      │  │  transport)  │  │
│  │ pending→sent    │  │ checksum, preview │  │              │  │
│  │ →delivered→read │  └───────────────────┘  └──────────────┘  │
│  └─────────────────┘                                           │
│     │ N:N (users ↔ reactions)                                  │
│  ┌──▼──────────────────────────────────────────────────────┐   │
│  │               MessageReaction                           │   │
│  │  userId, emoji, createdAt                               │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Conversation Types

| Type | Description | Participants | Title |
|---|---|---|---|
| `direct` | 1-to-1 chat between two users | Exactly 2 | Not required |
| `group` | N-to-N chat with multiple participants | 3+ | Required |
| `broadcast` | 1-to-many (radio broadcast, system alert) | 1 sender, many recipients | Required |
| `radio` | Conversation linked to a radio channel/frequency | Varies | Optional |

**Key rules:**
- A `direct` conversation is identified by its two participant IDs (ordered) — no duplicates
- A user leaving a `group` conversation is soft-tracked via `conversation_participants.left_at`
- A `broadcast` conversation has recipients who cannot reply
- Conversations are never hard-deleted — only `archived_at` is set

---

## 4. Message Lifecycle

```
Client composes message
  │
  ▼
POST /api/v1/conversations/:id/messages
  │
  ▼ (idempotency check via client_message_id)
message.status = 'sending'
message persisted (created_at = now)
  │
  ▼
MessagingService.publishMessage()
  │
  ├─► DeliveryService.createDeliveries(message, conversation.participants)
  │     └─► Creates message_delivery rows (status=pending) per recipient per channel
  │
  ├─► Durable delivery scheduling from the write path
  │     └─► BullMQ jobs / recovery hooks for offline or delayed delivery
  │
  ├─► EventBus.publish('message.new', { message, conversation })
  │     │
  │     ├─► WebSocket Gateway → MESSAGE_NEW to online recipients
  │     └─► Observability / non-durable fan-out subscribers
  │
  └─► message.status = 'sent'
        │
        ▼
  [Async delivery]
  WebSocket delivery → mark delivered_at
  Push notification → mark delivered_at
  Radio/UUCP (if offline) → queued for next connection window
        │
        ▼
  Recipient opens conversation / sends MESSAGE_ACK
  → mark read_at for that recipient
  → EventBus.publish('message.read', { ... })
  → MESSAGE_READ sent to sender via WebSocket
```

---

## 5. Message Delivery Tracking

Delivery tracking is **per-recipient, per-channel**. This is fundamentally different from the current model.

### Status Machine

```
pending → sent → delivered → read
             └────→ failed (retries exhausted)
```

### Channel Types

| Channel | Trigger | Delivery Confirmation |
|---|---|---|
| `websocket` | Recipient is online (WebSocket connected) | MESSAGE_ACK from client |
| `push` | Recipient has push token (mobile) | FCM/APNs delivery receipt |
| `radio` | Target station is reachable via HF | UUCP acknowledgment |
| `email` | Recipient has email configured | SMTP delivery report |

### Multi-Device Delivery

When a user has multiple devices, a delivery record is created **per device per channel**:

```
message_deliveries:
  message_id=X, recipient_id=user1, channel=websocket, device_id=device-A, status=delivered
  message_id=X, recipient_id=user1, channel=websocket, device_id=device-B, status=pending
  message_id=X, recipient_id=user1, channel=push,      device_id=device-B, status=sent
```

The **aggregate status** shown to the sender is the highest status across all delivery records for that recipient.

---

## 6. Threading & Replies

The model supports **flat threading** (WhatsApp/Telegram style), not nested threads:

- `reply_to_message_id` points to the parent message
- A reply is displayed inline within the conversation, indented
- Replies to replies point to the original message (prevents infinite nesting)
- True thread branching (Discord-style threads) is reserved for a future iteration

```
Conversation:
  Message A (top-level)
  Message B (reply to A) [reply_to_message_id = A.id]
  Message C (top-level)
  Message D (reply to A) [reply_to_message_id = A.id]
```

---

## 7. Email Interoperability Layer

Email compatibility is achieved through the `message_envelopes` table — a separate record that attaches email metadata to a message without polluting the message model.

### Design Principle

The chat UX never knows or cares about email envelopes. Email is a **transport concern**, not a messaging concern.

```
┌──────────────────────────────────────────────────────────┐
│                    Message (chat-native)                  │
│  content, contentType, conversationId, senderId          │
│  status, replyToMessageId, createdAt                     │
└────────────────────────────┬─────────────────────────────┘
                             │ 0..1 (optional envelope)
┌────────────────────────────▼─────────────────────────────┐
│                  MessageEnvelope (transport)              │
│  fromAddress, toAddresses, subject, headers (JSONB)      │
│  transport: smtp | uucp | radio | hmp                    │
│  externalMessageId (Message-ID header)                   │
│  rawMessagePath (original .eml or .hmp file)             │
└──────────────────────────────────────────────────────────┘
```

### Inbound Email → Chat Message Flow

```
Inbound .hmp or .eml file received
  │
  ▼
EmailIngestWorker (BullMQ job)
  ├─ Parse headers (from, to, subject, message-id)
  ├─ Find or create conversation
  │    └─ Match by: sender callsign, subject thread, or explicit address
  ├─ Create message (content = email body, contentType = 'text' or 'html')
  ├─ Create message_envelope (type=inbound, headers, externalMessageId)
  ├─ Create attachments (for MIME parts)
  └─ Publish 'message.new' → real-time delivery to chat clients
```

### Outbound Chat Message → Email Flow (Future)

```
User sends message in conversation
  │
  ▼ (if conversation has email transport configured)
EmailExportWorker (BullMQ job)
  ├─ Read message + conversation
  ├─ Create message_envelope (type=outbound, transport=smtp)
  ├─ Compose .eml from message content + attachments
  ├─ Send via SMTP or UUCP
  └─ Update envelope status=sent
```

---

## 8. Attachment Pipeline

```
Client POSTs file to /api/v1/attachments (multipart/form-data)
  │
  ▼
AttachmentController
  ├─ Stream file to temp storage (never buffer whole file in memory)
  ├─ Validate MIME type (allowlist: image/*, application/pdf, audio/*, video/*)
  ├─ Calculate SHA-256 checksum (stream, not read-all)
  ├─ Check for duplicate (lookup by checksum → reuse if exists)
  ├─ Persist attachment row (status=pending, storage_path=temp)
  └─ Enqueue AttachmentProcessingJob

AttachmentProcessingJob (BullMQ)
  ├─ Move from temp to permanent storage path
  ├─ Generate thumbnail/preview (for images: sharp; for audio: ffmpeg)
  ├─ Update attachment (status=ready, preview_path)
  └─ Emit 'attachment.ready' event (if linked to a message)
```

### Security Controls

- MIME type validated from file content (using `file-type` library), not from client header
- Maximum file size enforced at streaming level (configurable, default 50MB)
- Storage path constructed server-side using UUID — client filename is stored as `original_filename` only
- Downloads served via signed time-limited URLs (not direct filesystem access)
- Attachment access requires membership in the conversation it belongs to

---

## 9. Conversation Queries

The most performance-critical query is conversation list with unread counts and an indexed on-read lookup of the latest message:

```sql
-- Conversations for a user, sorted by activity, with unread counts
SELECT 
  c.id,
  c.type,
  c.title,
  c.last_activity_at,
  lm.content AS last_message_content,
  lm.created_at AS last_message_at,
  u.callsign AS last_sender_callsign,
  COUNT(msgs.id) FILTER (
    WHERE msgs.created_at > cp.last_read_at
    AND msgs.sender_id != :userId
    AND msgs.deleted_at IS NULL
  ) AS unread_count
FROM conversation_participants cp
JOIN conversations c ON c.id = cp.conversation_id
LEFT JOIN LATERAL (
  SELECT id, sender_id, content, created_at
  FROM messages
  WHERE conversation_id = c.id
    AND deleted_at IS NULL
  ORDER BY created_at DESC
  LIMIT 1
) lm ON TRUE
LEFT JOIN users u ON u.id = lm.sender_id
LEFT JOIN messages msgs ON msgs.conversation_id = c.id
WHERE cp.user_id = :userId
  AND cp.left_at IS NULL
  AND (c.archived_at IS NULL OR :includeArchived = true)
GROUP BY c.id, cp.last_read_at, lm.content, lm.created_at, u.callsign
ORDER BY c.last_activity_at DESC NULLS LAST
LIMIT 50;
```

This query uses the `idx_conv_participants_active` partial index and `idx_conversations_last_activity` index.

---

## 10. MessagingService API (Internal)

```typescript
// src/messaging/MessagingService.ts

interface MessagingService {
  // Conversations
  createDirectConversation(userId1: string, userId2: string): Promise<Conversation>
  createGroupConversation(creatorId: string, title: string, participantIds: string[]): Promise<Conversation>
  getConversationsForUser(userId: string, options: PaginationOptions): Promise<PaginatedResult<Conversation>>
  getConversation(id: string, requesterId: string): Promise<Conversation>
  archiveConversation(id: string, requesterId: string): Promise<void>

  // Messages
  sendMessage(params: SendMessageParams): Promise<Message>
  editMessage(id: string, content: string, senderId: string): Promise<Message>
  deleteMessage(id: string, senderId: string): Promise<void>
  getMessages(conversationId: string, options: MessageQueryOptions): Promise<PaginatedResult<Message>>

  // Delivery
  acknowledgeDelivery(messageId: string, recipientId: string, deviceId: string): Promise<void>
  markRead(conversationId: string, userId: string, upToMessageId: string): Promise<void>

  // Reactions
  addReaction(messageId: string, userId: string, emoji: string): Promise<MessageReaction>
  removeReaction(messageId: string, userId: string, emoji: string): Promise<void>

  // Participants
  addParticipant(conversationId: string, userId: string, addedBy: string): Promise<void>
  removeParticipant(conversationId: string, userId: string, removedBy: string): Promise<void>
}
```

---

## 11. Comparison: Old Model vs New Model

| Feature | Old (hermes-api) | New (poc-hermes-server) |
|---|---|---|
| Message container | inbox / outbox | conversation |
| Message format | HMP (.tar.gz) | JSON + optional envelope |
| Transport | UUCP only | websocket, push, radio, email |
| Delivery tracking | none | per-recipient, per-channel |
| Threading | none | flat reply chains |
| Reactions | none | emoji reactions |
| Message editing | none | edit with edited_at timestamp |
| Message deletion | hard delete | soft delete |
| Attachments | separate FileController | linked to messages |
| Real-time | none | WebSocket events |
| Multi-device | none | sync cursor per device |
| Email compat | native (only) | optional envelope abstraction |
| Group messages | none | group conversations |
| Offline support | none | sync queue + cursors |
