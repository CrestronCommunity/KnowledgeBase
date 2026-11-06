# Crestron SIMPL# Pro on .NET 6/8 — Lessons Learnt

Nov 5, 2026 · @Rod Driscoll

## About this document

The short version: use .NET 8 for new SIMPL# Pro (SSP) programs, keep .NET Framework 4.7.2 for anything SIMPL+ has to load, and prefer standard .NET classes over Crestron's wrappers wherever the control subnet doesn't force your hand.

This is a distillation of the #simplsharp-net6-net8 channel on the CrestronProfessionals Discord, from its creation on 29 Feb 2024 to 23 Oct 2026 (about 3,000 messages). It covers what practitioners actually hit in the field, not Crestron's official position.

- Contributors quoted most often: Troy Garner (TBD Enterprises), Andrew Welker (PepperDash), Niklas Westman (JaDeVa), Jeff McAleer (AVI-SPL), Scott Pidzarko (ClearAlan), Simon Tuffley (River Controls), and Crestron staff Yuri Staal and James Battaglia.
- Firmware behaviour changes often. Where a lesson is tied to a firmware version, the version is given; re-test on your current build.
- Community workarounds are not Crestron-supported. TrueBlue support for C# is limited.

## Quick reference: what runs where

Pick the target framework by what will load the code, not by preference. SIMPL+ is the hard constraint.

| Use case | Target | Runtime on box | Notes |
| --- | --- | --- | --- |
| SIMPL# library loaded by SIMPL+ (clz) | net472 (or net35 CF for 3-Series) | Mono | The S+ manager runs .NET Framework 4.7. A net6/8 clz fails with "dependency on SimplSharpHelperInterface 2.21.x … system is using 1.0.0.0". |
| SSP program on 4-Series appliance | net8.0 (recommended) | .NET 8 | Needs firmware 2.8005+ (late 2024). net6 worked from 2.8003.00037 (Mar 2024). net7 is not supported. |
| SSP program on VC-4 | net8.0 or net472 | .NET 8 / Mono | VC-4 was net472-only until the .NET 8 firmware; it now runs both. |
| Crestron Certified Drivers (CCD / RAD SDK) | net472 | Mono | Driver packages restore as .NET Framework (NU1701). Put CCDs in a separate net472 program slot, or try the 2026 `Crestron.SimplSharp.SDK.Compatibility` package. |
| ASP.NET Core, Kestrel, SignalR, Blazor | net8.0 | .NET 8 + ASP.NET Core runtime | Runtime shipped on 2.8005; absent on .NET 6 firmware. |
| Code shared between SIMPL+ and SSP | net472;net8.0 multi-target | both | S+ consumes the net472 output. Expect `#if` directives and polyfills. |

Migration cost: net6 → net8 is a one-line retarget on an SDK-style project. net472 old-style → net8 means converting the project first (see next section). Mono-only behaviour (registry keys on Linux, lax paths) disappears on .NET 8.

## Project setup, build and packaging

Use SDK-style projects everywhere, and expect to own part of the build because Crestron's NuGet targets have repeatedly lagged.

**Project format**

- Never start from the legacy .NET Framework project template. An SDK-style csproj can target net472 and get modern C#, nullable reference types and easy retargeting. Convert old projects with the `dotnet upgrade-assistant` CLI.
- Upgrade-assistant can strip package references and leave dangling `using` directives (e.g. Newtonsoft); review the diff.
- Put solution-wide settings and warning suppressions in `Directory.Build.props` / `Directory.Build.targets`, and code style in `.editorconfig`, rather than `#pragma` in every file.
- Multi-target with `<TargetFrameworks>net472;net8.0</TargetFrameworks>`. Use net472, not net48. Reference the Crestron NuGet packages, not loose DLLs.
- Useful tools: `dotnet-outdated`, `dotnet nuget why` (why a transitive package is present). Keep the dependency tree small.

**Crestron NuGet / MSBuild quirks**

