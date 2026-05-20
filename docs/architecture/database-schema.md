# Database Schema Design

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Design Philosophy

The schema is fully normalized (3NF minimum) with deliberate denormalizations where query performance demands it. Every design decision documents its rationale. The schema is migration-managed via Drizzle ORM — no manual SQL patches.

**Guiding principles:**
- UUIDs as primary keys (globally unique, safe for future federation)
- `created_at` / `updated_at` on every mutable table (managed by application layer)
- `deleted_at` for soft deletes on user-visible entities
- `TIMESTAMPTZ` (not `TIMESTAMP`) everywhere — timezone-aware
- JSONB for extensible metadata without schema bloat
- TimescaleDB hypertables for time-series telemetry
- All foreign keys have explicit `ON DELETE` behavior
- Named indexes with descriptive names for debuggability

---

## 2. Schema Diagram (Entity Relationship)

```
users ──────────────────────────────────────────────────────┐
  │ 1                                                         │ 1
  │ ∞                                                         │ ∞
user_sessions          user_devices                  conversation_participants
                                                              │ ∞
                                                              │
conversations ────────────────────────────────────────────────┘
  │ 1
  │ ∞
messages ──────────────── message_reactions
  │ 1                             │ ∞
  │ ∞                          users
message_deliveries
  │ (per recipient per channel)
  │
attachments ──────────────────── messages (optional)

message_envelopes ────────────── messages
  (email interoperability)

radio_profiles ───────────────── radio_sessions
radio_telemetry (TimescaleDB hypertable)

frequencies ──────────────────── connection_schedules
gps_readings (TimescaleDB hypertable)

audit_logs
sync_cursors
sync_queue
```

---

## 3. Schema Definitions

### 3.1 Identity & Authentication

```sql
-- ─────────────────────────────────────────────────────────────────
-- users
-- Represents both human operators and remote station identities
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    callsign        TEXT NOT NULL UNIQUE,          -- Amateur radio callsign or station ID
    display_name    TEXT NOT NULL,
    email           TEXT UNIQUE,                   -- Optional: for email interoperability
    password_hash   TEXT,                          -- NULL for remote-only station users
    role            TEXT NOT NULL DEFAULT 'user'   -- admin | operator | user | readonly
                    CHECK (role IN ('admin', 'operator', 'user', 'readonly')),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'suspended', 'pending')),
    avatar_path     TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ
);

CREATE INDEX idx_users_callsign ON users (callsign);
CREATE INDEX idx_users_role ON users (role);
CREATE INDEX idx_users_status ON users (status);

-- ─────────────────────────────────────────────────────────────────
-- user_sessions
-- JWT refresh token tracking (token stored as hash, never plaintext)
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE user_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    token_hash      TEXT NOT NULL UNIQUE,          -- SHA-256 of refresh token
    device_id       UUID,                          -- Links to user_devices if known
    ip_address      INET,
    user_agent      TEXT,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_sessions_user_id ON user_sessions (user_id);
CREATE INDEX idx_user_sessions_token_hash ON user_sessions (token_hash);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions (expires_at)
    WHERE revoked_at IS NULL;

-- ─────────────────────────────────────────────────────────────────
-- user_devices
-- Tracks devices per user for multi-device sync
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE user_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    device_name     TEXT NOT NULL,
    device_type     TEXT NOT NULL                  -- mobile | desktop | station | browser
                    CHECK (device_type IN ('mobile', 'desktop', 'station', 'browser')),
    push_token      TEXT,                          -- FCM/APNs token for push notifications
    platform        TEXT,                          -- ios | android | web | linux
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_devices_user_id ON user_devices (user_id);
```

---

### 3.2 Conversations & Messaging

