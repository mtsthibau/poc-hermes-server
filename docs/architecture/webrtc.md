# WebRTC Architecture

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-06-10

---

## 1. Overview

The platform requires real-time audio communication to enable voice conversations between Hermes station operators, and to bridge HF radio audio into the browser/mobile UI. This document covers the WebRTC architecture, stack selection, signaling design, session lifecycle, and radio-to-WebRTC bridging.

---

## 2. Use Cases

| Use Case | Priority |
|---|---|
| Same-station / LAN technical validation call between two browsers | Phase 2 |
| Browser-to-browser voice call between two operators | Phase 3 |
| Multiple operators in a group voice channel | Phase 3 |
| Radio audio streamed to browser (receive) | Phase 3 |
| Browser audio transmitted via radio (transmit) | Phase 4 |
| Radio-to-WebRTC bidirectional bridge | Phase 4 |
| Future: Video channels | Phase 5+ |

---

## 3. Architecture: SFU (Selective Forwarding Unit)

### Why SFU over Mesh

| Aspect | Mesh (P2P) | SFU (mediasoup) |
|---|---|---|
| Scalability | Max 4-6 participants (bandwidth) | 100+ participants |
| Server load | Low | Moderate (media routing) |
| Latency | Lowest (direct P2P) | Low (~10ms additional) |
| NAT traversal | Hard (each peer needs TURN) | Only server needs TURN |
| Radio bridging | Impossible | Native server-side |
| Recording | Impossible client-side | Native server-side |
| Bandwidth control | None | Per-participant bitrate |
| Deployment complexity | Low | Medium |

**Decision**: SFU is required because radio bridging and group calls with >2 participants are first-class requirements. Mesh would be blocked by these at the first group call.

### Stack

| Component | Technology | Rationale |
|---|---|---|
| SFU | mediasoup v3 | Node.js native, TypeScript, battle-tested, excellent docs |
| Signaling | Existing WebSocket Gateway | Reuses auth/subscription infrastructure |
| STUN | coturn (self-hosted) | Free, battle-tested, full control |
| TURN | coturn (self-hosted) | Required for NAT traversal in field deployments |
| Audio codec | Opus | Best-in-class for voice, adaptive bitrate |
| Future video | VP8/H.264 | mediasoup supports both |

---

## 4. Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           Clients                                          │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────┐  │
│  │  Browser         │    │  Mobile App      │    │  Station B (remote)  │  │
│  │  (hermes-chat)   │    │  (Capacitor)     │    │  (future)            │  │
│  └────────┬─────────┘    └────────┬─────────┘    └──────────┬───────────┘  │
│           │                       │                         │              │
│    WebSocket (Signaling)   WebSocket (Signaling)     WebSocket/NATS        │
└────────────╪══════════════════════╪═════════════════════════╪═════════════╝
             │                      │                         │
