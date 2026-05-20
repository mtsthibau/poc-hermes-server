# ADR-005: Chat-Native Messaging Model

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [messaging-domain.md](../architecture/messaging-domain.md)

---

## Context

The existing Hermes API treats messages as **email packets**. The schema and API surface reflect this:

- Messages have `inbox` and `outbox` folders
- Messages are fetched by "type" (HMP, UUCP)
- There is no concept of a conversation or thread
- Reading a message is navigating to `/message/inbox/{id}`
- Sending a message is submitting to `/message/outbox`
- Group communication requires sending individual messages to each recipient

Meanwhile, the primary use cases for HERMES in rural and emergency deployments are:

- Operators communicating in real-time during coordination
- Field teams sending status updates to a base station
- Groups sharing location and situation reports
- Text-based coordination similar to Signal, WhatsApp, or Zello

The email mental model creates unnecessary cognitive load and technical constraints. Users do not think in terms of inbox/outbox — they think in terms of conversations.

---

## Decision

Adopt a **chat-native messaging model** where:

- The central entity is `conversation` (not inbox/outbox)
- Messages belong to conversations
- Conversations have participants
- Delivery is tracked per recipient, per channel, per device
- Email/radio transport is an implementation detail of the delivery layer

The email transport (UUCP, HMP, SMTP) is preserved but abstracted into the `message_envelopes` table — a separate transport concern that does not pollute the messaging domain.

---

## Rationale

### Why Chat-Native Beats Email-Native

| Requirement | Email model | Chat model |
|---|---|---|
| Group communication | Multiple individual messages | One conversation, N participants |
| Real-time delivery | Not modeled | WebSocket push |
| Delivery tracking | Not available | Per-recipient, per-channel, per-device |
| Reply threading | Not available | `reply_to_message_id` |
| Message reactions | Not available | `message_reactions` table |
| Unread count | Full table scan | `last_read_message_id` partial index |
| Offline sync | Not available | Sync cursor + sync queue |
| Message editing | Not available | `updated_at` + `is_edited` flag |
| Soft delete | Not available | `deleted_at` soft delete |

### Why Email Transport Is Preserved (As a Delivery Channel)

Email/radio transport (UUCP/HMP/SMTP) remains critical for:

- Stations with no direct internet connection
- Scheduled HF radio contacts (UUCP)
- Legacy HMP compatibility with other Hermes stations

The key architectural decision: **transport is separate from messaging**. A message in a conversation can be delivered via:

- WebSocket (real-time, primary)
- Push notification (mobile, secondary)
- HMP/UUCP radio packet (radio delivery, fallback)
- SMTP email (email gateway, future)

The `message_deliveries` table tracks delivery per recipient per channel. The `message_envelopes` table records the email/radio envelope when that transport is used.

This means: when station A sends a message to station B, the message exists in a conversation (chat model), and is also scheduled for UUCP delivery (radio delivery channel). Both views are consistent without duplicating the message.

### Conversation Types

```typescript
type ConversationType =
  | 'direct'      // 1-to-1 between two users
  | 'group'       // N users, named group
  | 'broadcast'   // 1-to-many (no replies, announcement style)
  | 'radio'       // Linked to a radio frequency/schedule
```

The `radio` conversation type enables logging all communications on a specific frequency as a conversation, with the delivery channel being the radio transport.

---

## Consequences

### Positive

- Frontend hermes-chat gains full chat UX (reactions, threads, unread, real-time)
- Delivery tracking enables "delivered/read receipts" — critical for emergency coordination
- Offline sync becomes tractable via conversation+message cursors
- Email interop is preserved without polluting the messaging domain
- WebRTC audio sessions can be linked to a conversation

### Negative

- Existing hermes-api email-style API must be maintained for compatibility during transition
- Client migration from inbox/outbox mental model requires UI changes
- The `message_envelopes` table adds complexity for the radio delivery path

### Mitigations

- Legacy API endpoints kept under `/api/v1/message` with compatibility shim
- Compatibility shim translates inbox/outbox queries to conversation queries internally
- Client migration is UI-driven, not API-forced — old clients continue working

---

## Alternatives Considered

- **Extend the email model**: Add threading, real-time, and delivery tracking to the current model. Rejected — the email model's inbox/outbox schema cannot support group conversations or per-recipient delivery tracking without a full redesign anyway. Better to redesign cleanly.
- **Separate chat and email subsystems**: Maintain both independently. Rejected — creates data duplication and synchronization complexity between two systems representing the same communication.
- **XMPP protocol**: Standards-based chat protocol. Considered but rejected — XMPP's complexity (stanzas, extensions, federation protocol) exceeds current needs; custom model gives full control over transport and delivery semantics.
- **Matrix protocol**: Federated, E2EE capable. Excellent but significant operational complexity. Noted as a long-term consideration for Phase 4+ federation.

---

## Related Documents

- [Messaging Domain Architecture](../architecture/messaging-domain.md)
- [Database Schema](../architecture/database-schema.md)
- [Offline-First Architecture](../architecture/offline-first.md)