```sql
-- ─────────────────────────────────────────────────────────────────
-- conversations
-- A conversation is the fundamental unit of messaging context.
-- It replaces the inbox/outbox email model entirely.
-- type=direct: 2 participants, no title required
-- type=group: N participants, optional title
-- type=broadcast: 1-to-many, no replies (radio broadcasts)
-- type=radio: associated with a specific radio channel/frequency
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type            TEXT NOT NULL
                    CHECK (type IN ('direct', 'group', 'broadcast', 'radio')),
    title           TEXT,                          -- NULL for direct conversations
    description     TEXT,
    avatar_path     TEXT,
    created_by      UUID NOT NULL REFERENCES users (id) ON DELETE RESTRICT,
    last_message_id UUID,                          -- FK updated after INSERT (circular, deferred)
    last_activity_at TIMESTAMPTZ,                  -- Denormalized for sort performance
    archived_at     TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',   -- e.g., associated frequency, radio profile
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conversations_type ON conversations (type);
CREATE INDEX idx_conversations_last_activity ON conversations (last_activity_at DESC NULLS LAST);
CREATE INDEX idx_conversations_created_by ON conversations (created_by);

-- ─────────────────────────────────────────────────────────────────
-- conversation_participants
-- Many-to-many: users in conversations
-- last_read_message_id enables unread count without a full scan
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE conversation_participants (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id         UUID NOT NULL REFERENCES conversations (id) ON DELETE CASCADE,
    user_id                 UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    role                    TEXT NOT NULL DEFAULT 'member'
                            CHECK (role IN ('owner', 'admin', 'member')),
    last_read_message_id    UUID,                  -- For unread tracking
    last_read_at            TIMESTAMPTZ,
    muted_until             TIMESTAMPTZ,
    joined_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at                 TIMESTAMPTZ,
    UNIQUE (conversation_id, user_id)
);

CREATE INDEX idx_conv_participants_conversation ON conversation_participants (conversation_id);
CREATE INDEX idx_conv_participants_user ON conversation_participants (user_id);
CREATE INDEX idx_conv_participants_active ON conversation_participants (user_id)
    WHERE left_at IS NULL;

-- ─────────────────────────────────────────────────────────────────
-- messages
-- The core message entity. Chat-native design with email-compat fields.
-- client_message_id: client-generated UUID used for deduplication
--   (idempotency key — client resends on failure, server deduplicates)
-- reply_to_message_id: threading support (replies, not true threading)
-- subject: NULL for chat messages, populated for email-originated messages
-- status: tracks the sending pipeline state (not delivery — see message_deliveries)
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE messages (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id         UUID NOT NULL REFERENCES conversations (id) ON DELETE CASCADE,
    sender_id               UUID NOT NULL REFERENCES users (id) ON DELETE RESTRICT,
    client_message_id       UUID UNIQUE,           -- Client-generated idempotency key
    content                 TEXT,                  -- NULL for attachment-only messages
    content_type            TEXT NOT NULL DEFAULT 'text'
                            CHECK (content_type IN ('text', 'markdown', 'html', 'audio', 'system', 'attachment_only')),
    reply_to_message_id     UUID REFERENCES messages (id) ON DELETE SET NULL,
    forwarded_from_id       UUID REFERENCES messages (id) ON DELETE SET NULL,
    subject                 TEXT,                  -- Email interoperability: optional subject line
    status                  TEXT NOT NULL DEFAULT 'sending'
                            CHECK (status IN ('draft', 'sending', 'sent', 'failed')),
    edited_at               TIMESTAMPTZ,
    deleted_at              TIMESTAMPTZ,           -- Soft delete; content replaced with NULL
    metadata                JSONB NOT NULL DEFAULT '{}',
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Critical: conversation + time is the primary query pattern
CREATE INDEX idx_messages_conversation_created ON messages (conversation_id, created_at DESC)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_messages_sender ON messages (sender_id);
CREATE INDEX idx_messages_client_id ON messages (client_message_id)
    WHERE client_message_id IS NOT NULL;
CREATE INDEX idx_messages_reply_to ON messages (reply_to_message_id)
    WHERE reply_to_message_id IS NOT NULL;

-- ─────────────────────────────────────────────────────────────────
-- message_deliveries
-- Per-recipient, per-channel delivery tracking.
-- This replaces the simplistic "sent/received" inbox/outbox model.
-- channel=websocket: delivered via realtime WebSocket push
-- channel=email: delivered via SMTP (future)
-- channel=radio: delivered via HF radio / UUCP / HMP
-- channel=push: delivered via mobile push notification
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE message_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages (id) ON DELETE CASCADE,
    recipient_id    UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    channel         TEXT NOT NULL
                    CHECK (channel IN ('websocket', 'email', 'radio', 'push', 'sms')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'sent', 'delivered', 'read', 'failed')),
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,
    error           TEXT,
    attempts        SMALLINT NOT NULL DEFAULT 0,
    next_retry_at   TIMESTAMPTZ,
    UNIQUE (message_id, recipient_id, channel)
);

CREATE INDEX idx_deliveries_message ON message_deliveries (message_id);
CREATE INDEX idx_deliveries_recipient_status ON message_deliveries (recipient_id, status)
    WHERE status IN ('pending', 'sent', 'delivered');
CREATE INDEX idx_deliveries_next_retry ON message_deliveries (next_retry_at)
    WHERE status = 'pending' AND next_retry_at IS NOT NULL;

-- ─────────────────────────────────────────────────────────────────
-- message_reactions
-- Emoji reactions on messages (WhatsApp/Telegram model)
-- Unique per (message, user, emoji) — a user can react multiple times
-- with different emojis but not duplicate the same emoji
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE message_reactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages (id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    emoji           TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (message_id, user_id, emoji)
);

CREATE INDEX idx_reactions_message ON message_reactions (message_id);
```

