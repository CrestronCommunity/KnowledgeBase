# Crestron Programming – Lessons Learnt from #modern-tools

Nov 6, 2026 · @Rod Driscoll

## About this document

These are the practical lessons distilled from the CrestronProfessionals Discord **#modern-tools** channel, covering its full history from April 2020 to November 2026 (about 2,800 messages). The channel is where Crestron programmers discuss tooling around the job: Git, IDEs, build chains, CI, terminals, debuggers, remote access and, more recently, AI assistants.

Lessons are written as advice, not attributed quotes. Where the community disagreed, both positions are given. Version-specific details (Rider settings, Crestron NuGet behaviour, .NET support) reflect the date they were discussed and should be re-checked against current tooling.

## Version control for SIMPL Windows, VT Pro and SIMPL+ projects

The consensus is clear: use Git even when most files are binary, because a single, consistent history beats "is program\_v2.12 the latest?" on a shared drive a year later.

### Tell Git which files it must never merge

- Mark `.smw`, `.umc` and `.vtp` as binary in `.gitattributes`. Left as text, Git will happily merge two different SMW files and corrupt the program.
- Put large or compiled files in Git LFS (`.vtp`, `.vta`, `.vtz`, `.smw`, `.sig`, `.lpz`, `.cpz`, `.clz`, `.cplz`, `.zip`). VTP files get very large, and GitLab's 100 MB file limit bites without LFS.
- Keep `.usp` and `.ush` as text with `eol=crlf`.
- Commit `.gitattributes` and `.gitignore` in every repo rather than relying on global files. It survives a new laptop and lets a colleague open your project when you are on leave. A repo template with your standard folder structure plus these two files makes this automatic.

A trimmed version of the attributes file one large integrator uses:

```
*.usp text eol=crlf
*.ush text eol=crlf
*.umc binary
*.smw filter=lfs diff=lfs merge=lfs -binary
*.sig filter=lfs diff=lfs merge=lfs -binary
*.vtp filter=lfs diff=lfs merge=lfs -binary
*.vta filter=lfs diff=lfs merge=lfs -binary
*.vtz filter=lfs diff=lfs merge=lfs -binary
*.lpz filter=lfs diff=lfs merge=lfs -binary
*.cpz filter=lfs diff=lfs merge=lfs -binary
*.clz filter=lfs diff=lfs merge=lfs -binary
*.cplz filter=lfs diff=lfs merge=lfs -binary
*.zip filter=lfs diff=lfs merge=lfs -binary
```

### Structure repos around the project

- Simplest: one repo per project containing every program slot, UI file, DSP file, config backup and drawing.
- At scale: a "super-repo" per project holding configs and scripts, with submodules for the SIMPL program, the UI and shared scripts. Name related repos so they sort together.
- A shared HTML5 UI forked per job and pulled in as a submodule is a reasonable pattern; nobody in the channel had found a cleaner one.

### You cannot merge SIMPL, so plan around it

- Divide work by file: different slots, programs, VTPs or modules per person, never the same file at once.
- For long parallel work on one module, keep a master copy plus a personal copy per programmer and merge manually.
- On a binary conflict: save your version under another name, pull theirs, diff them with a SIMPL compare tool (Ultamation Simplified), then merge by hand and commit.
- Git will refuse a push when you are behind, so the second person is always forced to reconcile rather than silently overwrite.

### Keep modules inside the project

- Modules picked up from the global `UsrMacro` / `UsrSPlus` folders end up in the compiled output but not the project folder, so the next person to clone the repo cannot build.
- Leave those folders empty and copy each module into the project folder. SIMPL Windows also starts much faster with them empty.
- Keep the shared module library in its own repo, or publish versioned releases (one team uses an internal NuGet feed).
- If you do version the global folders in Git, commit both `.usp` and `.ush` (the module will not appear without the `.ush`). SIMPL rewrites `.ush` files on every compile, so discard those changes unless the module's signature really changed.

