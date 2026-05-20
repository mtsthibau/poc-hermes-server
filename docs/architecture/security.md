# Security Architecture

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Security Principles

1. **Defense in depth**: Multiple independent layers, not a single perimeter
2. **Least privilege**: Every component accesses only what it needs
3. **Fail closed**: Default deny, explicit allow
4. **No trust at boundaries**: All inputs validated, all CLI args sanitized
5. **Auditability**: All significant actions logged with actor, timestamp, context
6. **Secrets never in code**: All secrets in environment variables or a secrets manager

---

## 2. Authentication System

### 2.1 JWT Token Architecture

```
                    RS256 (asymmetric signing)
                    Private key: server only
                    Public key: distributable to future gateway/verifier services

Access Token:   15-minute expiry, contains userId, callsign, role, sessionId
Refresh Token:  30-day expiry, opaque random string (SHA-256 hash stored in DB)
```

```typescript
// JWT access token payload
interface AccessTokenPayload {
  sub: string         // userId (UUID)
  callsign: string    // Amateur radio callsign
  role: UserRole      // admin | operator | user | readonly
  sessionId: string   // Links to user_sessions row
  iat: number         // Issued at (Unix timestamp)
  exp: number         // Expiry (Unix timestamp)
  // No sensitive data (email, password, etc.) in token payload
}
```

### 2.2 Login Flow

```
POST /api/v1/auth/login
  { callsign, password }
  │
  ▼
1. Lookup user by callsign (constant-time to prevent timing attacks)
2. Verify password using bcrypt (cost factor ≥ 12)
3. Generate access token (JWT RS256, 15min)
4. Generate refresh token (crypto.randomBytes(48).toString('hex'))
5. Store SHA-256(refreshToken) in user_sessions
6. Return { accessToken, refreshToken, expiresIn: 900 }
```

### 2.3 Token Refresh Flow

```
POST /api/v1/auth/refresh
  { refreshToken }
  │
  ▼
1. Look up SHA-256(refreshToken) in user_sessions
2. Verify session not expired, not revoked
3. Issue new access token
4. Optionally rotate refresh token (sliding window)
```

### 2.4 WebSocket Authentication

```
Client connects to wss://host/ws
  │
  ▼ [Server starts 10-second auth timeout]
  │
Client sends: { type: 'AUTHENTICATE', payload: { token: 'eyJ...' } }
  │
  ▼
1. Verify JWT signature using public key (no DB lookup needed)
2. Verify expiry, issuer, audience
3. Mark connection as authenticated, store userId, role, sessionId
4. Cancel auth timeout
5. Send: { type: 'AUTHENTICATED', payload: { userId, callsign, role, sessionId } }
  │
  ▼
[If auth not received within 10 seconds]
  → Close connection with code 4001 (UNAUTHENTICATED)
```

---

## 3. Authorization (RBAC)

### Role Definitions

| Role | Description | Allowed Operations |
|---|---|---|
| `admin` | Full system access | All operations |
| `operator` | Radio operator | All radio controls, messaging, schedule management |
| `user` | Standard user | Messaging, read radio status, view schedules |
| `readonly` | Observer | Read-only access to status, messages |

### Permission Matrix

| Operation | admin | operator | user | readonly |
|---|---|---|---|---|
| Send messages | ✓ | ✓ | ✓ | — |
| Delete own messages | ✓ | ✓ | ✓ | — |
| Delete any message | ✓ | — | — | — |
| Read messages (own conversations) | ✓ | ✓ | ✓ | ✓ |
| Set radio frequency | ✓ | ✓ | — | — |
| Set PTT | ✓ | ✓ | — | — |
| View radio telemetry | ✓ | ✓ | ✓ | ✓ |
| Manage users | ✓ | — | — | — |
| Manage frequencies | ✓ | ✓ | — | — |
| Create schedules | ✓ | ✓ | — | — |
| View system config | ✓ | ✓ | — | — |
| Modify system config | ✓ | — | — | — |
| Reboot/shutdown | ✓ | — | — | — |
| Access audit logs | ✓ | — | — | — |
| Erase SD card | ✓ | — | — | — |

### RBAC Middleware

