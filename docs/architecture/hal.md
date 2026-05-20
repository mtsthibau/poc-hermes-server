# Hardware Abstraction Layer (HAL)

**Project**: poc-hermes-server  
**Status**: Architecture Design  
**Last Updated**: 2026-05-20

---

## 1. Problem Statement

In the current `hermes-api`, hardware communication is scattered throughout the codebase:

- `app/Functions.php` contains raw CLI wrappers called directly from controllers
- `app/Caller.php` calls CLI commands synchronously in request handlers
- `RadioController.php`, `SystemController.php` call the CLI inline
- There is no timeout handling, no retry logic, no command queuing
- Testing requires real hardware or mocking at the PHP level
- Adding a second radio type or a simulated device is impossible without forking the codebase

The HAL solves all of these problems with a clean abstraction boundary.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Application Services                          │
│    RadioService, TelemetryService, SystemService, ...            │
│                                                                  │
│    → All hardware access goes through IRadioDriver ONLY         │
│    → No service may import from src/hal/drivers/ directly       │
└────────────────────────────┬─────────────────────────────────────┘
                             │  IRadioDriver interface
                             │  (defined in src/hal/types.ts)
┌────────────────────────────▼─────────────────────────────────────┐
│                   HAL Core (src/hal/)                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ RadioCommandQueue (BullMQ)                               │   │
│  │  - Ordered command execution                             │   │
│  │  - Configurable concurrency (default: 1)                 │   │
│  │  - Retry with exponential backoff                        │   │
│  │  - Per-command timeout                                   │   │
│  │  - Priority levels                                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ TelemetryPoller                                          │   │
│  │  - Configurable poll interval (default: 1000ms)          │   │
│  │  - Change detection (only emits on state change)         │   │
│  │  - Normalized RadioTelemetryEvent output                 │   │
│  │  - Backoff on hardware errors                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ HAL Event Emitter                                        │   │
│  │  → EventBus.publish('radio.telemetry', event)            │   │
│  │  → EventBus.publish('radio.state_changed', event)        │   │
│  │  → EventBus.publish('radio.command_result', event)       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────┐  ┌──────────────────────────────┐  │
│  │   SBitxCLIDriver        │  │   SimulatedRadioDriver       │  │
│  │   (production)          │  │   (dev / test / CI)          │  │
│  │                         │  │                              │  │
│  │  Wraps: sbitx_client    │  │  In-memory state machine     │  │
│  │         ubitx_client    │  │  Configurable scenarios      │  │
│  │  via child_process      │  │  Deterministic responses     │  │
│  └─────────────────────────┘  └──────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────────────┘
                             │  CLI execution (child_process.execFile)
┌────────────────────────────▼─────────────────────────────────────┐
│                    sBitx Hardware                                │
│               sbitx_client / ubitx_client                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. IRadioDriver Interface

This is the central contract. All drivers implement this interface.