### Versioned module filenames are the accepted exception

Changing the inputs or outputs of a SIMPL+ wrapper or `.umc` that is already in a program mangles its signals. Bump the version in the filename, add the new module beside the old one, copy the signals across, then delete the old one. This breaks the usual Git rule of no version numbers in filenames, but SIMPL leaves no better option.

### Compiled output: two schools

| Store `_compiled.zip` / `.lpz` / `.vtz` in Git | Keep only source |
| --- | --- |
| Exact binary as built against the original Crestron database versions; a later recompile on newer databases can introduce bugs you never tested | Source control is for source; generated files bloat the repo |
| Lets field techs load without recompiling | Publish builds as GitHub/GitLab Releases or to cloud storage instead |
| `.hash` files let you verify what is actually on site | Use LFS if you do store them |

The SIMPL Windows and database versions used for a build are recorded inside the `.smw` itself (`RelVrs`, `DbVr`, `DvcDbVr`), which helps when you need to reproduce an old build.

### Small Git fixes worth knowing

- Git for Windows "detected dubious ownership": `git config --global --add safe.directory *` (no quotes around the asterisk).
- Self-hosted GitLab behind Let's Encrypt: point the web server at the full-chain certificate. Browsers accept the leaf certificate alone; Git's OpenSSL does not. With a corporate CA, switch Git to the `schannel` backend.
- Spreadsheets (`.xlsx`) cannot be merged either. Keep device/IP sheets as CSV or JSON if they must be diffed, or keep them outside the repo where non-programmers need access.

## Team workflow and process

Process first, tooling second: Git does not manage the workflow for you, and most repo messes come from people not following an agreed process.

- **Commit every time you load code to a processor.** One team also posts every commit message to a Slack channel, so everyone sees who changed which site.
- **Pull before you start.** Checking the repo first shows who else is working on the system and often avoids an unmergeable SIMPL conflict entirely.
- **Watch for unpushed commits.** The most common failure is committing on an offline job site and forgetting to push later. Push straight after commit when you can, pick a Git client that flags unpushed commits clearly, or run `git log --branches --not --remotes` across your repos.
- **Avoid a bus factor of one.** Spread service calls for big clients across several programmers, document the workflow, and get buy-in from "lone wolves" who prefer their own way. Siloed knowledge hurts the company when that person leaves.
- **Branching (GitFlow) works well for C# and HTML5,** with feature/bugfix branches off `dev`. Remember to merge hotfixes made off `main` back into `dev` as well. It adds little for SIMPL, where nothing merges.
- **Learn command-line Git** even if you use a GUI; it makes recovery possible when the GUI gets confused. `git rebase -i` is worth learning for tidying local history, but never rewrite a branch someone else has pulled. GUIs mentioned favourably: Fork (one-off licence, fast, workspaces per project), GitKraken (subscription), SourceTree (free but slow with many repos), plus the IDE integrations.
- **Watch branch names.** A push to `Master` instead of `master` silently creates a new branch on case-sensitive servers.

### Versioning C# builds

- Set versions in `AssemblyInfo` and read them at runtime via reflection if the program needs to report its version.
- Make local builds report something obvious like `0.0.0-local`, so you can tell at a glance when an untracked build is running on a processor.
- Let CI stamp real versions from Git tags, with pre-release suffixes for branch builds (e.g. `1.6.4-alpha-745`), and only build releases from a tagged release.
- Build and test locally before pushing; do not commit binaries for C# projects.

## SIMPL# and SIMPL# Pro toolchain

The toolchain, not C#, is where most time is lost. Settle your build setup once, script it, and template it.

### 3-series (.NET Compact Framework 3.5)

