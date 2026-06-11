# Phase 3 — Chat-Native Messaging & Synchronization Task List

**Project**: poc-hermes-server  
**Phase Duration**: Weeks 13–20  
**Prerequisites**: Phase 2 complete — M-2 gate passed  
**Generated**: 2026-06-10  
**Master Plan Reference**: [00-master-project-plan.md](../plan/00-master-project-plan.md)

---

## Phase Objectives (from roadmap)

1. Conversations, messages, reactions, and delivery tracking fully operational
2. WebSocket push delivery of new messages to online clients
3. Offline sync engine with cursor-based reconnect — correctness guaranteed
4. Attachment pipeline with MIME validation, dedup, and signed URLs
5. HMP/UUCP email interoperability preserved via `message_envelopes`
6. Message editing, deletion, and reply threading
7. Typing indicators

## Audit Findings Addressed This Phase

- **[AUDIT C-2]** `sync_queue` stores entity references (not payload snapshots) — current state fetched at sync time
- **[AUDIT C-3]** Sync cursor advanced for both WebSocket push delivery AND sync queue replay — eliminates duplicates on reconnect
- **[AUDIT M-1]** Payload duplication resolved by C-2 fix
- **[AUDIT M-2]** Inactive device cleanup — stop inserting sync_queue rows for devices not seen in 90 days
- **[AUDIT L-3]** Storage-agnostic language in docs; platform-specific notes added

---

## Week 13: Messaging Domain Model (EP-12)

### [ ] Task 3.1.1: ConversationRepository — full implementation
**Purpose**: Provide the typed database interface for all conversation operations.  
**Description**: Create `src/modules/messaging/repositories/ConversationRepository.ts`. Implement:
- `create(type, participantIds, metadata)` → creates conversation + participant rows atomically
- `getById(id, userId)` → verifies calling user is a participant (enforces access control)
- `listForUser(userId, cursor, limit)` → paginated, ordered by `last_activity_at DESC`; **[AUDIT H-1]**: computes `lastMessageId` via `SELECT id FROM messages WHERE conversation_id = $1 ORDER BY created_at DESC LIMIT 1` with index `idx_messages_conversation_created` — NOT from a denormalized column
- `archive(id, userId)` → sets `archived_at`
- `addParticipant(id, userId)` → inserts `conversation_participants`
- `removeParticipant(id, userId)` → sets `left_at` (soft remove)
- `getUnreadCount(conversationId, userId)` → counts messages after `last_read_message_id`

**Acceptance Criteria**:
- [ ] `getById` returns `null` if calling user is not a participant (no 500 error, no data leak)
- [ ] `listForUser` query uses `idx_messages_conversation_created` index — verified via `EXPLAIN ANALYZE`
- [ ] `create` with duplicate `direct` conversation (same two participants) returns existing conversation (idempotent)
- [ ] Unit tests cover all methods with mocked Drizzle `db`

**Files to Create/Edit**:
- `src/modules/messaging/repositories/ConversationRepository.ts`
- `src/modules/messaging/repositories/ConversationRepository.test.ts`

**Doc Reference**: `docs/architecture/messaging-domain.md §4`, `docs/roadmap/phase-3-messaging.md §Week 13`, Audit H-1  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Conversation list is the main view — sort order and unread count must be correct  
**Blocks**: 3.1.3 (REST API), 3.2.x (delivery)

---

### [ ] Task 3.1.2: MessageRepository and DeliveryRepository — full implementation
**Purpose**: Typed database interface for messages and per-recipient delivery tracking.  
**Description**: Create `src/modules/messaging/repositories/MessageRepository.ts`:
- `create(conversationId, senderId, content, clientMessageId)` → **idempotent**: if `client_message_id` exists, return existing message
- `getById(id, userId)` → access check: user must be a conversation participant
- `listByConversation(conversationId, cursor, limit)` → cursor pagination by `created_at`
- `update(id, userId, content)` → edit (sender only, within edit time window)
- `softDelete(id, userId)` → sets `deleted_at`