```typescript
// src/hal/types.ts

export interface RadioStatus {
  frequencyHz: number
  mode: RadioMode
  tx: boolean
  rx: boolean
  snr: number
  bitrate: number
  bytesTransmitted: number
  bytesReceived: number
  led: boolean          // system OK
  connected: boolean
  protection: boolean   // SWR protection active
  profileActiveIdx: number
  profiles: RadioProfile[]
  timeoutCounter: number
}

export interface RadioProfile {
  idx: number
  frequencyHz: number
  mode: RadioMode
  volume: number
  bfoHz: number
  digitalVoice: boolean
  powerLevel: number | null
}

export type RadioMode = 'USB' | 'LSB' | 'CW' | 'AM' | 'FM' | 'DIGITAL'

export interface RadioCommandResult<T = void> {
  success: boolean
  data?: T
  error?: RadioCommandError
}

export interface RadioCommandError {
  code: RadioErrorCode
  message: string
  raw?: string           // Raw CLI stderr/stdout for debugging
}

export type RadioErrorCode =
  | 'COMMAND_TIMEOUT'
  | 'COMMAND_FAILED'
  | 'INVALID_PARAMETER'
  | 'HARDWARE_UNAVAILABLE'
  | 'PERMISSION_DENIED'

export interface IRadioDriver {
  // Lifecycle
  initialize(): Promise<void>
  shutdown(): Promise<void>
  isAvailable(): Promise<boolean>

  // Status & Telemetry
  getStatus(profileIdx?: number): Promise<RadioCommandResult<RadioStatus>>

  // Frequency Control
  getFrequency(profileIdx: number): Promise<RadioCommandResult<number>>
  setFrequency(hz: number, profileIdx: number): Promise<RadioCommandResult>

  // Mode Control
  setMode(mode: RadioMode, profileIdx: number): Promise<RadioCommandResult>

  // Profile Management
  getActiveProfile(): Promise<RadioCommandResult<number>>
  setActiveProfile(profileIdx: number): Promise<RadioCommandResult>
  restoreDefaults(profileIdx: number): Promise<RadioCommandResult>

  // Volume & Audio
  getVolume(): Promise<RadioCommandResult<number>>
  setVolume(level: number): Promise<RadioCommandResult>

  // BFO
  getBfo(profileIdx: number): Promise<RadioCommandResult<number>>
  setBfo(hz: number, profileIdx: number): Promise<RadioCommandResult>

  // PTT & Transmission
  setPtt(active: boolean, profileIdx: number): Promise<RadioCommandResult>
  stopTransmission(): Promise<RadioCommandResult>

  // Power Level
  getPowerLevel(profileIdx: number): Promise<RadioCommandResult<number | null>>
  setPowerLevel(level: number): Promise<RadioCommandResult>

  // Protection & Safety
  getProtection(profileIdx: number): Promise<RadioCommandResult<boolean>>
  resetProtection(profileIdx: number): Promise<RadioCommandResult>
  setRefThreshold(value: number, profileIdx: number): Promise<RadioCommandResult>
  getRefThreshold(profileIdx: number): Promise<RadioCommandResult<number>>

  // System
  getSNR(): Promise<RadioCommandResult<number>>
  getBitrate(): Promise<RadioCommandResult<number>>
  setMasterCal(hz: number, profileIdx: number): Promise<RadioCommandResult>
  setConnectionStatus(connected: boolean, profileIdx: number): Promise<RadioCommandResult>

  // Voice
  restartVoiceTimeout(): Promise<RadioCommandResult>
  getVoiceTimeoutConfig(): Promise<RadioCommandResult<number>>
  setVoiceTimeoutConfig(seconds: number): Promise<RadioCommandResult>

  // Tone
  setTone(params: string): Promise<RadioCommandResult>

  // Digital Voice
  setDigitalVoice(enabled: boolean, profileIdx: number): Promise<RadioCommandResult>

  // SD Card
  eraseSDCard(): Promise<RadioCommandResult>
}
```

---

## 4. SBitxCLIDriver — Production Driver

Maps each `IRadioDriver` method to the corresponding `sbitx_client` or `ubitx_client` CLI command.

```typescript
// src/hal/drivers/sbitx/SBitxCLIDriver.ts

import { execFile } from 'node:child_process'
import { promisify } from 'node:util'
import type { IRadioDriver, RadioCommandResult, RadioStatus } from '../../types'

const execFileAsync = promisify(execFile)

export class SBitxCLIDriver implements IRadioDriver {
  private readonly cliPath: string
  private readonly timeoutMs: number

  constructor(config: SBitxConfig) {
    this.cliPath = config.cliPath           // Path to sbitx_client binary
    this.timeoutMs = config.commandTimeoutMs ?? 5000
  }

  private async exec(
    command: string,
    args: string[] = []
  ): Promise<RadioCommandResult<string>> {
    try {
      const { stdout, stderr } = await execFileAsync(
        this.cliPath,
        [command, ...args],
        {
          timeout: this.timeoutMs,
          // CRITICAL: Never use shell: true — prevents shell injection
          // Arguments are passed as array, not string concatenation
        }
      )
      return { success: true, data: stdout.trim() }
    } catch (err: unknown) {
      const error = err as NodeJS.ErrnoException
      if (error.signal === 'SIGTERM' || error.code === 'ETIMEDOUT') {
        return {
          success: false,
          error: { code: 'COMMAND_TIMEOUT', message: `Command ${command} timed out` }
        }
      }
      return {
        success: false,
        error: {
          code: 'COMMAND_FAILED',
          message: error.message,
          // raw stderr intentionally not exposed to callers above HAL boundary
        }
      }
    }
  }

  async setFrequency(hz: number, profileIdx: number): Promise<RadioCommandResult> {
    // Validates parameter range before CLI execution
    if (hz < 1_000_000 || hz > 30_000_000) {
      return { success: false, error: { code: 'INVALID_PARAMETER', message: 'Frequency out of HF range' } }
    }
    return this.exec('set_frequency', [hz.toString(), profileIdx.toString()])
  }

  // ... all other methods follow the same pattern
}
```

