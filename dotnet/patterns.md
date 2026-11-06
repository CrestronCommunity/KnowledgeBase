# Crestron .NET 8 + HTML Control System — Reusable Patterns

> **Cross-project reference.** Covers patterns found across three projects:
> `solventum` (Canteen), `st-pius-x` (Treacy), `unsw` (L5 Level 3 East).
> Use this when starting a new project to find canonical source files to copy from.

---

## Stack Summary

| Layer | Approach |
|---|---|
| Runtime | Crestron SimplSharpPro (.NET 8) |
| Entry point | `ControlSystem : CrestronControlSystem` (Host project) |
| Dev entry point | `Program.cs` console app (DevHost project) |
| Transport | `HttpListener`-based WebSocket server + static file host |
| Frontend | Single `index.html` (simple) or TypeScript modules (complex) |
| DI | `Microsoft.Extensions.DependencyInjection` via `CompositionRoot` |
| Tests | xUnit + Moq |

---

## Project Layout Convention

Every project follows the same assembly structure:

```
src/
  <Namespace>.Contracts/        interfaces, wire protocol, config models
  <Namespace>.Control.Core/     business logic (no I/O)
  <Namespace>.Control.Drivers/  device driver implementations + stubs
  <Namespace>.Control.Transport/ WebSocket server, static file host, protocol
  <Namespace>.Control.Composition/ CompositionRoot
  <Namespace>.Control.Host/     Crestron entry point (ControlSystem.cs)
  <Namespace>.Control.DevHost/  console dev entry point (Program.cs)
tests/
webapp/                         HTML + JS/TS frontend
```

Canonical source: `c:\repos\solventum\canteen\src\` (cleanest, most recent).

---

## High-Priority Reusable Components

These appear in 2–3 projects with near-identical implementations. Copy from the canonical source listed; adapt the namespace and config values.

---

### 1. WebSocket Server

**What:** `HttpListener`-based WebSocket server. Manages concurrent client connections, reads/writes framed text messages, raises events for connect/disconnect/message.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Transport\WebSocketServer.cs`

**Key characteristics:**
- `ConcurrentDictionary<string, WebSocket>` for client tracking
- Async accept loop with `CancellationToken` shutdown
- Exposes `OnMessage`, `OnConnected`, `OnDisconnected` events
- `BroadcastAsync(string json)` for pushing state to all clients

**Treacy variant:** `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Transport\WebSocketServer.cs` — identical pattern; also includes `AdminHandler.cs` for a separate HTTP admin API endpoint.

---

### 2. Static File Host

**What:** Serves `webapp/` files over HTTP from the same `HttpListener`. Maps file extensions to MIME types, validates paths to prevent traversal.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Transport\StaticFileHost.cs`

**Key characteristics:**
- Takes `IFileSystem` abstraction (testable)
- MIME type map covers: html, css, js, mjs, json, png, jpg, gif, webp, svg, ico, woff, woff2, ttf, otf, txt, xml, map
- 404 on missing files, 403 on path traversal attempts
- `webRoot` path injected at construction

---

### 3. Protocol (Wire Format)

**What:** Sits between `WebSocketServer` and application logic. Serializes/deserializes `RoomState` to JSON; sends full state on connect, JSON Patch diffs on update.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Transport\Protocol.cs`

**Key characteristics:**
- Full state on new client connect
- JSON Patch (`application/json-patch+json`) diffs on subsequent updates (reduces payload)
- Envelope types: `StateEnvelope`, `ActionEnvelope`, `ErrorEnvelope` — defined in Contracts
- `JsonOptions.cs` in Contracts defines serializer settings (camelCase, ignore nulls)

---

### 4. State Store

**What:** Thread-safe in-memory store for `RoomState`. Notifies subscribers on change with sequence numbers to detect missed updates.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Core\State\StateStore.cs`

**Key characteristics:**
- `lock` on `_state` for thread safety
- `Update(Func<RoomState, RoomState> mutate)` — functional mutation
- `Subscribe(Action<RoomState, long> handler)` — sequence number in callback
- Treacy adds `ApplyPatch(JsonPatch)` for patch-based updates

**UNSW variant:** `InMemoryStateStore.cs` — simpler, uses `event StateChanged` instead of delegate subscriptions, no sequence numbers. Good for simpler projects.

---

### 5. Action Dispatcher

**What:** Registry mapping action-name strings to handler delegates. Decouples frontend commands from implementation.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Core\Actions\ActionDispatcher.cs`

**Key characteristics:**
- `Register(string action, Func<JsonElement, Task<ActionResult>> handler)`
- `DispatchAsync(string action, JsonElement payload)` — returns `ActionResult` (success/error)
- `ActionResult.cs` — simple value type: `bool Success`, `string? Error`
- Handlers registered in `RoomController` constructor