```typescript
// src/api/middleware/rbac.ts

export function requireRole(...allowedRoles: UserRole[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.user  // Set by JWT middleware
    if (!user) {
      return reply.status(401).send({ code: 'UNAUTHENTICATED' })
    }
    if (!allowedRoles.includes(user.role)) {
      // Audit log: unauthorized access attempt
      await auditLogger.log({
        actorId: user.id,
        action: 'auth.unauthorized_access',
        entityType: 'route',
        entityId: request.routeOptions.url,
        metadata: { role: user.role, required: allowedRoles }
      })
      return reply.status(403).send({ code: 'FORBIDDEN' })
    }
  }
}

// Usage:
app.post('/api/v1/radio/profiles/:id/frequency',
  { preHandler: [requireAuth, requireRole('admin', 'operator')] },
  radioController.setFrequency
)
```

---

## 4. Input Validation & Injection Prevention

### 4.1 REST API — JSON Schema Validation

All request bodies are validated via Fastify's built-in AJV before reaching controllers. Unknown fields are stripped.

```typescript
// Route schema example
const schema = {
  body: {
    type: 'object',
    required: ['frequencyHz', 'profileIdx'],
    additionalProperties: false,  // Strip unknown fields
    properties: {
      frequencyHz: {
        type: 'integer',
        minimum: 1_000_000,       // 1 MHz minimum (HF band)
        maximum: 30_000_000,      // 30 MHz maximum (HF band)
      },
      profileIdx: {
        type: 'integer',
        minimum: 0,
        maximum: 9,
      }
    }
  }
}
```

### 4.2 HAL CLI Injection Prevention

All CLI commands are executed via `child_process.execFile` with arguments as an array:

```typescript
// SAFE: Arguments are separate strings, not shell-interpolated
execFile('/usr/local/bin/sbitx_client', ['set_frequency', '14200000', '0'])

// NEVER: Shell injection risk
exec(`sbitx_client set_frequency ${userInput}`)   // ← FORBIDDEN
execFile('/usr/local/bin/sbitx_client', [`set_frequency ${userInput}`])  // ← FORBIDDEN
```

Additionally:
- Numeric CLI parameters are parsed and validated as integers/floats before CLI invocation
- String parameters (mode values) are matched against an explicit allowlist
- The CLI binary path is read from environment configuration, never from request input

### 4.3 SQL Injection Prevention

All database queries use Drizzle ORM parameterized queries. Raw SQL is only used in migration files.

```typescript
// SAFE: Parameterized
await db.select().from(messages).where(eq(messages.conversationId, conversationId))

// FORBIDDEN in application code:
await db.execute(sql`SELECT * FROM messages WHERE id = '${userInput}'`)
```

### 4.4 WebSocket Input Validation

All WebSocket message payloads are validated against typed schemas using AJV before routing:

```typescript
const payloadSchema = ajv.compile({
  type: 'object',
  required: ['conversationId'],
  additionalProperties: false,
  properties: {
    conversationId: { type: 'string', format: 'uuid' }
  }
})

if (!payloadSchema(message.payload)) {
  return sendError(connection, 'INVALID_PAYLOAD', payloadSchema.errors)
}
```

---

## 5. Transport Security

### 5.1 TLS

- All HTTP/HTTPS and WebSocket connections must use TLS (minimum TLS 1.2, TLS 1.3 preferred)
- Certificates: Let's Encrypt for internet-accessible stations; self-signed with custom CA for field deployments
- HSTS header enforced: `Strict-Transport-Security: max-age=31536000; includeSubDomains`

### 5.2 CORS

```typescript
await app.register(fastifyCors, {
  origin: (origin, callback) => {
    const allowed = process.env.CORS_ALLOWED_ORIGINS!.split(',')
    if (!origin || allowed.includes(origin)) {
      return callback(null, true)
    }
    callback(new Error('CORS not allowed'), false)
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
})
```

### 5.3 Security Headers

```typescript
await app.register(fastifyHelmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      connectSrc: ["'self'", 'wss://*'],
      mediaSrc: ["'self'"],
    }
  },
  hsts: { maxAge: 31536000, includeSubdomains: true },
  noSniff: true,
  xssFilter: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
})
```

---

## 6. Rate Limiting