### CLI Command Mapping

| IRadioDriver Method | CLI Command | Arguments |
|---|---|---|
| `getFrequency(profileIdx)` | `get_frequency` | `<profile>` |
| `setFrequency(hz, profileIdx)` | `set_frequency` | `<hz> <profile>` |
| `setMode(mode, profileIdx)` | `set_mode` | `<mode> <profile>` |
| `getActiveProfile()` | `get_profile` | — |
| `setActiveProfile(idx)` | `set_profile` | `<profile>` |
| `getVolume()` | `get_volume` | — |
| `setVolume(level)` | `set_volume` | `<level>` |
| `getBfo(profileIdx)` | `get_bfo` | `<profile>` |
| `setBfo(hz, profileIdx)` | `set_bfo` | `<hz> <profile>` |
| `setPtt(active, profileIdx)` | `set_ptt` | `<0\|1> <profile>` |
| `stopTransmission()` | `stop` | — |
| `getPowerLevel(profileIdx)` | `get_power_level` | `<profile>` |
| `setPowerLevel(level)` | `set_power_level` | `<level>` |
| `getProtection(profileIdx)` | `get_protection` | `<profile>` |
| `resetProtection(profileIdx)` | `reset_protection` | `<profile>` |
| `getSNR()` | `get_snr` | — |
| `getBitrate()` | `get_bitrate` | — |
| `getStatus(profileIdx?)` | Multiple CLI calls, normalized | — |
| `eraseSDCard()` | `erase_sdcard` | — |
| `restartVoiceTimeout()` | `restart_voice_timeout` | — |

### Security: CLI Execution Safety

**CRITICAL**: All arguments are passed as arrays to `execFile`, never concatenated into shell strings.

- `execFile` does NOT invoke a shell — no shell injection possible
- All numeric parameters are validated before CLI invocation
- String parameters are allowlisted (modes, status values)
- Raw CLI stderr is never propagated above the HAL boundary
- CLI binary path is read from configuration (not user input)

---

## 5. SimulatedRadioDriver — Development & Test Driver

```typescript
// src/hal/drivers/simulated/SimulatedRadioDriver.ts

export class SimulatedRadioDriver implements IRadioDriver {
  private state: SimulatedRadioState = DEFAULT_RADIO_STATE
  private scenarios: SimulationScenario[] = []

  // Supports configurable scenarios for testing:
  // - High SNR / low SNR
  // - PTT active / inactive
  // - Protection triggered
  // - Connection lost / restored
  // - Command timeout simulation
  // - Profile switching

  withScenario(scenario: SimulationScenario): this {
    this.scenarios.push(scenario)
    return this
  }

  async setFrequency(hz: number, profileIdx: number): Promise<RadioCommandResult> {
    await this.simulateLatency(50, 200)  // Simulate CLI latency
    this.state.profiles[profileIdx].frequencyHz = hz
    return { success: true }
  }

  // All methods maintain consistent in-memory state
  // Telemetry polling returns realistic random variation (SNR ±2, etc.)
}
```