**Treacy variant:** Adds `ModeMachine` validation — actions are validated against current mode before dispatch. Copy from `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Core\Modes\` if the project has multiple operating modes.

---

### 6. PIN Service

**What:** Verifies PIN with lockout after repeated failures. Persists lockout state across restarts.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Core\Pin\PinService.cs`

**Key characteristics:**
- PBKDF2 hashing via `PinHasher.cs` (in Contracts)
- 5 failed attempts → 5-minute lockout
- Lockout state persisted to `state/lockout.json` (survives restart)
- `LockoutChanged` event for UI feedback
- Thread-safe via `lock`

**Treacy variant:** Also supports BCrypt (legacy migration path) and interactive buffered PIN entry. See `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Core\Pin\PinService.cs`.

**UNSW variant:** `PinHasher.cs` — BCrypt-only static utility, no lockout. Sufficient when lockout is not required.

---

### 7. Config Management Stack

**What:** Load, watch, and write JSON config files with hot-reload support.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Core\Config\`

Files:
- `ConfigLoader.cs` — deserializes `appsettings.json` into `LoadedConfig`
- `ConfigWatcher.cs` — `FileSystemWatcher` wrapper; fires `ConfigChanged` event on save
- `ConfigWriter.cs` — writes config back to disk (used by admin UI)
- `IFileSystem.cs` / `LocalFileSystem.cs` — abstraction over `System.IO` (testable)
- `IClock.cs` / `SystemClock.cs` — abstraction over `DateTime.UtcNow` (testable)

**Treacy variant:** Adds `ConfigValidator.cs` — validates config schema and returns `ValidationResult`. Copy from `c:\repos\st-pius-x\src\Treacy.Contracts\ConfigSchema\ConfigValidator.cs` when config is complex.

---

### 8. Inactivity Monitor

**What:** Fires a configurable action after a period of no user activity. Resets on any user interaction.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Core\Room\InactivityMonitor.cs`

**Key characteristics:**
- `System.Threading.Timer` internally
- `Reset()` called from action dispatcher on any user action
- Timeout duration from config
- On timeout: dispatches a configured action (e.g. `"power.off"`)

---

### 9. Device Status Poller

