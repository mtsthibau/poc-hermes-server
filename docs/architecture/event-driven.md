# Event-Driven Backend Architecture

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Rationale

A radio communication platform has inherently event-driven behavior:

- Radio state changes (frequency, mode, PTT, SNR) happen asynchronously
- Messages arrive from remote stations asynchronously
- Multiple clients subscribe to real-time updates
- Background jobs process delivery queues independently
- Synchronization needs to fan out to multiple subscribers

A pure request-response architecture cannot serve these requirements without polling, which is wasteful and adds latency. The event-driven approach decouples producers (HAL, messaging service) from consumers (WebSocket gateway, delivery workers, sync engine) without direct dependencies.

---

## 2. Event Bus Design

The internal event bus is implemented using **Redis Pub/Sub**. This choice was made because:

- Redis is already in the stack (caching, sessions, BullMQ)
- Redis Pub/Sub provides immediate fanout to all subscribers
- Simpler operational footprint than adding NATS or Kafka
- Sufficient for single-station and small-cluster deployments
- Easy upgrade path to NATS JetStream when federation/persistence is needed

**Important**: Redis Pub/Sub is fire-and-forget — if a subscriber is not connected when an event is published, it misses the event. This is handled by:

1. The sync engine writing durable state to PostgreSQL before publishing
2. Offline clients recovering via the sync cursor mechanism on reconnect
3. BullMQ (persistent) for delivery jobs that must not be lost

### Event Bus API

```typescript
// src/events/EventBus.ts

export class EventBus {
  private publisher: Redis
  private subscriber: Redis

  async publish<T>(topic: string, payload: T): Promise<void> {
    const envelope: EventEnvelope<T> = {
      id: generateId(),
      topic,
      payload,
      timestamp: new Date().toISOString(),
      version: 1,
    }
    await this.publisher.publish(topic, JSON.stringify(envelope))
  }

  subscribe<T>(topic: string, handler: EventHandler<T>): Unsubscribe {
    this.subscriber.subscribe(topic)
    this.subscriber.on('message', (channel, message) => {
      if (channel !== topic) return
      const envelope = JSON.parse(message) as EventEnvelope<T>
      handler(envelope)
    })
    return () => this.subscriber.unsubscribe(topic)
  }

  subscribePattern<T>(pattern: string, handler: EventHandler<T>): Unsubscribe {
    this.subscriber.psubscribe(pattern)  // Pattern: 'radio.*', 'message.*'
    // ...
  }
}

interface EventEnvelope<T> {
  id: string
  topic: string
  payload: T
  timestamp: string
  version: number
}
```

---

## 3. Topic Taxonomy

All topics follow the `domain.event` or `domain.entity.event` naming pattern.

```
radio
  radio.telemetry           — Periodic telemetry snapshot (every ~1s)
  radio.state_changed       — Significant state change (PTT, connection, protection)
  radio.command_queued      — Command added to HAL queue
  radio.command_completed   — Command executed successfully
  radio.command_failed      — Command failed (error, timeout)
  radio.error               — Hardware error / unavailable

message
  message.new               — New message created
  message.updated           — Message content edited
  message.deleted           — Message soft-deleted
  message.delivered         — Message marked delivered to a recipient
  message.read              — Message marked read by a recipient

conversation
  conversation.created      — New conversation created
  conversation.updated      — Conversation metadata updated
  conversation.archived     — Conversation archived
  conversation.participant_joined — Participant added
  conversation.participant_left   — Participant removed

reaction
  reaction.added            — Reaction added to message
  reaction.removed          — Reaction removed from message

attachment
  attachment.uploaded       — Attachment upload received (not yet processed)
  attachment.ready          — Attachment processed and available
  attachment.failed         — Attachment processing failed

sync
  sync.needed               — Sync required for user/device
  sync.completed            — Sync batch delivered

presence
  presence.online           — User came online
  presence.offline          — User went offline
  presence.away             — User marked away

typing
  typing.started            — User started typing in conversation
  typing.stopped            — User stopped typing

geolocation
  geolocation.updated       — New GPS reading stored
  geolocation.sos           — SOS emergency event

webrtc
  webrtc.session_created    — New WebRTC session created
  webrtc.participant_joined — Participant joined WebRTC session
  webrtc.participant_left   — Participant left WebRTC session
  webrtc.session_ended      — WebRTC session terminated

system
  system.notification       — System-level notification
  system.maintenance        — Maintenance mode toggle
  system.radio_unavailable  — Radio hardware disconnected
```

---

## 4. Event Flow Diagrams

### 4.1 New Message Flow

```
MessagingService.sendMessage()
  │
  ├─► [PostgreSQL] INSERT INTO messages (status=sending)
  ├─► [PostgreSQL] INSERT INTO message_deliveries (per recipient, per channel)
  ├─► [PostgreSQL] UPDATE messages SET status='sent'
  │
  └─► EventBus.publish('message.new', { message, conversationId, participants })
        │
        ├─► WebSocket Gateway (subscribed to 'message.*')
        │     └─► Push MESSAGE_NEW frame to all online subscribers of conversation
        │
        ├─► DeliveryWorker (BullMQ, subscribed to 'message.new')
        │     └─► Create BullMQ jobs per offline recipient
        │           ├─► PushNotificationJob (mobile devices)
        │           └─► RadioDeliveryJob (if target station offline)
        │
        └─► SyncWorker (subscribed to 'message.new')
              └─► Append to sync_queue for offline devices
```

### 4.2 Radio Telemetry Flow