This driver is automatically selected when `RADIO_DRIVER=simulated` in `.env`, enabling full development and testing without hardware.

---

## 6. RadioCommandQueue

Commands are queued through BullMQ for ordering, retry, and observability.

```typescript
// src/hal/RadioCommandQueue.ts

export class RadioCommandQueue {
  private queue: Queue
  private worker: Worker

  constructor(driver: IRadioDriver, redis: Redis) {
    this.queue = new Queue('radio-commands', { connection: redis })
    this.worker = new Worker('radio-commands', async (job) => {
      return this.executeCommand(driver, job.data)
    }, {
      connection: redis,
      concurrency: 1,           // Serial execution — only 1 command at a time
      limiter: {
        max: 100,
        duration: 60_000,       // Max 100 commands per minute
      }
    })
  }

  async enqueue<T>(
    command: RadioCommand,
    options?: { priority?: number; timeout?: number }
  ): Promise<RadioCommandResult<T>> {
    const job = await this.queue.add(command.type, command, {
      priority: options?.priority ?? 5,
      attempts: 3,
      backoff: { type: 'exponential', delay: 500 },
      removeOnComplete: { age: 3600 },    // Keep completed jobs 1 hour for debugging
      removeOnFail: { age: 86400 },       // Keep failed jobs 24 hours
    })
    return job.waitUntilFinished(this.queueEvents)
  }
}
```

**Priority levels:**
- `1`: Critical (e.g., stop transmission, reset protection)
- `3`: High (e.g., PTT, mode change during active session)
- `5`: Normal (e.g., frequency set, volume)
- `8`: Low (e.g., calibration, non-urgent config)

---

## 7. TelemetryPoller

```typescript
// src/hal/TelemetryPoller.ts

export class TelemetryPoller {
  private intervalMs: number
  private lastState: RadioStatus | null = null

  async start(driver: IRadioDriver, eventBus: EventBus): Promise<void> {
    setInterval(async () => {
      const result = await driver.getStatus()
      if (!result.success) {
        // Emit a radio.error event and backoff
        eventBus.publish('radio.error', { error: result.error })
        return
      }

      const status = result.data!

      // Always emit telemetry for display updates
      eventBus.publish('radio.telemetry', { status, timestamp: new Date() })

      // Only emit state_changed when something significant changed
      if (this.hasSignificantChange(this.lastState, status)) {
        eventBus.publish('radio.state_changed', { previous: this.lastState, current: status })
      }

      this.lastState = status
    }, this.intervalMs)
  }

  private hasSignificantChange(prev: RadioStatus | null, curr: RadioStatus): boolean {
    if (!prev) return true
    return (
      prev.tx !== curr.tx ||
      prev.rx !== curr.rx ||
      prev.connected !== curr.connected ||
      prev.protection !== curr.protection ||
      prev.profileActiveIdx !== curr.profileActiveIdx ||
      Math.abs(prev.frequencyHz - curr.frequencyHz) > 100
    )
  }
}
```

---

## 8. Configuration

```typescript
// Environment variables

RADIO_DRIVER=sbitx              // 'sbitx' | 'simulated'
SBITX_CLI_PATH=/usr/local/bin/sbitx_client
SBITX_COMMAND_TIMEOUT_MS=5000
RADIO_TELEMETRY_INTERVAL_MS=1000
RADIO_COMMAND_QUEUE_CONCURRENCY=1
```

---

## 9. Future Extensibility

The HAL is explicitly designed for future extension:

| Future Scenario | Extension Point |
|---|---|
| Support uBitx radio | Implement `UBitxCLIDriver extends IRadioDriver` |
| Remote radio node | Implement `RemoteRadioDriver` (HTTP or gRPC to remote station) |
| Multi-radio station | `RadioDriverRegistry` managing multiple `IRadioDriver` instances |
| Virtual/SDR radio | Implement `SdrDriver` (SoapySDR or similar) |
| Radio simulation cluster | `SimulatedRadioDriver` with network-synchronized state |

No application service above the HAL needs to change when a new driver is added. The driver is selected via configuration.