Create `src/modules/messaging/repositories/DeliveryRepository.ts`:
- `createBatch(messageId, recipientIds, deviceIds, channels)` → one row per recipient per device per channel
- `markDelivered(messageId, recipientId, channel, deviceId)` → status transition `pending → delivered`
- `markRead(messageId, recipientId)` → status transition `delivered → read`
- `getPendingOlderThan(seconds)` → used by C-1 scanner (see Phase 2 Task 2.5.3)

**Acceptance Criteria**:
- [ ] `create` with duplicate `clientMessageId` returns existing message, not a new row
- [ ] `update` by non-sender returns error (not 500)
- [ ] `update` after edit window (configurable, default 24 h) returns `EDIT_WINDOW_EXPIRED` error
- [ ] `createBatch` produces exactly `len(recipientIds) × len(deviceIds per recipient) × len(channels)` rows
- [ ] Unit tests for idempotency and access control

**Files to Create/Edit**:
- `src/modules/messaging/repositories/MessageRepository.ts`
- `src/modules/messaging/repositories/DeliveryRepository.ts`

**Doc Reference**: `docs/architecture/messaging-domain.md §5`, `docs/roadmap/phase-3-messaging.md §Week 13`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Message idempotency prevents duplicates when poc-hermes-app retries on network error  
**Blocks**: 3.1.3, 3.2.x

---

### [ ] Task 3.1.3: REST API — Conversations and Messages
**Purpose**: Expose full chat messaging via HTTP.  
**Description**: Implement all conversation and message routes from `docs/roadmap/phase-3-messaging.md §Week 13`:

**Conversations**:
- `POST /api/v1/conversations` (create)
- `GET /api/v1/conversations` (list for user, cursor paginated)
- `GET /api/v1/conversations/:id` (get + participants)
- `POST /api/v1/conversations/:id/participants` (add — operator/admin)
- `DELETE /api/v1/conversations/:id/participants/:userId` (remove)

**Messages**:
- `POST /api/v1/conversations/:id/messages` (send — idempotent via `clientMessageId`)
- `GET /api/v1/conversations/:id/messages` (list, cursor paginated)
- `PATCH /api/v1/conversations/:id/messages/:messageId` (edit)
- `DELETE /api/v1/conversations/:id/messages/:messageId` (soft delete)

All routes: JSON Schema, OpenAPI annotation, RBAC.

**Acceptance Criteria**:
- [ ] `POST /conversations/:id/messages` twice with same `clientMessageId` returns same message ID both times
- [ ] `GET /conversations` returns correct `unreadCount` per conversation
- [ ] `PATCH /:id/messages/:msgId` by non-sender returns `403`
- [ ] Integration test: create conversation → send message → list messages → verify

**Files to Create/Edit**:
- `src/api/routes/conversations.ts`
- `src/api/routes/messages.ts`
- `src/modules/messaging/MessagingService.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 13`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: These are the primary endpoints the new hermes-chat frontend uses; must be consistent with Phase 1 shim behavior  
**Blocks**: 3.2.x (delivery pipeline)

---

## Week 14: Delivery Pipeline (EP-13)

### [ ] Task 3.2.1: Online delivery — WebSocket push with fail-fast fallback
**Purpose**: Deliver messages to online clients in real-time.  
**Description**: In `MessagingService.sendMessage()`: after DB write, publish `message.new` to IEventBus. WebSocket gateway subscribes (fan-out) to `message.*`. For each participant with an active WebSocket connection subscribed to `conversation.<id>`: push `MESSAGE_NEW` frame. **[AUDIT H-2]**: On `ws.send()` error callback → IMMEDIATELY trigger offline queue insert for that device. On send success: start 30-second ACK timer; if `MESSAGE_ACK` not received within 30 s → insert sync_queue row for that device. On `MESSAGE_ACK`: call `DeliveryRepository.markDelivered()` for WebSocket channel; update sync cursor for that device. **[AUDIT C-3]**: Advance `sync_cursors` for the device after successful push.

