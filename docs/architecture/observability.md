# Observability & Reliability

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Observability Pillars

The three pillars of observability — **logs**, **metrics**, and **traces** — are all implemented. An opaque system is an operationally fragile system.

---

## 2. Logging (Pino)

**Library**: Pino — fastest Node.js logger, structured JSON output, minimal overhead

```typescript
// src/shared/logger.ts

import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: {
    service: 'poc-hermes-server',
    version: process.env.npm_package_version,
    environment: process.env.NODE_ENV,
  },
  redact: {
    paths: ['password', 'token', 'refreshToken', 'authorization', '*.password'],
    censor: '[REDACTED]',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  // Production: output as JSON
  // Development: pino-pretty for human-readable output
})
```

### Log Levels

| Level | When to Use |
|---|---|
| `trace` | HAL CLI command details, individual SQL queries (dev only) |
| `debug` | WebSocket frame processing, cache hits/misses |
| `info` | Startup, shutdown, successful auth, schedule execution |
| `warn` | Degraded state (hardware unavailable, retry attempt), rate limit hits |
| `error` | Failed delivery, HAL errors, unhandled exceptions |
| `fatal` | Unrecoverable failure — process will exit |

### Request Correlation

Every HTTP request and WebSocket connection gets a unique `requestId` (UUID), propagated through all log entries for that request:

```typescript
// Fastify request logging middleware
app.addHook('onRequest', async (request) => {
  request.log = logger.child({ requestId: request.id, userId: request.user?.id })
})
```

### Log Output Examples

```json
{"level":"info","time":"2026-05-20T14:32:00.000Z","service":"poc-hermes-server","requestId":"uuid-123","userId":"uuid-456","msg":"POST /api/v1/conversations/uuid-789/messages 201","statusCode":201,"durationMs":12}

{"level":"error","time":"2026-05-20T14:32:05.000Z","service":"poc-hermes-server","requestId":"uuid-124","msg":"HAL command failed","command":"set_frequency","error":{"code":"COMMAND_TIMEOUT","message":"sbitx_client did not respond within 5000ms"}}
```

---

## 3. Metrics (Prometheus)

**Library**: `prom-client` — official Prometheus client for Node.js

Metrics exposed at `/metrics` (internal only — not exposed via public API gateway).

### Key Metrics

```typescript
// HTTP
http_requests_total{method, route, status_code}
http_request_duration_seconds{method, route, status_code} [histogram]
http_request_size_bytes{method, route} [histogram]

// WebSocket
ws_connections_active [gauge]
ws_connections_total{status} [counter]
ws_messages_received_total{type} [counter]
ws_messages_sent_total{type} [counter]
ws_auth_duration_seconds [histogram]

// HAL / Radio
hal_commands_total{command, status} [counter]
hal_command_duration_seconds{command} [histogram]
hal_queue_depth [gauge]
radio_telemetry_polls_total{status} [counter]
radio_snr_current [gauge]
radio_bitrate_current_bps [gauge]
radio_tx_active [gauge]
radio_connected [gauge]

// Messaging
messages_sent_total{content_type} [counter]
message_delivery_total{channel, status} [counter]
message_delivery_duration_seconds{channel} [histogram]
sync_queue_depth{user_id_bucket} [histogram]

// Jobs (BullMQ)
bullmq_jobs_active{queue} [gauge]
bullmq_jobs_waiting{queue} [gauge]
bullmq_jobs_completed_total{queue} [counter]
bullmq_jobs_failed_total{queue} [counter]
bullmq_job_duration_seconds{queue} [histogram]

// Database
pg_connection_pool_size [gauge]
pg_connection_pool_idle [gauge]
pg_query_duration_seconds{query_type} [histogram]

// Cache
redis_operations_total{operation, status} [counter]
redis_operation_duration_seconds{operation} [histogram]

// System
process_cpu_usage [gauge]
process_memory_usage_bytes [gauge]
nodejs_event_loop_lag_seconds [gauge]
```

### Grafana Dashboard Panels

- **Radio Status**: Current SNR, bitrate, TX/RX state, connection status
- **API Performance**: Request rate, latency percentiles (p50, p95, p99), error rate
- **WebSocket**: Active connections, message throughput, auth failures
- **Message Delivery**: Delivery rate by channel, delivery latency, failure rate
- **Queue Health**: Queue depths, job processing rates, failed jobs
- **Database**: Query latency, connection pool usage, slow queries
- **System**: CPU, memory, event loop lag

---

## 4. Distributed Tracing (OpenTelemetry)

**Library**: `@opentelemetry/sdk-node` with auto-instrumentation

```typescript
// src/instrumentation.ts (loaded before any other code)

import { NodeSDK } from '@opentelemetry/sdk-node'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { Resource } from '@opentelemetry/resources'
import { SEMRESATTRS_SERVICE_NAME } from '@opentelemetry/semantic-conventions'

const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: 'poc-hermes-server',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-ioredis': { enabled: true },
      '@opentelemetry/instrumentation-fs': { enabled: false }, // Too noisy
    })
  ],
})

sdk.start()
```

Traces are exported to a compatible backend (Jaeger, Tempo, Honeycomb, etc.).