---

### 3.3 Attachments

```sql
-- ─────────────────────────────────────────────────────────────────
-- attachments
-- Normalized attachment model. An attachment can exist before
-- being linked to a message (upload-first flow).
-- checksum: SHA-256 of file content (enables deduplication)
-- storage_path: relative path within the storage backend
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE attachments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id          UUID REFERENCES messages (id) ON DELETE SET NULL,
    conversation_id     UUID REFERENCES conversations (id) ON DELETE SET NULL,
    uploader_id         UUID NOT NULL REFERENCES users (id) ON DELETE RESTRICT,
    filename            TEXT NOT NULL,             -- Sanitized, stored filename
    original_filename   TEXT NOT NULL,             -- Original client-provided name
    mime_type           TEXT NOT NULL,
    size_bytes          BIGINT NOT NULL,
    storage_path        TEXT NOT NULL,             -- Relative to storage root
    storage_backend     TEXT NOT NULL DEFAULT 'local'
                        CHECK (storage_backend IN ('local', 's3', 'gcs')),
    checksum            TEXT NOT NULL,             -- SHA-256 (enables dedup)
    preview_path        TEXT,                      -- Thumbnail/preview path
    status              TEXT NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'processing', 'ready', 'failed', 'expired')),
    metadata            JSONB NOT NULL DEFAULT '{}', -- image dimensions, duration, etc.
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at          TIMESTAMPTZ,               -- For temporary attachments
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_attachments_message ON attachments (message_id)
    WHERE message_id IS NOT NULL;
CREATE INDEX idx_attachments_checksum ON attachments (checksum);
CREATE INDEX idx_attachments_uploader ON attachments (uploader_id);
CREATE INDEX idx_attachments_status ON attachments (status)
    WHERE status IN ('pending', 'processing');
```

---

### 3.4 Email Transport Envelope