┌────────────▼══════════════════════▼═════════════════════════▼═════════════╗
║                      poc-hermes-server                                    ║
║                                                                           ║
║  ┌────────────────────────────────────────────────────────────────────┐   ║
║  │                  WebSocket Gateway (Signaling Layer)               │   ║
║  │                                                                    │   ║
║  │  Message types: WEBRTC_OFFER, WEBRTC_ANSWER, WEBRTC_ICE_CANDIDATE │   ║
║  │                 WEBRTC_SESSION_JOINED, WEBRTC_SESSION_ENDED        │   ║
║  └────────────────────────────┬───────────────────────────────────────┘   ║
║                               │                                           ║
║  ┌────────────────────────────▼───────────────────────────────────────┐   ║
║  │                   WebRTC Session Manager                           │   ║
║  │                                                                    │   ║
║  │  - Session lifecycle (create, join, leave, terminate)             │   ║
║  │  - Router management (1 mediasoup Router per session)             │   ║
║  │  - Transport lifecycle (WebRtcTransport per participant)           │   ║
║  │  - Producer/Consumer tracking                                      │   ║
║  └────────────────────────────┬───────────────────────────────────────┘   ║
║                               │                                           ║
║  ┌────────────────────────────▼───────────────────────────────────────┐   ║
║  │                   mediasoup v3 (SFU)                               │   ║
║  │                                                                    │   ║
║  │  Workers (max(1, min(cpuCount - 2, 4)))                            │   ║
║  │  Routers (1 per active session)                                    │   ║
║  │  WebRtcTransports (1 per participant, per direction)               │   ║
║  │  Producers (audio/video track sources)                             │   ║
║  │  Consumers (audio/video track sinks)                               │   ║
║  └────────────────────────────┬───────────────────────────────────────┘   ║
║                               │  SRTP/DTLS (encrypted media)              ║
╚═══════════════════════════════╪═══════════════════════════════════════════╝
                                │
        ┌───────────────────────┼───────────────────────┐
        │ ICE/STUN              │ TURN relay             │ Radio Bridge
        ▼                       ▼                        ▼
  ┌──────────┐         ┌─────────────────┐    ┌─────────────────────┐
  │  coturn  │         │  coturn TURN    │    │  RadioAudioBridge   │
  │  (STUN)  │         │  relay          │    │  (ffmpeg + OPUS)    │
  └──────────┘         └─────────────────┘    └─────────────────────┘
  :3478 UDP/TCP        :5349 TLS                Port 40000-49999 UDP
```

---

## 5. Signaling Protocol

Signaling is delivered over the existing **WebSocket Gateway** using new message types. This avoids building a separate signaling server.

### Session Creation

```
Client A                     Server (SessionManager)           Client B
   │                                │                              │
   │── WEBRTC_CREATE_SESSION ──────►│                              │
   │   { conversationId }           │                              │
   │◄─ WEBRTC_SESSION_CREATED ──────│                              │
   │   { sessionId, rtpCapabilities }│                             │
   │                                │                              │
   │── WEBRTC_GET_ROUTER_CAPS ─────►│                              │
   │◄─ WEBRTC_ROUTER_CAPS ──────────│                              │
   │   { rtpCapabilities }          │                              │
   │                                │                              │
   │── WEBRTC_CREATE_TRANSPORT ────►│                              │
   │   { direction: 'send' }        │                              │
   │◄─ WEBRTC_TRANSPORT_CREATED ───│                              │
   │   { id, iceParameters,         │                              │
   │     iceCandidates, dtlsParams} │                              │
   │                                │                              │
   │── WEBRTC_CONNECT_TRANSPORT ───►│                              │
   │   { transportId, dtlsParams }  │                              │
   │                                │                              │
   │── WEBRTC_PRODUCE ─────────────►│                              │
   │   { transportId, kind='audio', │                              │
   │     rtpParameters }            │                              │
   │◄─ WEBRTC_PRODUCED ─────────────│                              │
   │   { producerId }               │                              │
   │                                │─── WEBRTC_NEW_PRODUCER ─────►│
   │                                │    { sessionId, producerId,   │
   │                                │      kind, callsign }         │
   │                                │                              │
   │                                │◄── WEBRTC_CONSUME ───────────│
   │                                │    { producerId,              │
   │                                │      rtpCapabilities }        │
   │                                │                              │
   │                                │─── WEBRTC_CONSUMER_CREATED ─►│
   │                                │    { consumerId, rtpParams }  │
   │                                │◄── WEBRTC_CONSUMER_RESUME ───│
   │                                │    { consumerId }             │
   │                                │── media flows ──────────────►│
```

### Message Type Definitions

```typescript
// WebRTC signaling message types (extends WsMessageType)

// Client → Server
'WEBRTC_CREATE_SESSION'     // Create a new WebRTC session
'WEBRTC_JOIN_SESSION'       // Join an existing session
'WEBRTC_GET_ROUTER_CAPS'    // Get mediasoup router RTP capabilities
'WEBRTC_CREATE_TRANSPORT'   // Create WebRTC transport (send or recv)
'WEBRTC_CONNECT_TRANSPORT'  // Connect transport (DTLS handshake)
'WEBRTC_PRODUCE'            // Start producing media (audio/video track)
'WEBRTC_CONSUME'            // Start consuming a remote producer
'WEBRTC_CONSUMER_RESUME'    // Resume a paused consumer
'WEBRTC_LEAVE_SESSION'      // Leave session gracefully