- The `SimplSharpPostProcess` targets broke multi-targeting for most of 2024. Workaround: override them with empty targets (`SimplSharpPostProcess`, `SimplSharpPostProcess47`) in Directory.Build.targets and build the cpz yourself. Troy Garner's pinned template in the channel does this.
- Their task-factory condition compared framework strings instead of using `$(MSBuildRuntimeType)`. It was finally fixed by 2026, but still produces "unreachable code" warnings in multi-project solutions.
- On net8 the `.Program` package does not generate `ControlSystem.cs`. Write it yourself.
- A .NET 8 cpz needs `SimplSharpPro.runtimeconfig.json`. Missing it gives `Cannot open assembly '/simpl/app01/SimplSharpPro.exe'`. As of Oct 2026 the Program package still lacks a proper net8 target whose runtimeconfig points at the 8.0 runtime; check `Environment.Version` at startup to confirm you are really on .NET 8, not 6.
- Editing the runtimeconfig inside the package is pointless: it is overwritten on every package update. Use your own targets.
- `SimplSharpPro.exe` checks for `SimplSharpData.dat`; a 0-byte file satisfies it if you package by hand.
- From 2.21.54 the net6+ packages dropped `SimplSharpNewtonsoft`, `SimplSharpExchangeWebServices` and `SimplSharpReflectionInterface`. RAD SDK DLLs still reference the last one — add `Crestron.SimplSharp.SDK.Compatibility` (released Oct 2026).
- The net472 packages still ship `SimplSharpNewtonsoft`, which clashes with real Newtonsoft. Alias it out with a target that sets `<Aliases>doNotUse</Aliases>` on that `ReferencePath` (pinned in the channel), or use `extern alias`.
- Crestron's package signing certificate chains to Entrust, which Linux trust stores distrusted in late 2024. `dotnet restore` / `nuget verify` then fails in GitHub Actions and devcontainers; build on Windows runners.
- Stale Crestron packages cause odd runtime errors: `InvalidCastException` from `DmNvxE30` to `DmNvxBaseClass` or `Ts1070` to `TsX70Base`, and `Could not load Microsoft.AspNetCore 8.0.0`. Update every project to the same package version.

**Third-party packages**

- On .NET 6 firmware, newer packages failed to load because the box lacked their dependencies (`System.Diagnostics.DiagnosticSource` 7.0 from new Serilog, `System.Text.Json` 8.0 from `Serilog.Settings.Configuration`). Pin versions to what the on-box runtime provides.
- "Unable to locate System.Private.CoreLib.XmlSerializers.dll" messages at load are informational noise.
- Packages with native code (SQLite) need a runtime identifier. Appliances run a 32-bit `linux-arm` process on an arm64 OS; VC-4 is `linux-x64`.
- Blazor: add an MSBuild target that copies `wwwroot` and the scoped CSS bundle (`<Project>.styles.css`) into the output before `SimplSharpPostProcess` packs the cpz. Component libraries such as MudBlazor/Radzen need extra csproj work for `_content` assets.
- Rider can publish straight to the processor over SFTP; then just restart the program.

## SIMPL+ / SIMPL# module interop

SIMPL+ can only load net472 (or 3-Series net35 CF) libraries, and its cross compiler is the most fragile part of the toolchain.

- The single most-asked question in the channel: "my clz won't compile in S+". The cause is always a net6/net8 library. Build it for net472, or multi-target and feed S+ the net472 output.
- Async code can't be called from S+ directly. Libraries used by both S+ and SSP end up with a sync surface for S+ and an async one for SSP.
- Keep S# libraries for S+ light. The cross compiler loads your assemblies (via reflection) once per module instance and runs static initialisers. Heavy dependencies or static setup can add minutes to SIMPL compiles. A module initializer that detects the SPlus cross compiler and skips setup avoids this.
- Never reference test frameworks (xUnit etc.) from the library project. The clz then carries test DLLs and S+ fails to load it. Put tests in a separate project.
- Intermittent "Statement outside of function scope" errors on the line that instantiates a class are cross-compiler/API-generator flakiness. Recompile; there is no clean fix.
- To reuse a SIMPL+ module's library in C#, rename the clz to .zip and reference the DLL inside. Using net472 DLLs from net8 sometimes works (Niklas) and sometimes doesn't (a Lutron LEAP module failed for Simon). Recompile or multi-target when you can.
- Between a SIMPL program and an SSP program use EISC: 3-Series EISC to 127.0.0.2 for programs on the same processor, EISC Server/Client between processors or to VC-4. Successful `Register()` does not mean online. Identical programs on several processors need IPIDs assigned from config.
- Resources for wrapping: Kiel the Coder's blog (SIMPL and delegates), TBD Enterprises' SIMPL# Fundamentals course, and the Masters session videos.

