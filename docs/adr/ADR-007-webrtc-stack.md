# ADR-007: WebRTC Stack Selection

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [webrtc.md](../architecture/webrtc.md)

---

## Context

HERMES stations require real-time audio communication between:

- Operators at different terminals on the same station (LAN)
- Remote stations connected via internet
- Eventually: internet users connected to HF radio stations (radio bridge)

The radio bridge use case is critical: audio from the HF radio receiver must be streamed to connected WebRTC clients, and audio from WebRTC clients must be routed to the radio transmitter. This is not standard P2P WebRTC — it requires server-side audio processing.

We need to evaluate WebRTC architectures (Mesh vs SFU vs MCU) and library choices.

---

## Decision

**Architecture**: Selective Forwarding Unit (SFU)  
**Library**: mediasoup v3  
**TURN server**: coturn  
**Signaling**: Existing WebSocket gateway (hermes-v1 protocol)

---

## Rationale

### Why SFU over Mesh or MCU

| Criteria | Mesh P2P | SFU (selected) | MCU |
|---|---|---|---|
| Server CPU (audio processing) | None | Low (forward only) | High (mix/transcode) |
| Client CPU | High (N-1 upstreams) | Low (1 upstream) | Low |
| Latency | Lowest | Low | High (transcoding) |
| Group call scalability | Poor (>4 people) | Excellent | Good |
| Radio bridge support | Impossible | Yes (PlainTransport) | Yes |
| Recording capability | Not available | Yes (per-track) | Yes (mixed) |
| NAT traversal complexity | High | Centralized (TURN) | Centralized |
| Server infrastructure | None | SFU server | MCU server |

The radio bridge use case makes Mesh P2P architecturally impossible: the HF radio is not a WebRTC endpoint — audio must be forwarded through a server that bridges ALSA audio (from radio) into the WebRTC media graph. Only SFU and MCU architectures support this.

MCU was rejected because:
- Higher CPU usage for mixing/transcoding — problematic on Raspberry Pi class hardware
- Higher latency from transcoding pipeline
- SFU's lower latency is better for real-time radio coordination

### Why mediasoup v3 over Janus or Jitsi

| Criteria | mediasoup v3 (selected) | Janus 1.x | Jitsi Videobridge |
|---|---|---|---|
| Language | C++ core, Node.js API | C library + REST | Java |
| TypeScript API | Native | Via HTTP REST | Via XMPP |
| Node.js integration | Seamless (same process) | External process | External process |
| Resource usage | Low | Medium | High |
| Radio bridge support | PlainTransport (native) | GStreamer plugin | GStreamer plugin |
| OPUS codec support | First-class | Yes | Yes |
| Custom signaling | Full control | Janus signaling | Jitsi signaling |
| Raspberry Pi support | Excellent | Good | Poor (Java) |

mediasoup's key advantages:

1. **PlainTransport**: A non-WebRTC RTP transport that receives RTP streams from arbitrary sources (e.g., ffmpeg piping radio audio). This is the cleanest path for the radio bridge.

2. **Node.js-native**: mediasoup is designed to be called from Node.js directly — no HTTP API, no external process, no language boundary. Worker processes are managed by the mediasoup Node.js library.

3. **Resource bounded**: Worker pool size capped at `min(os.cpus().length, 4)` — appropriate for station hardware ranging from Raspberry Pi 4 to dedicated servers.

4. **Full signaling control**: mediasoup provides no signaling protocol — you implement it. This is an advantage: our WebSocket gateway (hermes-v1) handles signaling naturally alongside messaging and telemetry.

Jitsi was rejected due to Java runtime overhead — unacceptable for Raspberry Pi class hardware.

### coturn for TURN/STUN

coturn is the reference TURN server implementation. Key configuration decisions:

- `denied-peer-ip` set to all RFC 1918 ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) — prevents SSRF via TURN
- Time-limited HMAC credentials (1-hour TTL, per-session)
- Port range 40000-49999 (UDP media relay)
- TURN secret stored in environment variable, never in code

### Signaling via Existing WebSocket Gateway

Adding WebRTC signaling through the existing `hermes-v1` WebSocket protocol avoids a second connection:

- Client already authenticated via WebSocket (JWT validated)
- WebRTC message types (`WEBRTC_CREATE_SESSION`, `WEBRTC_TRANSPORT_CREATED`, etc.) added to the protocol
- Client does not need a separate WebRTC signaling connection
- Server-side routing: WebSocket gateway receives WebRTC frames and routes to WebRTC session manager

---

## Consequences

### Positive

- mediasoup PlainTransport enables radio bridge (audio in/out via RTP)
- TypeScript API eliminates language boundaries
- No separate signaling server needed
- TURN credentials are per-session and time-limited (security)
- Single Raspberry Pi 4 can handle 4 concurrent WebRTC sessions (mediasoup worker per CPU core)

### Negative

- mediasoup requires native C++ compilation (`node-gyp`) — adds build complexity
- Worker processes add memory overhead (~50MB per worker)
- Radio bridge audio pipeline (ffmpeg → RTP → mediasoup) adds ~200-500ms latency
- Single-region SFU — multi-region requires SFU cascade (Phase 4)

### Mitigations

- Docker build image handles `node-gyp` compilation in CI
- Worker count bounded to 4 — memory usage predictable
- Radio bridge latency is acceptable for coordinated radio communication (better than PTT radio)
- Single-region is sufficient for all current deployment scenarios

---

## Alternatives Considered

- **Janus**: Valid choice, but external HTTP API and less native Node.js integration
- **Jitsi Videobridge**: Java overhead; unsuitable for Raspberry Pi hardware
- **Mesh P2P**: Impossible for radio bridge use case
- **LiveKit**: Cloud-managed SFU, excellent — but closed-source cloud dependency; field deployments require self-hosted
- **Ion SFU (Go)**: Promising, but immature and no PlainTransport equivalent

---

## Related Documents

- [WebRTC Architecture](../architecture/webrtc.md)
- [Security Architecture](../architecture/security.md) — TURN credential security
- [ADR-001: Backend Framework](ADR-001-backend-framework.md) — Node.js selection enabling mediasoup