**Acceptance Criteria**:
- [ ] Online client receives `MESSAGE_NEW` within 500 ms of `POST /conversations/:id/messages`
- [ ] `MESSAGE_ACK` received → `message_deliveries.status = 'delivered'`
- [ ] Test: disconnect client socket mid-send → offline fallback triggered immediately (no 30-second wait)
- [ ] Sync cursor updated after push (verified in `sync_cursors` table)

**Files to Create/Edit**:
- `src/gateway/handlers/messageDelivery.ts`
- `src/modules/messaging/MessagingService.ts` (update)

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 14`, Audit C-3, Audit H-2  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: poc-hermes-app must send `MESSAGE_ACK` to confirm delivery; must handle `MESSAGE_NEW` frame  
**Blocks**: 3.2.2

---

### [ ] Task 3.2.2: Offline delivery — sync queue with entity-reference model
**Purpose**: Durably queue messages for offline clients using entity references (not payload snapshots).  
**Description**: Create `src/modules/sync/SyncQueueService.ts`. `enqueue(targetUserId, targetDeviceId, entityType, entityId, operation)`: insert `sync_queue` row with `entity_id` and `entity_type` only — **[AUDIT C-2]**: NO `payload` column. `SyncService.sendDelta(userId, deviceId, cursors)`: query `sync_queue WHERE sequence > cursor`; for each row, fetch current entity state (e.g., `MessageRepository.getById()`) at sync time; batch into 100-event `SYNC_DELTA` frames. Update `sync_cursors` after `SYNC_COMPLETE` acknowledged. **[AUDIT C-3]**: `sync_cursors` updated for both online push AND sync replay paths — single source of truth.

**Acceptance Criteria**:
- [ ] `sync_queue` rows contain only `entity_id`, `entity_type`, `operation`, `sequence`, `target_user_id` — no `payload` column
- [ ] Editing a message after it is queued → offline client receives CURRENT content (not pre-edit) on reconnect
- [ ] Integration test: send message → disconnect client → reconnect → SYNC_REQUEST → SYNC_DELTA contains correct current content → no duplicates
- [ ] Replaying SYNC_REQUEST with same cursor returns identical delta (idempotent)

**Files to Create/Edit**:
- `src/modules/sync/SyncQueueService.ts`
- `src/modules/sync/SyncService.ts`
- `src/gateway/handlers/syncRequest.ts`
- `src/modules/sync/SyncService.test.ts`

**Doc Reference**: `docs/architecture/offline-first.md §3, §6`, Audit C-2, Audit C-3  
**Estimated Complexity**: Very High  
**poc-hermes-app Integration Impact**: poc-hermes-app must handle `SYNC_DELTA` and `SYNC_COMPLETE` frames; cursor persistence on client side  
**Blocks**: 3.5.x (HMP interop), M-3 gate

---

### [ ] Task 3.2.3: Read status — MESSAGE_READ propagation
**Purpose**: Show senders when recipients have read their messages.  
**Description**: Handle `MESSAGE_READ { conversationId, messageId }` WebSocket frame from client. Update `conversation_participants.last_read_message_id` and `last_read_at`. Publish `message.read` event to IEventBus. Gateway fans out `MESSAGE_READ` frame to sender's active connections. `GET /conversations` response includes correct `unreadCount` derived from `last_read_message_id`.

**Acceptance Criteria**:
- [ ] Sender receives `MESSAGE_READ` frame within 500 ms of recipient opening the conversation
- [ ] `GET /conversations` unreadCount drops to 0 after recipient reads messages
- [ ] Read receipt NOT sent to the reading user themselves

**Files to Create/Edit**:
- `src/gateway/handlers/messageRead.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 14`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: poc-hermes-app read-receipt indicator depends on this  
**Blocks**: None

---

## Week 15: Reactions, Threading, Typing Indicators (EP-17)

### [ ] Task 3.3.1: Message reactions
**Purpose**: Allow emoji reactions to messages, consistent with modern chat UX.  
**Description**: Implement `src/modules/messaging/repositories/ReactionRepository.ts`. REST endpoints: `POST /api/v1/conversations/:id/messages/:messageId/reactions` (add), `DELETE /api/v1/conversations/:id/messages/:messageId/reactions/:emoji` (remove). Unique constraint: one reaction per user per emoji per message. On add: publish `reaction.added` → gateway fans out `REACTION_ADDED` to conversation subscribers. On remove: `REACTION_REMOVED`.

**Acceptance Criteria**:
- [ ] Duplicate reaction (same user, same emoji, same message) returns `409 CONFLICT`
- [ ] Online conversation participants receive `REACTION_ADDED` within 500 ms
- [ ] `GET /conversations/:id/messages` includes `reactions: [{ emoji, count, userIds }]` in response

**Files to Create/Edit**:
- `src/modules/messaging/repositories/ReactionRepository.ts`
- `src/api/routes/reactions.ts`
- `src/gateway/handlers/reactions.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 15`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: hermes-chat frontend reaction UI depends on this  
**Blocks**: None

---

### [ ] Task 3.3.2: Reply threading
**Purpose**: Allow messages to be sent as replies to existing messages.  
**Description**: `POST /conversations/:id/messages` body accepts `replyToMessageId` (optional). `messages.reply_to_message_id` FK enforced. `GET /conversations/:id/messages` returns `replyToMessage` as embedded object via single JOIN — no N+1 queries. `MESSAGE_NEW` WebSocket frame includes `replyToMessage` object. Validate: `replyToMessageId` must be in the same conversation.

**Acceptance Criteria**:
- [ ] `replyToMessageId` pointing to a message in a different conversation returns `400`
- [ ] `GET /messages` with threading uses a single JOIN query (verified via `EXPLAIN` in test)
- [ ] `MESSAGE_NEW` frame includes `replyToMessage: { id, senderId, content }` when present

**Files to Create/Edit**:
- `src/api/routes/messages.ts` (update)

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 15`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: hermes-chat reply UI depends on this  
**Blocks**: None