```sql
-- ─────────────────────────────────────────────────────────────────
-- message_envelopes
-- Email-compatible transport metadata attached to messages.
-- This enables messages to transit through SMTP, UUCP, or HMP
-- without polluting the core message model.
-- One message can have at most one envelope.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE message_envelopes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id          UUID NOT NULL UNIQUE REFERENCES messages (id) ON DELETE CASCADE,
    envelope_type       TEXT NOT NULL
                        CHECK (envelope_type IN ('inbound', 'outbound')),
    from_address        TEXT NOT NULL,
    to_addresses        TEXT[] NOT NULL DEFAULT '{}',
    cc_addresses        TEXT[] NOT NULL DEFAULT '{}',
    bcc_addresses       TEXT[] NOT NULL DEFAULT '{}',
    subject             TEXT,
    headers             JSONB NOT NULL DEFAULT '{}',
    raw_message_path    TEXT,                      -- Path to original raw message file
    transport           TEXT NOT NULL
                        CHECK (transport IN ('smtp', 'uucp', 'radio', 'hmp', 'internal')),
    external_message_id TEXT,                      -- Message-ID from email header
    status              TEXT NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'processing', 'sent', 'received', 'failed')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at        TIMESTAMPTZ
);

CREATE INDEX idx_envelopes_message ON message_envelopes (message_id);
CREATE INDEX idx_envelopes_status ON message_envelopes (status, transport)
    WHERE status IN ('pending', 'processing');
CREATE INDEX idx_envelopes_external_id ON message_envelopes (external_message_id)
    WHERE external_message_id IS NOT NULL;
```

---

### 3.5 Radio Domain

```sql
-- ─────────────────────────────────────────────────────────────────
-- radio_profiles
-- Replaces the flat profile struct from sBitx.
-- Maps 1:1 to hardware profiles but stored in normalized DB.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE radio_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE, -- Station user
    profile_index   SMALLINT NOT NULL,             -- 0-based hardware profile index
    name            TEXT NOT NULL,
    frequency_hz    BIGINT NOT NULL,               -- Hz (not kHz/MHz) for precision
    mode            TEXT NOT NULL
                    CHECK (mode IN ('USB', 'LSB', 'CW', 'AM', 'FM', 'DIGITAL')),
    volume          SMALLINT NOT NULL DEFAULT 50   CHECK (volume BETWEEN 0 AND 100),
    bfo_hz          INT NOT NULL DEFAULT 0,
    digital_voice   BOOLEAN NOT NULL DEFAULT false,
    power_level     SMALLINT,                      -- NULL = default
    is_active       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (station_id, profile_index)
);

-- ─────────────────────────────────────────────────────────────────
-- radio_sessions
-- Records when the radio was active and its aggregate stats.
-- Useful for diagnostics, billing (future), and audit.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE radio_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    profile_id      UUID REFERENCES radio_profiles (id) ON DELETE SET NULL,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    bytes_tx        BIGINT NOT NULL DEFAULT 0,
    bytes_rx        BIGINT NOT NULL DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_radio_sessions_station ON radio_sessions (station_id, started_at DESC);

-- ─────────────────────────────────────────────────────────────────
-- radio_telemetry (TimescaleDB hypertable)
-- High-frequency telemetry data. Partitioned by time automatically.
-- Retention policy: 90 days by default (configurable).
-- DO NOT add non-time indexes on this table without benchmarking.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE radio_telemetry (
    time            TIMESTAMPTZ NOT NULL,
    station_id      UUID NOT NULL,
    frequency_hz    BIGINT,
    mode            TEXT,
    tx              BOOLEAN,
    rx              BOOLEAN,
    snr             SMALLINT,
    bitrate         INT,
    bytes_tx        BIGINT,
    bytes_rx        BIGINT,
    led_ok          BOOLEAN,
    connected       BOOLEAN,
    protection      BOOLEAN,
    power_level     SMALLINT,
    profile_idx     SMALLINT,
    timeout_counter BIGINT,
    metadata        JSONB
);

-- TimescaleDB: create hypertable partitioned by time (1 day chunks)
-- Run after CREATE TABLE:
-- SELECT create_hypertable('radio_telemetry', 'time', chunk_time_interval => INTERVAL '1 day');
-- SELECT add_retention_policy('radio_telemetry', INTERVAL '90 days');

CREATE INDEX idx_telemetry_station_time ON radio_telemetry (station_id, time DESC);
```

---

### 3.6 Frequencies & Scheduling