## Program lifecycle, threading and async

Get out of the constructor and `InitializeSystem()` fast, and treat Crestron's event model as synchronous.

**Startup**

- `InitializeSystem()` must return within the watchdog window or the program is killed. Queue the real start-up (e.g. registering 100+ NVX endpoints) onto a thread-pool work item or Task and return. Jeff McAleer's pattern: queue work, sleep \~750 ms so the program reports started, then initialise.
- Keep the `ControlSystem` constructor and especially static constructors trivial. An exception in a static ctor surfaces only as "Error loading Application Specific constructor".
- Initialise in two phases: create and register every device first, then wire up resources (serial/IR/relay ports, NVX-hosted ports). Alternatively hand devices a port proxy that is attached when the real port appears. This removes load-order dependencies.
- `MaxNumberOfUserThreads` only throttles `CrestronThread`. If you use System.Threading you can leave it alone or set it to the 400 maximum.

**Threads and tasks**

- Use Tasks or the thread pool, not raw threads, so the program can stop cleanly on `progreset`.
- Crestron SDK events aren't async. Async all the way down only works where you own the stack; at the event boundary, catch exceptions inside the handler.
- `Task.Delay` is not precise; with a busy thread pool a 10 s delay was observed taking 30 s. Measure with `Stopwatch`, not console timestamps, and use `CreateRamp` on panel analogs for smooth progress bars.
- Always pass a `CancellationToken` to long-running loops, connects and process launches so shutdown is clean.
- A runaway `CTimer` that resets itself in a loop caused 50% CPU on all 8 VC-4 cores. When CPU spikes, bisect by removing features.
- `TaskScheduler.UnobservedTaskException` is worth logging, but it also catches noise from libraries (e.g. Seq sink IO errors). It fires from the GC finalizer, so timing is unpredictable.
- `System.Threading.Channels` suits producer/consumer queues such as a prioritised command queue to a device. Channels are one-to-one and don't replace events.
- `lock()` works normally on 4-Series.

**Closures**

- Subscribing to events in a loop with a lambda that captures a `for` index attributes every event to the last item. Capture the item, copy the index to a local, or call a local function. Crestron's `UsbCInputs`/`HdmiInputs` collections don't hand you the index in the event args, so this matters.

## Networking: TCP, control subnet, multicast, SSH, TLS

Standard `System.Net` sockets are the better default, except on control-subnet processors where only Crestron's classes reach the LAN port.

**TCP clients**