**What:** Periodically queries device status and pushes updates into `StateStore`.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Core\Room\DeviceStatusPoller.cs`

**Key characteristics:**
- `PeriodicTimer` (one timer per device, configurable interval)
- `Interlocked` guard prevents overlapping polls
- Updates state via `StateStore.Update()`
- Implements `IAsyncDisposable` — cancels and awaits in-flight polls on shutdown

---

### 10. Composition Root

**What:** Central DI wiring. One static `Build()` method returns a configured `ServiceProvider`.

**Canonical source:** `c:\repos\solventum\canteen\src\Solventum.Control.Composition\CompositionRoot.cs`

**Key characteristics:**
- `Build(string configDir, string? webRoot, ILogger? logger)` factory signature
- Registers: config stack, state store, action dispatcher, all drivers, transport
- `webRoot` is `null` in Host (Crestron serves from its own path), set in DevHost
- Treacy adds second `Build()` overload for the Telemetry subsystem

---

## Medium-Priority Reusable Patterns

These are proven patterns worth knowing about; copy when the project warrants the complexity.

---

### Mode Machine (Treacy)

**What:** Explicit state machine for room modes (e.g. Idle → Presentation → Conferencing). Actions validated against current mode before dispatch.

**Source:** `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Core\Modes\ModeMachine.cs`

**When to use:** Projects with 3+ distinct operating modes where some actions are invalid in certain states.

---

### Coordinator Pattern (Treacy)

**What:** Decomposes `RoomController` responsibilities into single-concern coordinators (Audio, Video, Camera, Pin, Recording, etc.). Each coordinator owns one domain and subscribes to state/events independently.

**Source:** `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Core\Rooms\Coordinators\`

**When to use:** Projects with many devices and complex interaction rules. Avoids a monolithic `RoomController`.

---

### Admin PIN Middleware (UNSW)

**What:** ASP.NET middleware handling first-boot PIN setup and bearer-token authentication for the admin UI.

**Source:** `c:\repos\unsw\L5 Level 3 East\src\UI\AdminPinMiddleware.cs`

**When to use:** When using ASP.NET Core as the web server (not `HttpListener`). Provides first-boot flow where admin sets PIN on first access.

---

### IPC Protocol (Treacy)

**What:** Newline-delimited JSON over a named pipe or TCP socket. Used to communicate between Control and Telemetry subsystems running as separate Crestron programs.

**Source:** `c:\repos\st-pius-x\src\Treacy.Telemetry\Treacy.Telemetry.Core\Ipc\`, `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Core\Ipc\`

**When to use:** Projects split into two Crestron programs (e.g. control + telemetry/monitoring).

---

### TypeScript Frontend Pattern (UNSW)

**What:** Modular TypeScript frontend with separate files for state management, API communication, and UI components (screens, modals).

**Source:** `c:\repos\unsw\L5 Level 3 East\webapp\scripts\`

Key files: `api.ts` (HTTP + WebSocket), `state.ts` (client-side state), `fire.ts` (event library), individual screen/modal modules.

**When to use:** Projects with complex UIs requiring multiple screens or modal workflows. Overkill for simple single-screen panels.

---

## Device Driver Catalog

Interface-based driver pattern: define a port interface in Contracts, implement in Drivers, provide a `Stub*` in Drivers for tests.

| Device | Interface | Driver | Source |
|---|---|---|---|
| Epson projector (ESC/VP.net TCP) | `IProjector` | `EpsonProjectorDriver` | `c:\repos\solventum\canteen\src\Solventum.Control.Drivers\Epson\` |
| Grandview motorised screen (HTTP) | `IScreen` | `GrandviewScreenDriver` | `c:\repos\solventum\canteen\src\Solventum.Control.Drivers\Grandview\` |
| Crestron AM-3200 AirMedia | `IAirMedia` | `CrestronAirMediaDriver` | `c:\repos\solventum\canteen\src\Solventum.Control.Drivers\AirMedia\` |
| Samsung MDC display | `IDisplayPower` | `SamsungMdcDisplay` | `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Drivers\Displays\` |
| HTTP-controlled display | `IDisplayPower` | `HttpDisplay` | `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Drivers\Displays\` |
| Crestron NVX video router | `IVideoRouter` / `INvxPollingControl` | `NvxVideoRouter`, `HttpNvxControlPlane` | `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Drivers\Nvx\` |
| Lumens document camera | `ILumensControl` | `LumensDriver` | `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Drivers\Lumens\` |
| PTZ camera (VISCA) | `ICameraController` | `ViscaPtzDriver` | `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Drivers\Ptz\` |
| Biamp Tesira DSP (QRC) | `IDspControl` / `IMatrixMixer` | `QSysDspDriver` + `QrcClient` | `c:\repos\st-pius-x\src\Treacy.Control\Treacy.Control.Drivers\QSys\` |
| Shure ULXD wireless mics | `IWirelessMicSystem` | `ShureUlxdDriver` | `c:\repos\st-pius-x\src\Treacy.Telemetry\Treacy.Telemetry.Drivers\Wireless\Shure\` |
| Network monitor (SNMP) | `INetworkMonitor` | `SnmpNetworkMonitor` | `c:\repos\st-pius-x\src\Treacy.Telemetry\Treacy.Telemetry.Drivers\Snmp\` |
| TCP channel abstraction | — | `TcpChannel` / `ITcpChannel` | `c:\repos\solventum\canteen\src\Solventum.Control.Drivers\Tcp\` |

---

## Test Utilities

Reusable test helpers found in Treacy (most comprehensive test suite):

| Utility | Purpose | Source |
|---|---|---|
| `FakeClock` | Controllable `IClock` for time-sensitive tests | `c:\repos\st-pius-x\src\Treacy.Tests\` |
| `FakeFileSystem` | In-memory `IFileSystem` for config tests | `c:\repos\st-pius-x\src\Treacy.Tests\` |
| `EventCollector<T>` | Collects async events for assertion | `c:\repos\st-pius-x\src\Treacy.Tests\` |
| `Stub*` drivers | One per device port interface | Drivers project in each repo |

---

## Starting a New Project

Suggested order for bootstrapping project 4:

1. Copy project layout from `c:\repos\solventum\canteen\` — rename namespace throughout
2. Copy Transport layer (WebSocketServer, StaticFileHost, Protocol) — no changes needed
3. Copy Config stack (ConfigLoader, ConfigWatcher, ConfigWriter, IFileSystem, IClock)
4. Copy StateStore and ActionDispatcher
5. Copy PinService (+ PinHasher from Contracts)
6. Copy CompositionRoot — wire new drivers as you add them
7. Copy InactivityMonitor if the room needs auto-poweroff
8. Copy DeviceStatusPoller if devices need periodic status polling
9. Write new port interfaces and drivers for project-specific devices
10. Adapt `RoomController` to register project-specific actions
11. Build frontend from `webapp/index.html` in Solventum (simple) or UNSW TypeScript pattern (complex)