---

### [ ] Task 3.3.3: Typing indicators
**Purpose**: Show real-time typing state for a more responsive chat experience.  
**Description**: Handle `TYPING_START { conversationId }` and `TYPING_STOP { conversationId }` WebSocket frames. On receive: validate user is participant. Publish `typing.started`/`typing.stopped` on IEventBus. Gateway fans out to conversation subscribers (excluding sender). Server-side auto-stop: if no `TYPING_START` received from a user within 5 seconds, publish `typing.stopped`.

**Acceptance Criteria**:
- [ ] Non-participant sending `TYPING_START` receives `ERROR { code: 'FORBIDDEN' }`
- [ ] `TYPING_START` from user A → other participants receive `TYPING_START { userId, conversationId }`
- [ ] No `TYPING_START` for 5 s → `typing.stopped` published automatically
- [ ] Sender does NOT receive their own typing indicator back

**Files to Create/Edit**:
- `src/gateway/handlers/typing.ts`
- `src/modules/messaging/TypingStateTracker.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 14`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: hermes-chat typing indicator depends on this  
**Blocks**: None

---

## Week 15–16: Attachment Pipeline (EP-16)

### [ ] Task 3.4.1: Full attachment upload pipeline
**Purpose**: Allow file attachments to be sent in conversations with security and deduplication.  
**Description**: Enhance `POST /api/v1/file/upload` (scaffolded in Phase 1 Task 1.5.3). Full pipeline: stream to temp → detect MIME from content bytes (`file-type`) → validate against allowlist → compute SHA-256 → dedup check → store via `IStorageBackend.write()` (UUID key) → insert `attachments` row → enqueue `AttachmentProcessingWorker` job → publish `attachment.uploaded` event.

Create `src/jobs/AttachmentProcessingWorker.ts`. For images: generate thumbnail via `sharp`. For PDFs: extract page count + first-page preview via `pdf-parse`. Update `attachments.preview_url`, `width`, `height`. Publish `attachment.ready` event.