- The Crestron `TCPClient` has an unfixed race on async reconnect: one instance can leave 2–4 sockets open. Against a Bose ControlSpace (4-connection limit) that locks you out.
- `System.Net.Sockets` gives `CancellationToken` support, clean shutdown and code that doesn't change with firmware. It doesn't raise a disconnect event.
- `Connected` is a cached last-known state on both stacks. Detect half-open connections with an application-layer heartbeat or poll (Stephen Cleary's "detection of half-open connections"). This also keeps TCP and RS-232 drivers consistent.
- To check socket state, `IPGlobalProperties.GetActiveTcpConnections()` filtered on the client's endpoints returns `TcpState`.

**Control subnet (CP4N, MC4-R, PRO4 etc.)**

- In isolation mode, `System.Net` listeners and clients see only the control-subnet NIC. Only Crestron's classes with `EthernetAdapterType.EthernetLANAdapter` reach the LAN. That includes Kestrel: it listens on the CS side only, so port-forward, which isolation mode breaks.
- On CP4N specifically, the LAN NIC is in the firmware's own network namespace — not accessible to the .NET process at all. Kestrel and HttpListener only bind `172.22.x.x` (CS side). The only way to serve HTTP or WebSocket traffic to LAN clients from an SSP program on CP4N is via CWS (`HttpCwsServer`), which routes through the firmware's port-80 web server on the LAN NIC. This makes CWS required on CP4N even though Kestrel is preferred everywhere else.
- Multicast from an N-processor (sACN, Visionary, custom UDP) often never leaves the CS side, even with the Crestron `UDPServer` on the LAN adapter. Unicast traverses the router; multicast needs IGMP proxy (CS processors only) and routing. AV Coders' wiki page "Multicast with a N processor" documents a working setup that still only receives from the LAN subnet.
- Without a control subnet (RMC4, CP4), multicast is just a network problem; `IGMPPROXY` doesn't exist on those models.

**SSH (Biamp Tesira etc.)**

- SSH.NET (Renci) works on 4-Series. Moving the same Tesira code from net472 to .NET 6/8 broke when `ShellStream` was wrapped in `StreamWriter`/`StreamReader`; read and write the `ShellStream` directly and set `TerminalModes.ECHO = 0`.
- The Crestron SSH client calls `Thread.Abort()` in `Connect()`, which throws on .NET 8 and crashed a whole program (Jul 2026). Use SSH.NET.
- Beginners: start with a TCP display driver before SSH/Tesira; there is a lot of protocol stacked up there.

**HTTP, TLS and crypto**

- FIPS mode is on by default from 2.8005, and .NET 8 uses OpenSSL 3. `MD5.Create()` then throws `OpenSslCryptographicException` — which breaks HTTP Digest auth. Turn FIPS off on that box or use another auth mode. For non-security hashing (config change detection) use CRC or xxHash.
- Ciphers that only worked through Crestron's HTTPS client were Mono limitations; retest with `HttpClient` on .NET 8. Crestron's HTTPS class also broke on one late-2024 firmware.
- `SecureTCPClient` allowed TLS without a client certificate (Lutron QSX). Replacements: `SslStream`, or a BouncyCastle TLS client (PepperDash's `LutronTlsClient` supports mTLS). Simon found his .NET version slower and reverted.
- Certificates: `new X509Certificate2("/user/…/cert.pfx")` works on .NET 8 without BouncyCastle. Mono had permission problems that pushed people to BouncyCastle. The processor's own self-signed cert is root-owned and not readable by the SSP app.
- Let's Encrypt via Certes and a DuckDNS DNS challenge works on VC-4 with auto-renewal; on appliances people generate a self-signed pfx for Kestrel/WebSockets.
- Keep firmware current for runtime security fixes (e.g. ASP.NET Core request-smuggling CVE-2025-55315).

## Web hosting: CWS vs ASP.NET Core

On .NET 8 firmware, host web APIs and UIs with ASP.NET Core (Kestrel) instead of CWS; the people who switched report large speed-ups and far less code.

**CWS (HttpCwsServer) on .NET 6/8 is unreliable — but required on CP4N**

- .NET 6: request cookies missing (acknowledged bug).
- .NET 8: handlers not invoked, HTTPS returning 500, response headers/CORS broken and OPTIONS preflight never reaching `ProcessRequest` (2.8005-era firmware).
- If you must keep legacy CWS routes, Kestrel can forward `/cws/*` to your existing handler while you migrate.
- **Exception: CP4N.** Because the LAN NIC is firmware-namespace-isolated (see networking section), CWS is the only viable HTTP transport for LAN clients on CP4N. Use it on CP4N, Kestrel everywhere else.

**CWS gotchas when you have to use it**

- **CWS does not support streaming.** The firmware always adds `Content-Length` and buffers the entire response until `ProcessRequest` returns (or `Write(s, final: true)` is called). `Response.Flush()` appears in the API but does not push data to the socket mid-request; bytes received by the client will be 0 until the handler returns. SSE (Server-Sent Events) is impossible over CWS. Use **long polling** instead: `ProcessRequest` blocks with `Monitor.Wait` until state changes (or a timeout), then writes one complete JSON response with `final: true`.
- **`context.Request.Path` is relative to `/cws/`.** For a request to `/cws/treacy/index.html`, `Path` is `/treacy/index.html` — the firmware strips the `/cws` prefix before calling the handler. Strip only the virtual path prefix (e.g., `/treacy`) when mapping to the filesystem.
- **CWS `ProcessRequest` runs on unmanaged firmware threads**, not .NET thread-pool threads. These threads survive .NET program restarts (PROGLOAD/PROGRESET) and cannot be killed without a full processor reboot. If `ProcessRequest` never returns, the hanging thread blocks PROGLOAD indefinitely. Keep all handlers time-bounded.
- **`HttpCwsServer.Dispose()` only unregisters its own registered instance.** A fresh `new HttpCwsServer(path)` followed by `Unregister()` or `Dispose()` is a no-op for a path registered by an earlier instance. Stale registrations from a previous program run cause `Register()` to return `false` on the next start. Fix: subscribe `CrestronEnvironment.ProgramStatusEventHandler` and call `_server.Dispose()` when `eProgramStatusEventType.Stopping` fires, so the path is unregistered before PROGLOAD starts the next run. A processor reboot clears all stale registrations.
- **Route `HttpRequestHandler` vs `RouteHandler`:** named routes (`HttpCwsRoute`) use the `RouteHandler` property; the catch-all handler uses `HttpCwsServer.HttpRequestHandler`. Adding named routes does not prevent the catch-all from firing for unmatched paths.

**ASP.NET Core on the processor**

- Works on 4-Series 2.8004+/2.8005 and VC-4 as part of an SSP program. Troy Garner's `CrestronCommunity/WebApp` repo is the reference for hosting it alongside `CrestronControlSystem` with Microsoft DI and hosted services.
- The entry assembly is `SimplSharpPro.exe`, so controller discovery finds nothing. Fix: `builder.Services.AddControllers().AddApplicationPart(typeof(ControlSystem).Assembly)`.
- Set the port in `appsettings.json` under `Kestrel:Endpoints`, or compute it as base + `ProgramId` so several slots don't collide.
- Serve a UI with `UseStaticFiles()`. For SPAs add a fallback to `index.html` so refreshing a deep link works. Next.js static export broke on refresh; Vite + react-router was fine. Crestron panels loading from the filesystem need `MemoryRouter`.
- Blazor Web Apps run on CP4 and VC-4 (see packaging notes). Point a panel at it in generic web-app mode with processor IP and port. `wms-bp/crestron-blazor-identity` and `plawall-qvest/CrestronBlazorDemo` are working examples.
- Hosted services work; one program that wouldn't load after reboot was traced to its hosted service, but others run several without issue. Put cleanup in `StoppingAsync` of an `IHostedLifecycleService`.
- SignalR replaced WebsocketSharp for several people ("a new dimension"). Use built-in JWT auth for SignalR authorisation.

**WebSockets and panels**

- WebsocketSharp is flaky on processors (null-ref crashes on PRO4 but not RMC4, broken `SendToAsync` in the NetCore fork). Mono's WS/SSL was poor and won't be fixed. Fleck works but WSS needs certificate access.
- Browsers answer WebSocket ping frames automatically, but browser JS can't send them. Add an app-level heartbeat (one team sends every 3 s and reloads the UI if it stops).
- CH5 XPanel without a login: the `/html` folder is gated by `userpageauth`. Options are `userpageauth off` plus your own login on config pages, or hosting the UI yourself. The `websockettoken` console command is admin-only and unavailable from SSP.
- A common state pattern: the device is the source of truth, the SSP keeps state, the UI requests full state on connect then applies deltas (e.g. into a Redux store).
- On iPad, Crestron ONE in HTML5 mode accepts self-signed certs; browsers nag.

## Files, paths, config and storage

.NET 8 on a 4-Series is plain Linux: absolute forward-slash paths, real file permissions, and no Mono path-mangling to save you.

**Paths**

- Use absolute paths starting `/user/…`. `PhysicalFileProvider` rejects relative roots ("The path must be absolute").
- A Windows-style path such as `\user\main\data\x.json` doesn't fail on .NET 8 — it creates a file literally named that in the app folder. Use `/` or `Path.Combine`.
- `Directory.GetApplicationRootDirectory()` returns an empty string on appliances (root is `/`). VC-4 differs, so branch on `CrestronEnvironment.DevicePlatform` (Appliance vs Server).
- Crestron's `AdjustPathForMono` rejects legacy paths like `\NVRAM\USER\…`, which broke the TV Presets DLL.
- To detect Mono at runtime, `Type.GetType("Mono.Runtime") != null` still works.

**Permissions**

- The SSP app runs as a different user from SFTP. Folders created over SFTP may be unreadable by the program; create them from code.
- Deleting and recreating `/user` over FTP broke writes until a reboot.
- Runtime behaviour differs between models on identical firmware: `Assembly.LoadFrom` from `/user` works on CP4 but is denied on MC4 (only the program directory is allowed). Some S# libraries load on CP4 but not MC4/DIN-AP4. Test on the target model.
- FileZilla can show a stale listing; reconnect. WinSCP handles processor timeouts better.

**Config and serialisation**

- `System.Text.Json` is faster, lighter and doesn't clash with `SimplSharpNewtonsoft`. Newtonsoft is still richer for custom converters and polymorphism.
- A JSON config that loaded with Newtonsoft failed with `System.Text.Json`; encoding/BOM was suspected but not confirmed. If you switch, re-test every config file.
- For XML enums use `[XmlEnum]`, not `[EnumMember]`.
- Persist only state you can't poll (screen and lift positions etc.), as JSON.
- `StringComparer.Ordinal` on string-keyed dictionaries (command dispatch tables) is noticeably faster; `FrozenDictionary` is net8-only.

**Databases and secrets**

- SQLite on appliances: the process is 32-bit `linux-arm`. Publish for that RID and call `raw.SetProvider(new SQLite3Provider_sqlite3()); raw.FreezeProvider();`. A 64-bit build gives "wrong ELF class"; some still hit "failed to map segment" and fell back to `Crestron.SimplSharp.Sql`. EF Core + SQLite works on VC-4 with a `linux-x64` publish.
- Connecting to an external MySQL/SQL Server from an RMC4 failed with "environment not supported" while VC-4 worked; results are mixed.
- Secrets: register a console command that writes them to `CrestronSecureStorage` (not available on VC-4). `Environment.SetEnvironmentVariable` works from code; there is no supported console way.
- Small static files in `/html` are served by the processor web server only if `userpageauth` is off.

## Device-class gotchas

The SSP help file is often wrong or silent for DM, NVX and NAX; when stuck, drop the device into SIMPL Windows to see how it behaves, or check its REST API docs.

| Device / area | Lesson |
| --- | --- |
| NVX routing | Route with `rx.Control.ServerUrl.StringValue = tx.Control.ServerUrlFeedback.StringValue`. Read a transmitter's address from `MulticastAddressFeedback`; setting `MulticastAddress` didn't stick in testing. |
| NVX stream change | On the base event watch `DMInputEventIds.ServerUrlEventId` and read `Control.ServerUrlFeedback`. |
| NVX at scale | 150+ endpoints by IPID is fine if registration runs off `InitializeSystem()`: register, subscribe `BaseEvent`, `OnlineStatusChange` and each `HdmiIn.StreamChange`, then set encoder/decoder mode from config. |
| NVX IR via REST/WebSocket | Payload to fire a parsed command from a loaded .ir file wasn't solved in-channel (May 2026). |
| New products | Some newer NAX models weren't in the SDK yet; check before quoting S# Pro. |
| Touch panel reserved joins | Joins above \~16000 belong to the panel's extenders; use the extender properties (`RegisterExtenders`) rather than raw sig numbers. Digital 17917 (pulsed) reboots a TSW-xx60. |
| Smart objects | Address sigs by name (the strings shown on the SIMPL symbol) via `StringInput["…"]` instead of guessing join offsets. |
| Panel progress bars | `UShortInput[n].CreateRamp(target, time)` gives smooth bars without timer loops. |
| DIN-1DIMU4 | Can't change `ParameterDimmable` after registration: Unregister, Dispose, recreate, set, Register. `TieInputToOutput` breaks non-dimmable loads because feedback ramps through interim values; instead set level once from the first `LevelOut` after each reconnect. |
| DIN-CENCN-2 | `Unregister()` returns success but the Cresnet ID isn't freed until program restart. |
| ST-IO | `Relay.StateChange` never fired (C2N-IO fine). |
| CH-HRMOT4-D4 | No CNWM-LT9 class in S#; declaring it as `C2NiCb` or `Un8io` drives the relays. |
| DM-NAX-AMP-X300 | Separate Commercial and Residential classes match the device mode; commercial needs stereo channels synced in code; mode change needs a reboot. |
| DIN-KXI (KNX) | No S# classes. It is a rebranded Weinzierl KNX IP BAOS 774: use its object-server protocol, or tunnelling with the Falcon SDK. |
| UC Engine / MTR | `ExtenderUcEngineFlexTeamsReservedSigs` values are sigs: set `.StringValue`/`.BoolValue`. Camera PTZ via repeated extender calls isn't smooth. |
| VC-4 / CrestronOne | Set `ParameterProjectName.Value`. `InitialParametersClass.ApplicationNumber` is the VC-4 database ID used by its API; use `RoomName` for room identity. |
| IP table | `GenericBase.Description` sets the entry's comment. |
| System info | `CrestronEnvironment.SystemInfo.TotalRamSize`/`RamFree` return 0 or crash the slot on .NET 8, even inside try/catch. Use .NET APIs behind `#if NET6_0_OR_GREATER`. |
| Class naming | S# class names don't always match SIMPL hardware names (e.g. CHV-TSTAT-FCU-PIR-10 is `ChvTstatFcuHotelPir10WT`). Search the namespace rather than guessing. |

## Debugging, logging and testing

Step-through debugging works on VC-4 today; on appliances plan around good logs and testable code.

- .NET 6/8 stack traces include line numbers — the single biggest day-to-day win over Mono.
- VC-4: remote debugging with Rider or VS 2022 works. Rider 2025.1+ needs `AllowTcpForwarding yes` in the VC-4 `sshd_config`. Full symbols work for net8; net472 debugging needs portable symbols. Crestron OLH articles 1000637 and 3102 cover it but lagged behind.
- Appliances: firmware 2.8006.00110 notes claim debugging support, but as of Mar 2026 nobody in the channel had it working and the OLH wasn't updated.
- dotnet-counters / dotnet-dump aren't available on processors.
- Logging: Serilog with a Seq sink is popular. Echo to console selectively so it doesn't drown Crestron's own commands. Sinks swallow their own exceptions, so missing logs may be a sink problem.
- Instead of a side-channel TCP shell, register one console command (e.g. `exec`) and dispatch subcommands from it; you connect over SSH as normal.
- Shelling out: `Process` with `/bin/bash -c`, redirect stdout and stderr, and enforce a timeout with a `CancellationToken` — commands like `top` never exit. The app only has its own user's permissions.
- Report the program's informational version through a console command; Toolbox's program info pane can't be customised.
- Unit tests: keep business logic free of Crestron types and hide SDK classes behind interfaces so they can be faked. `Crestron.SimplSharp` types throw off Crestron hardware. Split solutions into at least three projects: core logic, Crestron glue, tests.
- AI assistants are good at general C# and weak on Crestron specifics. The `wms-bp/ssp-tool` MCP server indexes the SSP and SIMPL Windows help files for coding agents; human review remains essential.

## Architecture and coding practices

The payoff from C# comes from a reusable, config-driven framework; writing each room from scratch is slower than SIMPL.

- Config-driven framework: read a JSON config, create devices (codec, displays, cameras, panels) from it, and let a room manager wire behaviour between them. Only instantiate what the room uses — unlike SIMPL, where every module runs. PepperDash Essentials is open source and a good starting point.
- Dependency injection: Microsoft DI works in SSP. Inject an abstraction such as `IControlSystemContext`, not `CrestronControlSystem` itself, to keep it mockable and limit what consumers can touch. Use factories (or Castle Windsor typed factories) for instance-specific arguments instead of injecting the service provider.
- Thread-safe events: `public event EventHandler<T>? Changed;` raised with `Changed?.Invoke(this, e)`. Many frameworks rely on `INotifyPropertyChanged` for device state.
- Null handling: null references and re-entrancy are the two classic pitfalls for SIMPL programmers moving to C#. Null Object and `Result<T>` patterns help; don't silence the compiler with the `!` operator. Crestron's net6+ packages carry nullable annotations that are sometimes wrong (`UserObject` can be null).
- Primary constructors: their parameters aren't readonly. At least one shop bans them in review.
- Prefer standard .NET libraries over Crestron wrappers where possible (sockets, HTTP, scheduling, SQLite, JSON). Firmware updates have broken Crestron classes repeatedly; standard libraries are better tested. `NetSchedulerLib` is a community replacement for `Crestron.SimplSharp.Scheduler`.
- Embed IR drivers as resources, write them to disk at start-up and call `LoadIRDriver`, so the program is self-contained.
- Keep source in git as text; don't commit zipped sources. Version programs properly (semantic or date-based informational version) rather than editing file names.
- Agree with the integrator up front whether they can support a C# deliverable; field techs often rely on SIMPL Debugger.

## Firmware and SDK history

Most "it worked yesterday" problems in the channel traced back to a specific firmware or NuGet release; check this list before debugging your own code.

| When | Release / event | What changed or broke |
| --- | --- | --- |
| Oct 2026 | `Crestron.SimplSharp.SDK.Compatibility` 2.22.40 | Lets net6/8 code use S# Framework-dependent libraries (RAD SDK). Program package still lacks a true net8 runtimeconfig target. |
| Jul 2026 | — | Crestron SSH client `Thread.Abort()` crash on .NET 8 reported. |
| Jun 2026 | Latest firmware | Extra binaries loadable via PUF (first: Zoom Rooms C++ SDK). .NET 10 would still need a firmware update. |
| Jul 2025 | 2.8006.00110 | Release notes claim .NET 8 debugging on appliances; docs not updated. |
| Feb 2025 | 2.8005 | FIPS on by default; OpenSSL 3 breaks MD5 under .NET 8. |
| Dec 2024 | 2.8005.00031 | ASP.NET Core runtime present. Some programs fail to reload after reboot (erase, reboot, reload); Crestron cases opened. |
| Nov 2024 | NuGet 2.21.54+ | SimplSharpNewtonsoft, ExchangeWebServices and ReflectionInterface dropped from net6+ packages. CWS on net8 broken; SystemInfo RAM properties broken. |
| Oct 2024 | 2.8005 + NuGet 2.21.x | .NET 8 support released for 4-Series and VC-4. |
| Sep 2024 | 2.8004 (pulled) | Fixed the .NET 6 progreset crash but was withdrawn for other bugs. |
| Aug 2024 | 2.8003.00049–00056 | .NET 6 programs crash with `free(): invalid next size` on progreset or reboot, even when empty; reload works. Fixed in .NET 8 firmware. |
| Mar 2024 | 2.8003.00037 | .NET 6 support for SSP on 4-Series appliances. No VC-4, no S+, no debugging, CCDs unsupported. |

## Resources and links

All links were shared in the channel; repos are community-maintained.

- [Source channel: #simplsharp-net6-net8, CrestronProfessionals Discord](https://discord.com/channels/314425881994985472/1212883338293944390) — Troy Garner's pinned multi-targeting template is there.
- [CrestronCommunity/WebApp](https://github.com/CrestronCommunity/WebApp) — ASP.NET Core + DI in an SSP program.
- [CrestronCommunity/KnowledgeBase](https://github.com/CrestronCommunity/KnowledgeBase)
- [PepperDash Essentials](https://github.com/PepperDash/Essentials) and [ICD Systems](https://github.com/ICDSystems) — open-source frameworks and drivers.
- [wms-bp/ssp-tool](https://github.com/wms-bp/ssp-tool) — MCP server over the SSP and SIMPL Windows help files.
- [wms-bp/crestron-blazor-identity](https://github.com/wms-bp/crestron-blazor-identity) and [plawall-qvest/CrestronBlazorDemo](https://github.com/plawall-qvest/CrestronBlazorDemo) — Blazor on a processor.
- [srmeteor/NetSchedulerLib](https://github.com/srmeteor/NetSchedulerLib) — replacement for the Crestron scheduler.
- [AV Coders wiki: Multicast with a N processor](https://github.com/AV-Coders/crestron-components/wiki/Multicast-with-a-N-processor)
- [Crestron.SimplSharp.SDK.Compatibility](https://www.nuget.org/packages/Crestron.SimplSharp.SDK.Compatibility)
- [Kiel the Coder: SIMPL and delegates](https://kielthecoder.com/2023/03/05/simpl-and-delegates/) and [CWS part 1](https://kielthecoder.com/2021/05/03/cws-part-1/)
- [Stephen Cleary: detection of half-open connections](https://blog.stephencleary.com/2009/05/detection-of-half-open-connections.html)
- [Crestron OLH 3102 (debugging)](https://community.crestron.com/s/article/3102) and [control subnet reference](https://docs.crestron.com/en-us/8559/Content/Topics/Reference/Control-Subnet.htm)
- [TBD Enterprises SIMPL# Fundamentals course](https://www.tbdenterprises.ca/product/simplsharp-fundamentals-course/)
- [Microsoft: performance improvements in .NET 8](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-8/)