// Server → Client
'WEBRTC_SESSION_CREATED'    // Session created response
'WEBRTC_SESSION_JOINED'     // Session join confirmed
'WEBRTC_ROUTER_CAPS'        // Router RTP capabilities
'WEBRTC_TRANSPORT_CREATED'  // Transport parameters for DTLS
'WEBRTC_PRODUCED'           // Producer ID confirmation
'WEBRTC_NEW_PRODUCER'       // A new remote producer appeared (subscribe?)
'WEBRTC_CONSUMER_CREATED'   // Consumer parameters
'WEBRTC_PRODUCER_CLOSED'    // A remote producer stopped (peer left/muted)
'WEBRTC_SESSION_ENDED'      // Session terminated by server
```

---

## 6. mediasoup Configuration

```typescript
// src/webrtc/mediasoup.config.ts

export const mediasoupConfig = {
  numWorkers: process.env.MEDIASOUP_WORKER_COUNT
    ? Number(process.env.MEDIASOUP_WORKER_COUNT)
    : Math.max(1, Math.min(os.cpus().length - 2, 4)),

  workerSettings: {
    logLevel: 'warn' as MediasoupLogLevel,
    rtcMinPort: 40000,
    rtcMaxPort: 49999,
  },

  routerOptions: {
    mediaCodecs: [
      {
        kind: 'audio',
        mimeType: 'audio/opus',
        clockRate: 48000,
        channels: 2,
        parameters: {
          // Optimize for voice (not music) in low-bandwidth radio environments
          'sprop-stereo': 1,
          useinbandfec: 1,    // Forward Error Correction — critical for radio bridges
          usedtx: 1,          // Discontinuous Transmission — saves bandwidth when silent
          maxaveragebitrate: 64000,  // 64kbps max for radio
          maxplaybackrate: 48000,
        }
      }
    ] as RtpCodecCapability[]
  },

  webRtcTransportOptions: {
    listenIps: [
      {
        ip: process.env.MEDIASOUP_LISTEN_IP ?? '0.0.0.0',
        announcedIp: process.env.MEDIASOUP_ANNOUNCED_IP, // Public IP for ICE candidates
      }
    ],
    enableUdp: true,
    enableTcp: true,
    preferUdp: true,
    initialAvailableOutgoingBitrate: 300000,
    minimumAvailableOutgoingBitrate: 100000,
    maxSctpMessageSize: 262144,
  }
}
```

---

## 7. coturn Configuration

```ini
# /etc/turnserver.conf

listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
relay-ip=<server-public-ip>
external-ip=<server-public-ip>
realm=hermes.local
server-name=hermes-turn

# HMAC credentials (time-limited, generated per-session)
use-auth-secret
static-auth-secret=<generated-secret>

# TLS certificates (Let's Encrypt or self-signed for field deployments)
cert=/etc/ssl/certs/coturn.pem
pkey=/etc/ssl/private/coturn.key

# UDP port range for relaying (must match mediasoup rtcMinPort/rtcMaxPort)
min-port=49152
max-port=65535

# Security: deny TURN requests to internal networks
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.168.0.0-192.168.255.255

no-multicast-peers
fingerprint
```

### Time-Limited TURN Credentials (per-session)

```typescript
// src/webrtc/credentials.ts
// Generates short-lived TURN credentials for authenticated users