**Acceptance Criteria**:
- [ ] Upload of `.exe` file with spoofed `Content-Type: image/jpeg` is rejected (MIME from bytes, not header)
- [ ] Second upload of identical file (same SHA-256) returns existing attachment ID — no duplicate file stored
- [ ] Image upload → thumbnail generated within 10 s → `attachment.ready` event published
- [ ] All acceptance criteria from Phase 1 Task 1.5.3 still pass

**Files to Create/Edit**:
- `src/jobs/AttachmentProcessingWorker.ts`
- `src/api/routes/files.ts` (enhance)

**Stack Notes**: `sharp` for image thumbnails; `file-type` for MIME detection; `IStorageBackend` injected (not hardcoded)  
**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 15–16`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Attachment download must continue working; thumbnail URLs must be reachable  
**Blocks**: None

---

## Week 16–17: Sync Engine (EP-15)

### [ ] Task 3.5.1: Sync queue population for all entity types
**Purpose**: Populate the sync queue for every mutation that offline clients need to catch up on.  
**Description**: Wire `SyncQueueService.enqueue()` to be called for every event type that offline clients care about. Events → sync queue inserts:
- `message.new` → `{ entity_type: 'message', entity_id: messageId, operation: 'create' }`
- `message.updated` → `{ entity_type: 'message', entity_id: messageId, operation: 'update' }`
- `message.deleted` → `{ entity_type: 'message', entity_id: messageId, operation: 'delete' }`
- `reaction.added` / `reaction.removed` → `{ entity_type: 'reaction', entity_id: reactionId, operation: 'create'/'delete' }`
- `delivery.updated` → `{ entity_type: 'delivery', entity_id: deliveryId, operation: 'update' }`

Each insert targets offline devices only — skip devices with active WebSocket connection (since cursor is advanced on push).

**Acceptance Criteria**:
- [ ] Sending a message produces sync_queue rows for all offline devices of recipients
- [ ] Editing a message produces sync_queue rows for affected offline devices
- [ ] Devices with active WebSocket connections do NOT get sync_queue rows (cursor was already advanced)
- [ ] `sync_queue.sequence` is monotonically increasing per user (PostgreSQL BIGSERIAL)

**Files to Create/Edit**:
- `src/modules/sync/SyncQueueService.ts` (update)

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 16–17`, Audit C-2, Audit C-3  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: poc-hermes-app offline sync depends on this being populated correctly  
**Blocks**: 3.5.2

---

### [ ] Task 3.5.2: SYNC_REQUEST handling and cursor management
**Purpose**: Deliver all missed events to reconnecting clients efficiently.  
**Description**: Handle `SYNC_REQUEST { cursors: { messages: N, reactions: N, deliveries: N, conversations: N } }`. Query `sync_queue WHERE sequence > cursor AND target_user_id = userId`. For each row: fetch current entity via repository (current state — not stale snapshot). Batch into 100-event `SYNC_DELTA` frames. After all batches: send `SYNC_COMPLETE { newCursors }`. Update `sync_cursors` table. **[AUDIT C-3]**: Single cursor table is the source of truth for both WebSocket push delivery and sync replay.

**Acceptance Criteria**:
- [ ] Offline client with cursor=0 receives all missed messages on reconnect (in order, no duplicates)
- [ ] Edited message sync delivers current (post-edit) content, not pre-edit snapshot
- [ ] SYNC_REQUEST with cursor at current head returns empty `SYNC_DELTA` immediately followed by `SYNC_COMPLETE`
- [ ] Performance test: 500 queued events → all delivered in `SYNC_DELTA` batches within 100 ms