```typescript
// Per-IP rate limiting for auth endpoints
await app.register(fastifyRateLimit, {
  max: 5,               // 5 attempts
  timeWindow: '15 minutes',
  keyGenerator: (req) => req.ip,
  errorResponseBuilder: () => ({
    code: 'RATE_LIMITED',
    message: 'Too many login attempts. Wait 15 minutes.'
  }),
  allowList: ['127.0.0.1'],  // Allow local health checks
})

// Per-user rate limiting for API endpoints
// Implemented via Redis sliding window counter
```

| Endpoint | Limit |
|---|---|
| `POST /auth/login` | 5 requests / 15 min per IP |
| `POST /auth/refresh` | 20 requests / hour per user |
| REST API (general) | 300 requests / minute per user |
| REST API (radio commands) | 60 requests / minute per user |
| WebSocket messages | 60 messages / minute per connection |
| WebSocket RADIO_COMMAND | 10 / minute per connection |
| File upload | 20 uploads / hour per user |

---

## 7. Attachment Security

- MIME type validated from file content bytes (using `file-type` library), not `Content-Type` header
- Filename sanitized: `path.basename(filename)` to strip path traversal attempts
- Stored with UUID filename, never original client filename
- Downloads served via signed time-limited URL (token in query string, 1-hour expiry)
- Downloads validate conversation membership before serving attachment
- Max file size enforced at streaming level: configurable (default 50MB)
- Virus scanning recommended for production (ClamAV integration hook provided)

---

## 8. Secrets Management

```bash
# .env (never committed to version control)
DATABASE_URL=postgresql://hermes:password@localhost:5432/hermes
REDIS_URL=redis://localhost:6379
JWT_PRIVATE_KEY_PATH=/secrets/jwt.key        # RSA private key file
JWT_PUBLIC_KEY_PATH=/secrets/jwt.pub         # RSA public key file
TURN_SECRET=<64-char random hex>             # TURN HMAC secret
STORAGE_ENCRYPTION_KEY=<32-char random>      # For attachment encryption at rest
```

For production:
- Secrets stored in HashiCorp Vault, AWS Secrets Manager, or equivalent
- Application fetches secrets at startup, not hardcoded
- Secret rotation supported without downtime (dual-key verification window)
- Private key rotation invalidates all active sessions (acceptable — users re-login)

---

## 9. Audit Logging

All significant actions are written to `audit_logs` before the operation completes:

```typescript
// Actions that MUST be audited:
'auth.login'                  // Successful login
'auth.login_failed'           // Failed login attempt
'auth.logout'                 // Logout / session revoke
'user.created'                // New user created
'user.updated'                // User modified
'user.deleted'                // User deleted
'user.role_changed'           // Role escalation/demotion
'message.deleted'             // Message deletion
'conversation.archived'       // Conversation archived
'radio.frequency_changed'     // Frequency modified
'radio.ptt_activated'         // PTT enabled
'radio.protection_reset'      // SWR protection reset
'system.config_changed'       // System configuration modified
'system.reboot'               // System reboot initiated
'system.shutdown'             // Shutdown initiated
'auth.unauthorized_access'    // RBAC violation attempt
'attachment.deleted'          // Attachment removed
```

---

## 10. OWASP Top 10 Coverage

| OWASP Risk | Mitigation |
|---|---|
| A01 Broken Access Control | RBAC middleware on all routes; per-user sync filtering |
| A02 Cryptographic Failures | JWT RS256; bcrypt passwords; TLS enforced; secrets in env |
| A03 Injection | AJV schema validation; Drizzle parameterized queries; execFile array args |
| A04 Insecure Design | Threat-modeled auth flows; fail-closed defaults; audit logging |
| A05 Security Misconfiguration | Helmet headers; CORS allowlist; HSTS; no debug in production |
| A06 Vulnerable Components | `npm audit` in CI; Dependabot alerts; regular dep updates |
| A07 Auth & Session Failures | JWT expiry; refresh token rotation; session revocation |
| A08 Software & Data Integrity | Dependency lockfile; attachment checksum; signed URLs |
| A09 Security Logging | Audit log for all significant events; structured Pino logs |
| A10 SSRF | No user-controlled URLs fetched; CLI binary path from config only |