```
TelemetryPoller (every 1s)
  │
  ├─► [HAL] driver.getStatus()
  │
  └─► EventBus.publish('radio.telemetry', { status })
        │
        ├─► WebSocket Gateway (subscribed to 'radio.telemetry')
        │     └─► Push RADIO_TELEMETRY frame to subscribed connections
        │
        └─► [TimescaleDB] Insert radio_telemetry row (via background worker)
              (Not on critical path — buffered and batch-inserted)
```

### 4.3 HAL Command Flow

```
REST API: POST /api/v1/radio/profiles/1/frequency
  │
  ▼
RadioController → RadioService → HAL.setFrequency(hz, profileIdx)
  │
  ▼
RadioCommandQueue.enqueue('set_frequency', { hz, profileIdx })
  │ [BullMQ: persistent, retried on failure]
  │
  ▼
CommandWorker (BullMQ)
  └─► driver.exec('set_frequency', [hz, profileIdx])
        │
        ├─► [SUCCESS] EventBus.publish('radio.command_completed', { command, result })
        │                 └─► EventBus.publish('radio.state_changed', ...)
        │                       └─► WebSocket Gateway → RADIO_STATE_CHANGE
        │
        └─► [FAILURE] EventBus.publish('radio.command_failed', { command, error })
                          └─► WebSocket Gateway → RADIO_COMMAND_RESULT (error)
```

---

## 5. Background Job System (BullMQ)

### Queue Definitions

```typescript
// src/jobs/queues.ts

export const Queues = {
  RADIO_COMMANDS: 'radio:commands',
  MESSAGE_DELIVERY: 'message:delivery',
  PUSH_NOTIFICATIONS: 'push:notifications',
  EMAIL_DELIVERY: 'email:delivery',
  RADIO_DELIVERY: 'radio:delivery',
  ATTACHMENT_PROCESSING: 'attachment:processing',
  TELEMETRY_INGEST: 'telemetry:ingest',
  SYNC_PROCESSING: 'sync:processing',
  CLEANUP: 'cleanup',
  SCHEDULE_RUNNER: 'schedule:runner',
} as const
```

### Job Definitions

| Queue | Job | Purpose | Concurrency | Retry |
|---|---|---|---|---|
| `radio:commands` | `ExecuteCommand` | Execute HAL command | 1 (serial) | 3x exponential |
| `message:delivery` | `DeliverMessage` | Deliver pending messages | 5 | 5x exponential |
| `push:notifications` | `SendPush` | FCM/APNs push notifications | 10 | 3x |
| `email:delivery` | `SendEmail` | SMTP email delivery | 2 | 5x |
| `radio:delivery` | `RadioTransfer` | UUCP/HMP radio transfer | 1 | 3x |
| `attachment:processing` | `ProcessAttachment` | Resize/thumbnail/preview | 3 | 2x |
| `telemetry:ingest` | `IngestTelemetry` | Batch insert telemetry to TimescaleDB | 2 | 1x |
| `sync:processing` | `ProcessSync` | Process sync queue for reconnected devices | 3 | 3x |
| `cleanup` | `CleanupExpired` | Delete expired sessions, attachments, etc. | 1 | 1x |
| `schedule:runner` | `RunSchedule` | Execute connection schedules (UUCP caller) | 1 | 2x |

### BullMQ Worker Pattern

```typescript
// src/jobs/workers/MessageDeliveryWorker.ts

export class MessageDeliveryWorker {
  private worker: Worker

  constructor(private deliveryService: DeliveryService, redis: Redis) {
    this.worker = new Worker(
      Queues.MESSAGE_DELIVERY,
      async (job: Job<DeliverMessageJobData>) => {
        const { messageId, recipientId, channel, deviceId } = job.data

        const result = await this.deliveryService.deliver({
          messageId, recipientId, channel, deviceId
        })

        if (!result.success) {
          // BullMQ will retry based on job options
          throw new DeliveryError(result.error)
        }
      },
      {
        connection: redis,
        concurrency: 5,
        limiter: { max: 500, duration: 60_000 },
      }
    )

    this.worker.on('failed', (job, err) => {
      logger.error({ jobId: job?.id, err }, 'Message delivery job failed')
    })
  }
}
```

---

## 6. Event Ordering & Consistency

### Challenges

Redis Pub/Sub does not guarantee ordering within a topic when multiple publishers are involved. For the current single-server deployment this is acceptable — Redis processes messages in order per connection.

For future multi-server deployments:

1. **Sequence numbers**: The `sync_queue` table uses a monotonic `sequence` column per-user
2. **Event sourcing**: Events written to PostgreSQL before publishing to Redis — the DB is the source of truth
3. **NATS JetStream** (Phase 4): Provides durable, ordered, at-least-once delivery with consumer groups

### Idempotency

All event consumers are designed to be idempotent:

- `message.new` handlers check `client_message_id` before processing
- Delivery workers check `message_deliveries` status before attempting delivery
- Sync workers skip already-applied events using sequence comparison

---

## 7. Future: NATS JetStream Migration Path

When federation between multiple stations is required (Phase 4), the event bus will migrate from Redis Pub/Sub to **NATS JetStream**:

| Capability | Redis Pub/Sub | NATS JetStream |
|---|---|---|
| Fire-and-forget fanout | Yes | Yes |
| Durable subscriptions | No | Yes |
| Message persistence | No | Yes (stream storage) |
| Consumer groups | No | Yes |
| At-least-once delivery | No | Yes |
| Ordered per-subject | Yes (single server) | Yes (globally) |
| Multi-server federation | No | Yes (cluster + leaf nodes) |
| Operational complexity | Low | Medium |

The migration path:

1. Abstract the EventBus interface (already done in design)
2. Implement `NatsEventBus` implementing `IEventBus`
3. Switch via configuration: `EVENT_BUS_DRIVER=nats`
4. No change to publishers or consumers — only the transport changes