**Files to Create/Edit**:
- `src/modules/sync/SyncService.ts` (update)
- `src/gateway/handlers/syncRequest.ts` (update)
- `src/modules/sync/SyncService.test.ts` (update)

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 16–17`, Audit C-2, Audit C-3  
**Estimated Complexity**: Very High  
**poc-hermes-app Integration Impact**: poc-hermes-app offline recovery depends on this being correct and performant  
**Blocks**: M-3 gate

---

## Week 17–18: HMP/UUCP Interoperability (EP-18)

### [ ] Task 3.6.1: HMP ingest service — radio message to conversation
**Purpose**: Preserve compatibility with HF radio message exchange (existing HMP/UUCP format).  
**Description**: Create `src/modules/hmp/HmpIngestService.ts`. Parses incoming HMP packets (existing format). Find or create `direct` conversation for sender callsign → target callsign. Create message in conversation with `source='radio'`. Insert `message_envelopes` row (`transport='uucp'`, raw headers preserved). Publish `message.new` → normal delivery pipeline handles the rest.

**Acceptance Criteria**:
- [ ] HMP packet from callsign PY2XA to PY8EME creates a direct conversation between those users
- [ ] Message appears in conversation list for both users
- [ ] `message_envelopes` row contains the raw HMP headers
- [ ] `source='radio'` visible in message metadata (for UI display differentiation)
- [ ] Integration test: inject mock HMP packet → verify message in conversation

**Files to Create/Edit**:
- `src/modules/hmp/HmpIngestService.ts`
- `src/modules/hmp/HmpIngestService.test.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 17–18`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: poc-hermes-app radio message inbox depends on this; callsign-to-user mapping must be consistent  
**Blocks**: None

---

### [ ] Task 3.6.2: Radio delivery worker — chat to UUCP
**Purpose**: Send chat messages to offline HF-only stations via UUCP.  
**Description**: Create `src/jobs/RadioDeliveryWorker.ts` (BullMQ). Triggered when `message_delivery.channel='radio'`. Packages message content as HMP packet. Submits to UUCP queue for target station callsign. Updates `message_deliveries.status='sent'` after UUCP acceptance. Marks `delivered` when UUCP acknowledgment received.

**Acceptance Criteria**:
- [ ] Message to a `connectivity_type='radio'` station triggers `RadioDeliveryWorker`
- [ ] Worker updates delivery status from `pending` → `sent` after UUCP submission
- [ ] Worker retries on UUCP failure (max 5 attempts, exponential backoff)
- [ ] Failed delivery after max retries updates status to `failed` and logs `error`

**Files to Create/Edit**:
- `src/jobs/RadioDeliveryWorker.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 17–18`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: Messages to HF-only stations visible in poc-hermes-app outbox shim  
**Blocks**: None

---

### [ ] Task 3.6.3: Update legacy messaging shim to use chat model
**Purpose**: Keep poc-hermes-app inbox/outbox working while redirecting to the new data model.  
**Description**: Update `src/api/routes/messagingShim.ts` (Phase 1 Task 1.5.5). `GET /message/inbox` → queries `messages` + `conversations` + `message_envelopes` for inbound messages; returns in legacy `{ from, to, subject, body, date, id }` format. Ensure HMP-ingested messages appear correctly in inbox. `POST /message/send` → routes through full `MessagingService` pipeline (not a shortcut).

**Acceptance Criteria**:
- [ ] poc-hermes-app inbox shows HMP-received messages with correct `from` callsign
- [ ] `POST /message/send` via shim produces a fully tracked message in the new model
- [ ] OpenAPI deprecation notice on shim endpoints references Phase 4 removal date

**Files to Create/Edit**:
- `src/api/routes/messagingShim.ts` (update)

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 17–18`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: **Critical** — backward compatibility bridge for all message-related poc-hermes-app screens  
**Blocks**: M-3 gate

---

## Week 18–20: Integration Testing, Performance, and Polish

### [ ] Task 3.7.1: Full messaging integration test suite
**Purpose**: Validate the complete end-to-end messaging flow before the Phase 3 gate.  
**Description**: Write integration tests covering all flows from `docs/roadmap/phase-3-messaging.md §Week 18–20`:
- Full message flow: send → deliver via WebSocket → MESSAGE_ACK → delivery status `delivered`
- Offline scenario: send → disconnect client → edit message → reconnect → SYNC_REQUEST → SYNC_DELTA delivers CURRENT content (not pre-edit)
- Attachment: upload → processing → download with access check
- Reaction: add → pushed to all conversation participants
- Edit/delete: MESSAGE_UPDATED / MESSAGE_DELETED pushed to all subscribers
- HMP ingest: mock HMP packet → message appears in conversation
- Read receipt: recipient reads → MESSAGE_READ sent to sender
- Cursor correctness: online push advances cursor; reconnect uses that cursor; no duplicates