- Officially this still needs Visual Studio 2008 with the Crestron plugin, because nothing newer supports .NET CF 3.5.
- Rider can build 3-series code if the MSBuild toolset is set to 4.0 and the C# language version is restricted to 3. Otherwise the IDE lets you use modern syntax that will not compile for the target.
- Community workarounds exist: a sandbox-build NuGet package that targets .NET 3.5 from Rider or modern VS (the VS2008 plugin still has to be installed for the tooling), and `VsIdeBuild` to drive the VS2008 build from the command line. One VS2022 sample on GitHub signs output with the wrong (SIMPL+) DLL, so check which one you copy.
- Just building a .NET 3.5 DLL is not enough; Crestron output also needs the manifest and signing steps the plugin provides.

### 4-series (.NET Framework 4.7.2, and later .NET 8)

- Use the Crestron NuGet packages; Rider works well and is now free for non-commercial use.
- When creating a project, do not pick a .NET Core template. Target `net472`, or `ControlSystem.cs` is never added.
- Library not producing a `.clz`? Remove the SIMPL# Pro program library reference unless the library genuinely needs devices; it blocks CLZ creation. For a `.cplz`, add a custom MSBuild target (the PepperDash Essentials `Directory.Build.targets` is a good example).
- JSON conflict (`JsonPropertyAttribute exists in both Newtonsoft.Json.Compact and SimplSharpNewtonsoft`): the SDK pulls in two JSON libraries; delete the `SimplSharpNewtonsoft` reference manually.
- .NET 8 on 4-series works, but as of 2026 Crestron's packages were not updated for it, so expect to adjust the NuGet runtime references yourself.
- Multi-targeting (`net472` + `net6.0`/`net8.0`, or `netstandard2.0` for shareable libraries) lets one code base serve both; target a specific framework only where it gives you something, such as .NET 8 performance.

### Build-and-deploy automation

Visual Studio's Post-Build event runs *before* the CLZ/CPZ exists, so a naive copy step ships the previous build. Hook the target after Crestron's packaging step instead (see Crestron Answer ID 1001078):

```
<Target Name="PostCLZBuild-MoveCLZ" AfterTargets="SimplSharpPostProcess">
  <Copy SourceFiles="$(ProjectDir)$(OutDir)$(TargetName).cpz" DestinationFolder="$(SolutionDir)" />
</Target>
```

The same hook can push and load the program: `pscp` the `.cpz` to `program01`, then `plink` `PROGLOAD -P:1` (Release), or for Debug `PROGLOAD -P:1 -D`, `DEBUGPROGRAM -P:1 -Port:50000 -IP:0.0.0.0 -S`, `PROGRESET -P:1`. Add `-l admin` (or `user@ip`) to each `plink` line, keep credentials out of shared repos, and use a separate build configuration if you also run the project locally as a console app.

### IDE housekeeping