```sql
-- ─────────────────────────────────────────────────────────────────
-- frequencies
-- Known frequency presets. Replaces the current flat frequencies table.
-- is_gateway: this frequency connects to an internet-accessible gateway
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE frequencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alias           TEXT NOT NULL UNIQUE,
    frequency_hz    BIGINT NOT NULL,
    mode            TEXT NOT NULL DEFAULT 'USB'
                    CHECK (mode IN ('USB', 'LSB', 'CW', 'AM', 'FM', 'DIGITAL')),
    description     TEXT,
    is_gateway      BOOLEAN NOT NULL DEFAULT false,
    region          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_frequencies_alias ON frequencies (alias);
CREATE INDEX idx_frequencies_gateway ON frequencies (is_gateway) WHERE is_gateway = true;

-- ─────────────────────────────────────────────────────────────────
-- connection_schedules
-- Replaces the /caller endpoint. Represents UUCP/radio call schedules.
-- recurrence: RRULE-like JSONB (e.g., {"freq":"DAILY","byhour":[12,18]})
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE connection_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_callsign TEXT NOT NULL,
    frequency_id    UUID REFERENCES frequencies (id) ON DELETE SET NULL,
    scheduled_at    TIMESTAMPTZ NOT NULL,
    recurrence      JSONB,                         -- NULL = one-time
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'running', 'completed', 'failed', 'cancelled')),
    last_run_at     TIMESTAMPTZ,
    next_run_at     TIMESTAMPTZ,
    created_by      UUID REFERENCES users (id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schedules_next_run ON connection_schedules (next_run_at)
    WHERE status IN ('pending', 'running');
```

---

### 3.7 Geolocation

```sql
-- ─────────────────────────────────────────────────────────────────
-- gps_readings (TimescaleDB hypertable)
-- Time-series GPS coordinates per station.
-- Retention: 365 days by default.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE gps_readings (
    time            TIMESTAMPTZ NOT NULL,
    station_id      UUID NOT NULL,
    latitude        DOUBLE PRECISION NOT NULL,
    longitude       DOUBLE PRECISION NOT NULL,
    altitude_m      DOUBLE PRECISION,
    accuracy_m      DOUBLE PRECISION,
    speed_kmh       DOUBLE PRECISION,
    heading_deg     DOUBLE PRECISION,
    source          TEXT DEFAULT 'gps'             -- gps | manual | estimated
                    CHECK (source IN ('gps', 'manual', 'estimated')),
    metadata        JSONB
);

-- SELECT create_hypertable('gps_readings', 'time', chunk_time_interval => INTERVAL '7 days');
-- SELECT add_retention_policy('gps_readings', INTERVAL '365 days');

CREATE INDEX idx_gps_station_time ON gps_readings (station_id, time DESC);
```

---

### 3.8 Synchronization

```sql
-- ─────────────────────────────────────────────────────────────────
-- sync_cursors
-- Tracks each device's synchronization position per entity type.
-- Used by the sync engine to deliver deltas on reconnection.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE sync_cursors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    device_id           UUID REFERENCES user_devices (id) ON DELETE CASCADE,
    entity_type         TEXT NOT NULL,             -- 'messages' | 'conversations' | 'reactions' | etc.
    last_synced_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_event_sequence BIGINT NOT NULL DEFAULT 0, -- Monotonic sequence for ordering
    UNIQUE (user_id, device_id, entity_type)
);

CREATE INDEX idx_sync_cursors_user ON sync_cursors (user_id, device_id);

-- ─────────────────────────────────────────────────────────────────
-- sync_queue
-- Outbound delivery queue for offline devices.
-- Populated when a target device is offline.
-- Consumed when the device reconnects.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE sync_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_user_id  UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    target_device_id UUID REFERENCES user_devices (id) ON DELETE CASCADE,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    operation       TEXT NOT NULL CHECK (operation IN ('create', 'update', 'delete')),
    payload         JSONB NOT NULL,
    priority        SMALLINT NOT NULL DEFAULT 5,   -- 1=highest, 10=lowest
    sequence        BIGINT NOT NULL,               -- Monotonic, per-user ordering
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ,
    error           TEXT
);

CREATE INDEX idx_sync_queue_target ON sync_queue (target_user_id, target_device_id, sequence)
    WHERE processed_at IS NULL;
CREATE INDEX idx_sync_queue_priority ON sync_queue (priority, created_at)
    WHERE processed_at IS NULL;
```