**Acceptance Criteria**:
- [ ] All listed scenarios have passing integration tests
- [ ] No scenario produces duplicate messages
- [ ] Edited message content is always current (never stale) across all delivery paths
- [ ] Integration test suite runs in CI (Docker Compose services)

**Files to Create/Edit**:
- `src/__tests__/phase3-integration.test.ts`

**Doc Reference**: `docs/roadmap/phase-3-messaging.md §Week 18–20`  
**Estimated Complexity**: High  
**poc-hermes-app Integration Impact**: These tests validate that poc-hermes-app UX scenarios work end-to-end  
**Blocks**: M-3 gate

---

### [ ] Task 3.7.2: Performance validation
**Purpose**: Verify system meets NFR performance targets before the phase gate.  
**Description**: Run performance tests using k6 / autocannon against the Phase 3 server:
- 50 concurrent WebSocket clients, 10 messages/s: verify 0 frame drops
- Sync delta: 500 queued events → < 100 ms delivery
- Unread count query on 10,000-message conversation: < 20 ms (verified via `EXPLAIN ANALYZE`)
- Attachment upload throughput at expected file sizes

Produce performance baseline report at `docs/qa/performance-phase-3.md`.

**Acceptance Criteria**:
- [ ] All NFR-P targets met (see `docs/plan/00-master-project-plan.md §3.1`)
- [ ] Performance report shows `idx_messages_conversation_created` index is used for unread count query
- [ ] No event loop lag spikes > 100 ms during load test

**Files to Create/Edit**:
- `docs/qa/performance-phase-3.md`
- `tests/performance/phase3.k6.js`

**Doc Reference**: `docs/plan/00-master-project-plan.md §3.1`  
**Estimated Complexity**: Medium  
**poc-hermes-app Integration Impact**: Performance baselines protect poc-hermes-app users from degraded response times  
**Blocks**: M-3 gate

---

## Phase 3 Compatibility Verification Gate — Checkpoint

### [ ] Task 3.8.1: Run poc-hermes-app against Phase 3 server
**Purpose**: Verify zero regressions across all functionality.  
**Description**: Deploy Phase 3 server. Run full Phase 1 + Phase 2 smoke test. Additionally test: legacy shim inbox/outbox works with new chat model, HMP-ingested messages appear correctly, offline reconnect scenario, attachment download. Produce compatibility report at `docs/qa/compat-phase-3.md`.  
**Acceptance Criteria**:
- [ ] All Phase 1 and Phase 2 smoke tests still pass
- [ ] poc-hermes-app inbox shows all messages (sent via shim, via REST, via HMP)
- [ ] Attachment download works from poc-hermes-app
- [ ] Offline reconnect scenario tested and verified (no duplicates)
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] Zero `tsc --noEmit` errors
- [ ] All Phase 3 audit findings marked resolved
- [ ] Compatibility report signed off by Engineering Lead and Radio Operator SME

**Files to Create/Edit**:
- `docs/qa/compat-phase-3.md`

**Blocks**: Phase 4 start

---

## Phase 3 Quality Gates

- [ ] `npm run build` passes with zero TypeScript errors
- [ ] `npm test` passes with ≥ 80% coverage on all new modules
- [ ] `npm audit --audit-level=high` returns zero findings
- [ ] Messaging APIs, WebSocket events, and compatibility shims verified against `poc-hermes-app` and representative chat clients
- [ ] Messages delivered in real-time to online clients via WebSocket
- [ ] Offline clients receive all missed messages (in current state) on reconnect via sync engine
- [ ] Attachments uploadable, processed, and downloadable with access control
- [ ] HMP/UUCP inbound messages appear as chat messages in conversations
- [ ] All Phase 1 and Phase 2 functionality still passing CI
- [ ] poc-hermes-app compatibility report approved