- Add Rider's `.idea` folder to `.gitignore`; committing it restores your window layout on everyone else's machine.
- Rider's NuGet UI did not use `nuget.config` credentials for private feeds and could overwrite the file; Visual Studio cannot change a mistyped feed password either, so you have to delete and re-add the feed.
- Visual Studio updates have broken Rider builds by removing the MSBuild version Rider was pointed at; re-select the toolset after updates.
- Align keyboard shortcuts across VS Code, Visual Studio and Rider with keymap extensions; mismatched shortcuts cause real mistakes.
- Your own project template (folder layout, C# projects, project references, SIMPL/S+ files, standard `ControlSystem.cs`) saves the repeated setup and fixes the typos in Crestron's default file.

## Debugging, release builds and runtime gotchas

Always deploy a Release build, and test that Release build before you walk away from site.

- **Release vs Debug.** One programmer saw a large performance difference on a TSR-310 system simply by switching the C# project to Release. Code has also been seen working in Debug and failing in Release, so test the build you ship.
- **Attaching a debugger.** On 4-series you start the program suspended and attach; on VC-4 you can attach to a running program. Visual Studio 2022 lacked the Mono debugger plugin for some time; Rider handled 4-series debugging well, including editing code while the program runs.
- **Stopping a debug session.** Stopping from Visual Studio could crash the processor. Stop from the console instead: `STOPPROGRAM -P:1`, clear the app folder, then `DEBUGPROGRAM -P:1 -C`. Save it as a Toolbox key sequence.
- **Many experienced programmers rely on logging, not breakpoints.** Plant debug messages generously; structured logging (Serilog has been back-ported to 3-series by at least one developer) pays off. Extremely verbose console output can freeze debug tools, so add log levels you can turn down.
- **Keep a reference to every `CTimer`.** A timer created without a stored reference, or stored only in a local, can be garbage-collected before it fires. Make it a class field, dispose it in the program-stopping handler, and suppress the analyzer's "never used" warning there rather than deleting the field.
- **Configure analyzers, don't fight them.** If you keep suppressing the same warning, lower its severity once in the rules instead of scattering `// ReSharper disable` or `#pragma` lines. Annotations such as `[CanBeNull]`, `[NotNull]` and `[PublicAPI]` stop false positives on console commands and entry points.
- **Write for the next reader.** Prefer a well-named variable over a clever ternary or one-liner (e.g. `var muteTuner = panel.Mode != PanelMode.UserProject;` plus a comment saying why). Use early returns, keep nesting to two or three levels, and break 250-line methods into named steps. A Rider/ReSharper cleanup profile run from a pre-commit hook can enforce house style without slowing you down while you type.
- **Optimise only with evidence.** Real-world testing often shows no benefit from a day of micro-optimisation; leave `// TODO optimise if needed` and move on.
- **Proofread panel text.** "TOUCH ANYWHERE TO BEIGN" on a client's panel at handover is avoidable; a spell-check pass over UI strings is cheap.

## SIMPL+ development and S# / S+ architecture

The S# + S+ wrapper route has the slowest edit-test loop in the Crestron world; several large integrators moved their logic to SIMPL# Pro for that reason.

- **Edit SIMPL+ in VS Code** with the community S+ extension rather than the stock editor. Open the *folder* ("Open with Code" on the folder, or the file's parent), not a single file, or the build task will not be found. The extension only calls Crestron's cross-compiler, so SIMPL Windows and the device databases must still be installed.
- **Test the S+ wrapper early.** The S+ cross-compiler can reject a type in a referenced dependency only after you have written the whole library and wrapper. Build a minimal wrapper against your public API on day one.
- **Compare the loops.** S# + S+: fix code → build library → copy CLZ → compile S+ → fix cross-compiler errors → compile SIMPL → upload → load → wait for startup → test. SIMPL# Pro in a second slot (talking to SIMPL via EISC): fix → build → upload → restart that slot → test, with the SIMPL program left running.
- **Wrappers still have a place.** For device drivers handed to SIMPL programmers or end users, a SIMPL+ wrapper feels familiar and avoids EISC confusion. For task or business logic, S# Pro is usually faster to develop and debug.
- **Design the wrapper for SIMPL, not just for C#.** Libraries built for C# efficiency and then exposed through many parameters and internal instance indexing can make the SIMPL side clumsy and slow.
- **Keep the functional code separate from Crestron APIs** so most of it can be unit-tested off the processor; only the thin Crestron-facing layer needs the slow deploy loop.
- **Learning curve.** The people who made the jump to C# say the barrier was mostly confidence and a lack of worked examples, not the language. Start with one small piece of a real project rather than a big first program, and use community frameworks (e.g. PepperDash Essentials plugins) as a sandbox.

## CI/CD, automation and cross-platform development

C# builds automate easily; SIMPL Windows and VT Pro do not, because Crestron offers no command-line compiler for them.

### What works in a pipeline

- 4-series C# libraries (multi-targeted `net472` / `net6.0` / `net8.0`) build fine in a Linux container on the standard `dotnet/sdk` image; no Windows runner needed.
- 3-series S# needs VS2008 and the Crestron plugin; one framework used a multi-gigabyte Docker image with all of it, which took 10–15 minutes just to pull on each run.
- Publish Docker images and packages from CI with explicit version tags, never `latest` from a developer's machine.

### Compiling SIMPL / VT Pro automatically

| Option | Notes |
| --- | --- |
| Ultamation Simplified 2 CLI compiler | Calls Crestron's own libraries, so needs Windows, SIMPL, the cross-compiler and databases installed, plus an annual licence |
| UI-automation tool that drives SIMPL / VT Pro by simulated clicks | Used by at least one large integrator in its pipelines; needs a desktop session, so a VM runner rather than a container |
| A dedicated Windows VM runner (cloud VM started on demand) | Most robust option for legacy projects; reset it regularly to clear leftovers |
| Wait for Crestron Construct's SIMPL support | Promised to be version-control and command-line friendly; was still early at last report and may not cover 3-series |

### Developing on Linux or macOS

- `dotnet build` can target `net472` on Linux/macOS using the reference-assembly packages; running it locally needs Mono.
- Rider works on Linux and macOS for 4-series C#.
- SIMPL, VT Pro and Toolbox still need Windows: a VM with a shared folder for compiling is the usual setup. Parallels on Apple Silicon compiles SIMPL quickly, but USB connections to processors did not work through it. WSL suits command-line tools but cannot run Windows-only AV utilities such as Dante Controller.
- Wine is not considered worth the effort for Crestron tools.

### Config as code

- Generating processor configs elsewhere (from a spreadsheet or database into JSON/XML) makes everything on the processor disposable and repeatable.
- To version config that changes on site, push it from the processor to a server that commits each change to Git; package that server as a Docker container so others can deploy it. Oxidized can do interval-based backups of device configs in the same way.

## Console, SSH, file transfer and debugging tools

Crestron's console is not a normal Linux shell, so standard terminals need a few settings before they behave.

- **PuTTY / MobaXterm settings that work:** terminal type `vt100`, local echo and local line editing forced on, backspace sends `^H`. The processor still echoes the command after Enter; ignore it.
- **Console differences.** A 4-series sends `<ProcType> Console` and a prompt on connect; a 3-series sends nothing until it receives a carriage return (any byte will do). Command echo is set with `echo` and can switch itself back on, so code that parses the console must tolerate echoed commands.
- **No server-side command history.** Up-arrow history exists only in clients that implement it (Toolbox: press Tab to edit a recalled command; SecureCRT's command window can also broadcast a command to several devices). Never put passwords in a command line, since client history stores them.
- **Terminal and transfer tools people use:** SecureCRT + SecureFX (shared address book, CSV session import), WinSCP + PuTTY, MobaXterm (opens SFTP alongside SSH), Termius, Tabby (check the user defaults to `admin` not `root`; older ciphers may need enabling), or plain `ssh`/`sftp` in Windows Terminal or tmux.
- **Transfers to 3-series can stall** with some SFTP clients; FileZilla was faster in one comparison. DMPS firmware uploads start fast and slow down progressively, so never start one late in the day on site.
- **Debugger reliability.** Toolbox's SIMPL debugger drops sessions; a third-party debugger (C5 Debugger) was repeatedly reported to stay connected for days. Whatever you use, cap console scrollback, because heavy logging can make any debug UI unresponsive.
- **Persist logs off the box.** Processor error logs can be shipped to a log platform (one team used Vector into Datadog and Humio); note the messages are not strictly standard syslog.
- **Typed APIs from JSON.** Paste a device's JSON (e.g. an HD-PS622 `/Device` response) into quicktype.io to generate C# classes instead of hand-writing them.

## Remote access, panel deployment and device quirks

### Remote access

- Ask for VPN access first; remote-desktop to an on-site PC is the fallback, which means your debug tools must be installable there (or on a jump box / field laptop).
- AnyDesk's TCP tunnels (for SSH, SFTP and WebSocket ports) proved unreliable for at least one heavy user, with no fix from support. Alternatives raised: TeamViewer VPN, RustDesk (self-hosted relays), Tailscale or WireGuard.
- Tailscale is painless, but its Windows virtual interface breaks Dante Controller (quit Tailscale to use it) and was about half WireGuard's throughput when relayed.
- Locked-down enterprise clients increasingly mandate their own path (2FA VPN to a whitelisted VM, Citrix, a client-issued laptop). Budget the extra connection time into support estimates.

### Panels, XPanels and CH5

- **XPanels hosted on Fusion (IIS):** enable directory browsing on the site and add a MIME type for `.c3p` (not `.cp3`), then point users at `core3xpanel.html`.
- **Loading a `.ch5z`:** `CSProjectLoad -T:MobileApp -U:CH5 -P:<file>.ch5z` works for Crestron One, but `-T:WebXPanel` reported "no WebXPanel found". Omitting `-T` loads it as both mobile app and web XPanel.
- **HTML5 build tooling ages fast.** Example projects built on old webpack only work on matching Node versions; record the required Node version in the README. Vite is the current direction, but check the oldest panel you support can run its output.

### Device quirks

- AirMedia backgrounds cannot be set by command. Copying the stored-files XML and images over SFTP made the web UI look right, but the device did not use them; plan on the web interface per unit.
- Some systems (Lutron hubs, Cisco codecs) require explicit subscriptions to feedback. This is deliberate, to avoid flooding the controller, so subscribe only to what the program needs.

## AI tools and knowledge sharing

AI assistants earn their keep on noisy analysis and first drafts, not on finished Crestron code.

- **Good uses reported:** analysing a batch of Mono thread dumps against framework source and pinpointing the faulty patterns; turning a confusing third-party API document into a draft SIMPL+ module as a starting point; updating XML doc comments across a large C# library; generating a full demo web UI quickly.
- **Verify everything.** AI-looked-up APIs are sometimes wrong, and generated device modules may not work as written, especially when the vendor's own API is unreliable.
- **Give the agent proper references.** An MCP tool that lets a coding agent navigate the SIMPL# Pro (SSP) help file helps with inheritance and events that the documentation explains poorly.
- **Persist what you learn.** Discord is good for real-time help but poor as a reference; pins become a dump and threads are hidden. Write lessons into the community knowledge base ([github.com/CrestronCommunity/KnowledgeBase](https://github.com/CrestronCommunity/KnowledgeBase)) or your own internal wiki.
- **Share how you work.** The community repeatedly noted that secrecy and embarrassment slow everyone down. Asking "basic" questions in public is how most people here learned.

## Quick-reference checklist

New project:

- [ ] Repo created from a template with folder structure, `.gitignore` and `.gitattributes`
- [ ] `.smw` / `.umc` / `.vtp` marked binary; large and compiled files in LFS
- [ ] All modules copied into the project folder (global `UsrMacro` / `UsrSPlus` empty)
- [ ] C# project targets the right framework (`net472` for 4-series, CF 3.5 toolchain for 3-series) with analyzers configured
- [ ] Post-build hooked to `AfterTargets="SimplSharpPostProcess"`
- [ ] Local builds report a `0.0.0-local` style version

Every load to site:

- [ ] Release build, tested as built
- [ ] Commit made when the code was loaded, and pushed (check `git log --branches --not --remotes` after offline work)
- [ ] Compiled output kept, either in the repo/LFS or as a release, with the database versions it was built against
- [ ] Panel text spell-checked

When changing a module's inputs or outputs:

- [ ] New version number in the filename, signals copied across, old version removed

Before leaving site:

- [ ] Debug session stopped from the console, not the IDE
- [ ] Remote access path confirmed for support (VPN, jump box or client-approved tool)
