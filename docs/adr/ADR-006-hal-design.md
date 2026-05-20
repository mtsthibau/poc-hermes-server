# ADR-006: Hardware Abstraction Layer (HAL) Design

**Status**: Accepted  
**Date**: 2026-05-20  
**Authors**: Platform Architecture Team  
**Related**: [hal.md](../architecture/hal.md)

---

## Context

The existing Hermes API calls `sbitx_client` CLI commands directly inside HTTP controllers:

```php
// Actual pattern in hermes-api controllers:
shell_exec("sbitx_client set_frequency {$freq}");
exec("sbitx_client set_mode " . $request->mode);
```

This creates 9 identified problems:

1. **No abstraction**: Radio logic is scattered across 8+ controllers
2. **CLI injection risk**: User input concatenated directly into shell commands
3. **No command serialization**: Multiple simultaneous requests may send conflicting commands
4. **No error handling contract**: `shell_exec` returns null on failure, controllers ignore it
5. **No testing isolation**: Cannot test business logic without hardware present
6. **No simulation mode**: Developers need physical radio hardware to test anything
7. **No telemetry abstraction**: Polling is duplicated in multiple controllers
8. **Hardcoded CLI path**: No configuration for different hardware variants
9. **No retry logic**: Transient CLI failures return 500 errors immediately

---

## Decision

Implement a **Hardware Abstraction Layer (HAL)** with:

- `IRadioDriver` interface defining all radio operations
- `SBitxCLIDriver` implementing `IRadioDriver` using `execFile` with array args
- `SimulatedRadioDriver` for development and testing (no hardware required)
- `RadioCommandQueue` (BullMQ) for serial command execution
- `TelemetryPoller` for periodic state polling and change detection

---

## Rationale

### Why a Typed Interface (`IRadioDriver`)

The interface provides a stable contract between application services and hardware:

```typescript
interface IRadioDriver {
  getStatus(): Promise<RadioStatus>
  setFrequency(hz: number, profileIdx: number): Promise<void>
  setMode(mode: RadioMode, profileIdx: number): Promise<void>
  setVolume(level: number): Promise<void>
  setPTT(active: boolean): Promise<void>
  // ... all other radio operations
}
```

This enables:
- Swapping `SBitxCLIDriver` → `SimulatedRadioDriver` without touching any service code
- Adding a `UBitxCLIDriver` or `RemoteRadioDriver` without changing callers
- Unit testing all radio-dependent services with the simulated driver
- Clean error propagation: all methods return typed results or throw typed errors

### Why `execFile` with Array Arguments

Security is non-negotiable:

```typescript
// SAFE: frequency is validated as integer, passed as separate arg
await execFile(this.cliPath, ['set_frequency', String(hz), String(profileIdx)])

// REJECTED: shell injection possible if hz contains shell metacharacters
exec(`${this.cliPath} set_frequency ${hz}`)
```

`execFile` does not invoke a shell — arguments are passed directly to the subprocess as separate argv entries. Shell metacharacters in arguments have no effect.

All numeric parameters are validated (range check) before being passed as strings. All string parameters (mode, profile name) are matched against an explicit allowlist before CLI invocation.

### Why a Command Queue (BullMQ)

The sBitx radio hardware accepts one command at a time. Sending concurrent commands causes undefined behavior:

- A PTT command during a frequency change may corrupt the radio state
- The CLI returns success before the hardware has settled

The `RadioCommandQueue` (BullMQ, concurrency=1) ensures serial execution:

```
[Operator 1: set_frequency 14.200MHz]
[Operator 2: set_mode USB      ]  → queued, waiting
[WebRTC: activate PTT          ]  → queued, waiting

Execution order guaranteed:
1. set_frequency → complete
2. set_mode → complete
3. activate PTT → complete
```

Priority levels prevent low-priority operations from blocking urgent commands:

| Priority | Level | Examples |
|---|---|---|
| Critical | 1 | Emergency stop, reset |
| High | 3 | PTT on/off (time-sensitive) |
| Normal | 5 | Frequency, mode, volume changes |
| Low | 8 | Profile saves, telemetry |

### Why a Simulated Driver

Development without hardware access is critical for:
- CI/CD pipeline tests (no radio hardware on GitHub Actions runners)
- Frontend developer testing without radio setup
- Unit tests of all HAL-dependent business logic
- Scenario simulation (SWR protection triggered, low SNR, radio disconnected)

The `SimulatedRadioDriver` maintains in-memory state and simulates realistic delays:

```typescript
class SimulatedRadioDriver implements IRadioDriver {
  private state: RadioStatus = defaultSimulatedState
  private scenarios: SimulationScenario = {}

  async setFrequency(hz: number): Promise<void> {
    await delay(50)  // Simulate CLI round-trip
    this.state.frequencyHz = hz
    // Optionally trigger scenario: protection_triggered, etc.
  }
}
```

---

## Consequences

### Positive

- All radio CLI calls go through one place — security is enforced consistently
- Hardware can be replaced (uBitx, SDR, remote radio) without touching business logic
- All services testable without hardware
- Command serialization prevents radio state corruption
- Telemetry polling is centralized — no duplicate polling across controllers
- Error handling is typed and consistent

### Negative

- Command queue introduces latency for radio commands (typically <100ms overhead)
- Queue depth can grow if many commands are submitted rapidly (bounded by priority)
- SimulatedDriver must be kept in sync with real driver as new operations are added

### Mitigations

- Priority queue ensures PTT (priority 3) is never blocked by low-priority operations
- Queue max depth is configurable with overflow rejection policy
- Driver interface is tested for both implementations (contract tests)

---

## Alternatives Considered

- **Direct CLI calls with better error handling**: Insufficient — still no injection protection, no serialization, no simulation
- **Unix socket / IPC to sbitx daemon**: Possible but requires changes to the existing C daemon; deferred to future (sbitx already provides a CLI interface)
- **Separate microservice for HAL**: Over-engineering — HAL is a local hardware concern, not a distributed service. Adds network hop and deployment complexity without benefit
- **D-Bus interface**: Platform-specific, not portable to future hardware variants

---

## Related Documents

- [HAL Architecture](../architecture/hal.md)
- [Security Architecture](../architecture/security.md) — CLI injection prevention
- [Event-Driven Backend](../architecture/event-driven.md) — HAL events