---

### 3.9 Audit

```sql
-- ─────────────────────────────────────────────────────────────────
-- audit_logs
-- Immutable audit trail. Never updated or deleted.
-- Records all significant state changes across the platform.
-- ─────────────────────────────────────────────────────────────────
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id        UUID REFERENCES users (id) ON DELETE SET NULL,
    action          TEXT NOT NULL,                 -- e.g., 'message.created', 'radio.freq_changed'
    entity_type     TEXT NOT NULL,
    entity_id       TEXT NOT NULL,
    old_value       JSONB,
    new_value       JSONB,
    ip_address      INET,
    user_agent      TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_actor ON audit_logs (actor_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_logs (entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_logs (action, created_at DESC);
-- Note: audit_logs should also become a TimescaleDB hypertable in production
-- for efficient time-range queries and retention management
```

---

## 4. Indexing Strategy

| Table | Primary Query Pattern | Index |
|---|---|---|
| `messages` | Paginate by conversation + time | `(conversation_id, created_at DESC)` |
| `messages` | Lookup by client_message_id (dedup) | `(client_message_id)` UNIQUE |
| `message_deliveries` | Pending deliveries for retry | `(recipient_id, status)` partial |
| `conversation_participants` | Active conversations for user | `(user_id)` WHERE left_at IS NULL |
| `sync_queue` | Unprocessed items for device | `(target_user_id, sequence)` partial |
| `radio_telemetry` | Station telemetry by time range | TimescaleDB time-series index |
| `gps_readings` | Station GPS history | `(station_id, time DESC)` |
| `audit_logs` | Entity audit trail | `(entity_type, entity_id, created_at)` |

---

## 5. Migration Strategy

- All migrations managed by Drizzle ORM via `drizzle-kit`
- Migrations are versioned, sequential, and checked into version control
- Zero-downtime migrations required for all production changes:
  - Add columns as nullable first, backfill, then add constraints
  - Never DROP COLUMN without a deprecation period
  - Use `CONCURRENTLY` for index creation in production
- Migration rollback scripts written for every migration
- TimescaleDB hypertables created via migration with explicit idempotency checks

---

## 6. Retention & Archival

| Table | Retention | Strategy |
|---|---|---|
| `radio_telemetry` | 90 days | TimescaleDB retention policy, drop chunks |
| `gps_readings` | 365 days | TimescaleDB retention policy |
| `audit_logs` | 2 years | TimescaleDB hypertable (future) or pg_partman |
| `sync_queue` | 30 days | BullMQ job + periodic cleanup |
| `messages` (deleted) | 90 days | Cron job purges `deleted_at` > 90 days |
| `user_sessions` (expired) | 7 days after expiry | Cron job purges expired sessions |

---

## 7. Drizzle ORM Schema Mapping (TypeScript)

The above SQL is mirrored in `src/db/schema/` using Drizzle ORM's TypeScript DSL:

```typescript
// src/db/schema/messages.ts
import { pgTable, uuid, text, boolean, timestamp, jsonb } from 'drizzle-orm/pg-core'
import { conversations } from './conversations'
import { users } from './users'

export const messages = pgTable('messages', {
  id: uuid('id').primaryKey().defaultRandom(),
  conversationId: uuid('conversation_id').notNull().references(() => conversations.id, { onDelete: 'cascade' }),
  senderId: uuid('sender_id').notNull().references(() => users.id, { onDelete: 'restrict' }),
  clientMessageId: uuid('client_message_id').unique(),
  content: text('content'),
  contentType: text('content_type').notNull().default('text'),
  replyToMessageId: uuid('reply_to_message_id'),
  subject: text('subject'),
  status: text('status').notNull().default('sending'),
  editedAt: timestamp('edited_at', { withTimezone: true }),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})

export type Message = typeof messages.$inferSelect
export type NewMessage = typeof messages.$inferInsert
```