function generateTurnCredentials(userId: string, sessionId: string): TurnCredentials {
  const ttl = 3600  // 1 hour
  const username = `${Math.floor(Date.now() / 1000) + ttl}:${userId}:${sessionId}`
  const password = createHmac('sha1', process.env.TURN_SECRET!)
    .update(username)
    .digest('base64')
  return { username, password, ttl, uris: [...] }
}
```

`GET /api/v1/webrtc/credentials` requires at least the `user` role, must log every issuance to `audit_logs`, and must be rate-limited. `readonly` users may not obtain TURN relay credentials.

---

## 8. Radio-to-WebRTC Bridge (Phase 4)

The radio audio bridge enables station audio to be streamed to WebRTC participants. This is the most complex integration point.

### Architecture

```
sBitx Radio Hardware
  │ (audio output via ALSA/PulseAudio)
  │
  ▼
RadioAudioCapture (ffmpeg or GStreamer)
  │ Capture audio from radio audio device
  │ Encode as Opus (48kHz, 64kbps)
  │
  ▼
RTP Stream (loopback UDP port)
  │
  ▼
mediasoup PlainTransport
  │ Receives RTP from radio audio process
  │ Forwards to WebRTC consumers as if it were a remote peer
  │
  ▼
WebRTC Consumers
  │ (all participants in session receive radio audio)
  │
  ▼
Browser / Mobile client (hears the radio)
```

### Transmit Direction (Browser → Radio)

```
Browser microphone (WebRTC Producer)
  │
  ▼
mediasoup Consumer (PlainTransport recv)
  │ Receives decoded audio
  │
  ▼
RadioAudioPlayback (ffmpeg / ALSA)
  │ Plays audio to radio audio input
  │
  ▼ (HAL: setPtt(true))
sBitx Radio Hardware
  │ Transmits via HF radio
```

**Note**: Transmit capability requires careful coordination with HAL PTT control. PTT must be active before audio reaches the transmitter. The `RadioAudioBridge` module handles this coordination via the HAL command queue.

---

## 9. Session Lifecycle

```typescript
// src/webrtc/SessionManager.ts

interface WebRtcSession {
  id: string
  conversationId: string
  createdBy: string
  participants: Map<string, Participant>
  router: mediasoup.Router
  radioProducerId?: string    // Set when radio bridge is active
  startedAt: Date
  endedAt?: Date
}

interface Participant {
  userId: string
  callsign: string
  sendTransport?: mediasoup.WebRtcTransport
  recvTransport?: mediasoup.WebRtcTransport
  producers: Map<string, mediasoup.Producer>
  consumers: Map<string, mediasoup.Consumer>
  joinedAt: Date
}
```

### Session Cleanup

- Sessions are auto-terminated when all participants have left
- Sessions have a maximum lifetime of 4 hours (configurable)
- Disconnected participants have 60 seconds to reconnect before being removed
- All mediasoup resources are explicitly closed and garbage collected on termination

---

## 10. NAT Traversal Strategy for Field Deployments

Hermes stations operate in remote environments with challenging network conditions:

| Scenario | Strategy |
|---|---|
| Both clients on same LAN | ICE host candidates — no STUN/TURN needed |
| Clients behind different NATs | STUN to discover public IPs |
| Symmetric NAT (e.g., cellular) | TURN relay required |
| Station with no internet | coturn on local station, TURN within LAN |
| Remote station connected via HF | TURN not usable — radio bridge approach instead |

**For radio-mediated connections (no internet)**: WebRTC is not possible end-to-end. The architecture falls back to the **radio audio bridge** running server-side, streaming audio over the HF link via the existing radio transport.

---

## 11. Limitations & Future Considerations

| Limitation | Impact | Future Solution |
|---|---|---|
| mediasoup runs in same process | Resource contention under load | Extract WebRTC as separate service |
| TURN server self-hosted | Operational overhead | Deploy coturn as separate container |
| No E2E encryption | Audio flows through server | SRTP to server; E2EE requires client-side key exchange |
| Radio bridge latency | ~200-500ms radio audio delay | Acceptable for radio; noted for UX |
| No recording | Cannot archive sessions | mediasoup supports recording via PlainTransport |
| Single-region SFU | Latency for remote stations | Multi-region SFU cluster (Phase 5+) |