### Manual Spans for Critical Paths

```typescript
import { trace } from '@opentelemetry/api'

const tracer = trace.getTracer('hermes-messaging')

async function sendMessage(params: SendMessageParams) {
  return tracer.startActiveSpan('messaging.sendMessage', async (span) => {
    span.setAttributes({
      'conversation.id': params.conversationId,
      'message.content_type': params.contentType,
    })
    try {
      const result = await this.doSend(params)
      span.setStatus({ code: SpanStatusCode.OK })
      return result
    } catch (err) {
      span.recordException(err as Error)
      span.setStatus({ code: SpanStatusCode.ERROR })
      throw err
    } finally {
      span.end()
    }
  })
}
```

---

## 5. Health Checks

```
GET /health            — Basic liveness (process is up)
GET /health/ready      — Readiness (DB, Redis, HAL available)
GET /health/live       — Liveness (no deadlock, event loop not lagging)
```

```typescript
// src/api/v1/health/health.routes.ts

app.get('/health/ready', async (_, reply) => {
  const checks = await Promise.allSettled([
    db.execute(sql`SELECT 1`),                          // PostgreSQL
    redis.ping(),                                       // Redis
    hal.isAvailable(),                                  // Radio HAL
  ])

  const [db, redis, radio] = checks.map(r => r.status === 'fulfilled')

  const status = db && redis ? 'ready' : 'not_ready'
  return reply
    .status(db && redis ? 200 : 503)
    .send({ status, checks: { database: db, cache: redis, radio } })
})
```

---

## 6. Alerting Rules (Prometheus Alertmanager)

```yaml
# Critical alerts (PagerDuty / SMS)
- alert: RadioHardwareUnavailable
  expr: radio_connected == 0
  for: 2m
  labels: { severity: critical }
  annotations:
    summary: "Radio hardware disconnected for >2 minutes"

- alert: DatabaseUnreachable
  expr: up{job="postgres"} == 0
  for: 1m
  labels: { severity: critical }

- alert: HighErrorRate
  expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
  for: 2m
  labels: { severity: critical }

# Warning alerts (Slack / email)
- alert: HighDeliveryFailureRate
  expr: rate(message_delivery_total{status="failed"}[15m]) > 0.1
  for: 5m
  labels: { severity: warning }

- alert: HALCommandQueueDepth
  expr: hal_queue_depth > 50
  for: 1m
  labels: { severity: warning }

- alert: SyncQueueBacklog
  expr: sum(sync_queue_depth) > 1000
  for: 5m
  labels: { severity: warning }

- alert: EventLoopLag
  expr: nodejs_event_loop_lag_seconds > 0.1
  for: 2m
  labels: { severity: warning }
```

---

## 7. Reliability Patterns

### 7.1 Graceful Shutdown

```typescript
const shutdown = async (signal: string) => {
  logger.info({ signal }, 'Graceful shutdown initiated')

  // 1. Stop accepting new HTTP requests
  await app.close()

  // 2. Stop accepting new WebSocket connections
  await wsGateway.close()

  // 3. Wait for active WebSocket connections to finish (max 30s)
  await wsGateway.waitForDrain(30_000)

  // 4. Stop BullMQ workers gracefully (finish current jobs)
  await workerManager.shutdown()

  // 5. Close HAL (stops telemetry poller, drains command queue)
  await hal.shutdown()

  // 6. Close DB pool
  await db.end()

  // 7. Close Redis
  await redis.quit()

  logger.info('Shutdown complete')
  process.exit(0)
}

process.on('SIGTERM', () => shutdown('SIGTERM'))
process.on('SIGINT', () => shutdown('SIGINT'))
```

### 7.2 Circuit Breaker (HAL)

If the HAL driver fails repeatedly, a circuit breaker prevents cascading failures:

```
State: CLOSED → (failure threshold reached) → OPEN
State: OPEN → (timeout elapsed) → HALF-OPEN
State: HALF-OPEN → (success) → CLOSED | (failure) → OPEN
```

When the circuit is OPEN:
- HAL commands return `{ success: false, error: { code: 'HARDWARE_UNAVAILABLE' } }` immediately
- Clients receive `RADIO_STATE_CHANGE` with `connected: false`
- System alert fires: `RadioHardwareUnavailable`

### 7.3 Database Connection Pooling

```typescript
// Drizzle + pg pool configuration
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,              // Max connections (tune based on PostgreSQL max_connections)
  min: 2,               // Keep 2 idle connections ready
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  // Pool health check
  query_timeout: 10_000,
})
```

---

## 8. Observability Stack (Self-Hosted)

```yaml
# docker-compose.observability.yml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes: [./infrastructure/observability/prometheus.yml:/etc/prometheus/prometheus.yml]
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3030:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "${GRAFANA_PASSWORD}"
    volumes: [./infrastructure/observability/dashboards:/var/lib/grafana/dashboards]

  tempo:
    image: grafana/tempo:latest
    ports: ["3200:3200", "4317:4317"]  # gRPC OTLP

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./infrastructure/observability/promtail.yml:/etc/promtail/config.yml
```

Grafana → queries Prometheus (metrics) + Tempo (traces) + Loki (logs). Full observability in a single pane of glass.
