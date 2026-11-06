# Crestron 4-Series Programming – Lessons Learnt

Nov 5, 2026 · @Rod Driscoll

The biggest lessons from a year of 4-series discussion: prefer non-router processors (CP4/RMC4 over CP4N), never rely on multicast crossing the processor's NAT, use EISC (Packed) for inter-slot comms, and treat every firmware update as a potential breaking change to security and console behaviour.

These notes are distilled from roughly 465 messages in the CrestronProfessionals Discord channel [#4-series-processors](https://discord.com/channels/314425881994985472/729826089282830407), covering September 2025 to October 2026. They are field reports from integrators and programmers, not Crestron documentation, so verify against current firmware before relying on them.

## Hardware selection and CP4N router reliability

The internal router (CEU) on control-subnet processors is the most-reported source of instability, and it runs even when the CS port is unused.

- **Prefer CP4 or RMC4 over CP4N** unless you genuinely need the control subnet. Several members now spec only LAN-only processors after repeated CP4N failures; one swapped a misbehaving CP4N for a CP4/RMC4 and recommended it as the first fix.
- **Know the symptoms:** LEDs stay on but the processor drops off the network; ping and TCP to specific devices fail intermittently; logs fill with `RQIF_IssueGetCommand - ROUTER NOT CONNECTED`, `CEU_SendRecvMsgWithRetry` and DHCP release/renew errors. A reboot clears it for hours.
- **Check the router firmware with `ver -v`.** Reports tie problems to RouterVersion 3.000.0016 and also 3.000.0042, with identical units on the same version behaving differently. The version alone does not predict failure.
- **Recovery trick reported for .0016 units:** power down and let the unit cool for about 20 minutes, connect over USB, then push firmware that moves the router past .0016. Heat in warm racks is suspected but unproven.
- **Swap-test before blaming code.** If the same program runs fine on a different unit on the same network, treat the processor as faulty and open an RMA with the logs.
- **TLDM `GET VERSION ERROR: Invalid parameters` spam** every few seconds on CP4N/PRO4 is a known router-related log flood; no fix was posted.
- **MPC4-3 now comes in two variants:** 301 (one button row, Ethernet only) and 302 (two rows, Ethernet plus COM/IO). Check which one is specified.

## Networking: multicast, NAT, UDP and ports

The processor's NAT router forwards unicast only, so design cross-subnet and discovery traffic around unicast to the processor's LAN IP.

- **Multicast will not cross the CP4N router.** Port forwards apply to unicast; the router does no PIM, so packets from other VLANs never reach the control subnet (only same-subnet packets got through in one test). Use unicast UDP/TCP to the processor's LAN address instead.
- **Long-running UDP can silently stop.** One CP4 stopped sending UDP to 12 PTZ cameras after extended uptime while TCP kept working; only a reboot restored it. Prefer per-device TCP control, or monitor UDP and alert when it stalls.
- **Crestron's registered ports:** 41794 CIP, 41795 CTP (console), 41796 secure CIP (SCIP), 41797 secure CTP. High ports such as 49417 in logs are the client side of the connection.
- **Link speed:** `ipconfig /all` on a CP4N reports the internal FEC1 link, not the LAN or CS port speed.
- **ARP table:** use the full command `showarptable` on 4-series; `showarp` is the 3-series/DMPS form.
- **Timezones:** set with `timezone <n>` (list with `timezone ?`). You pick a city per zone and DST follows it; you cannot enter a raw UTC offset. If your code maps zones, use IANA names, not descriptions such as "Atlantic Standard Time", which is ambiguous between regions with and without DST.

## Inter-program comms: choosing an EISC

Default to EISC (Packed) for slot-to-slot comms on the same 4-series processor; switch to the TCP variants only for VC-4.

| EISC type | Transport | Works to 127.0.0.2 (same processor) | Notes |
| --- | --- | --- | --- |
| EISC (Packed) | UDP | Yes | Packs many signal changes per packet; lowest chatter. Interoperates with standard EISC. Recommended default. |
| EISC | UDP | Yes | One packet per signal. No reason to choose it over Packed. |
| 3-Series TCP/IP EISC | TCP | Yes | Handles initialisation more robustly; reported as the safe choice for localhost. |
| EISC Client / Server | TCP | No | Fails on localhost with IP-ID-already-in-use errors. Useful for VC-4. |

UDP vs TCP rarely matters on a healthy LAN; the bigger difference is when and how each type connects and syncs state.

## Security, certificates, web server and CWS

Authentication is enforced by default on 4-series, and recent firmware has removed or changed certificate features without much notice, so plan PKI outside the processor.

- **Generate CSRs and certs off-box.** Recent firmware removed IP addresses as Subject Alternative Names in `CREATECsr -S` (Crestron confirmed it was removed for security), and Toolbox's "set CA" now fails because it sends an unsupported `tls` flag. Use OpenSSL or a script to build your own root CA, CSRs and SAN-IP certs, then load the root CA with `ssl ca`.
- **Self-signed certs have no IP SAN.** Some clients (notably iPads) may reject them. Compare the cert on 443 and on 41796 with `openssl s_client -connect <ip>:<port> -showcerts`.
- **iPads failing SCIP with `SSL Negotiation not complete`** while browser HTTPS works was seen on a residential CP4 with about a dozen iPads; moving the processor to a new IP was the only workaround found.
- **Regenerating the self-signed cert:** there is no direct command. `CERTIFICATE SELFEXPIREREBOOT FALSE` was suggested as a lever; changing the hostname also forces a new cert.
- **`certificate add websocket`** does not cover port 49200; update that cert with `ssl ca`.
- **Serving static HTML:** set `userpageauth off` to serve files from `\html` without login or HTTPS. Auth still applies once the page talks to a program over CIP/SCIP. Use the full filename in the URL (e.g. `/index.html`); a meta-refresh page there can redirect users elsewhere.
- **`userpageauth` does not affect CWS token routes** such as `/cws/websocket/getWebSocketToken`.
- **`cwsbasicauth` did not challenge programmatically added CWS routes** in one test; browsers caching basic-auth credentials can also mislead testing. Plan your own auth for custom CWS endpoints.
- **CP4N: CWS is the only way to serve HTTP to LAN clients from an SSP program.** On CP4N, the LAN NIC lives in the firmware's network namespace and is invisible to the .NET process — `System.Net` listeners (Kestrel, HttpListener) only bind `172.22.x.x`. CWS (`HttpCwsServer`) routes through the firmware's port-80 web server on the LAN NIC, so it reaches LAN clients. See the SIMPL# Pro doc "Web hosting: CWS vs ASP.NET Core" section for CWS streaming limits and long-polling patterns.
- **SSH after firmware upgrades:** a newer OpenSSH with FIPS mode on changed the default key exchange and broke a legacy console app. Use `ssh -vv user@host` to see the offered algorithms.
- **RADIUS logins:** domain users authenticated fine over SSH but the web UI looped on 301 redirects (open TrueBlue case at the time).

## Touch panel and UI gotchas

Act on release, not press, whenever the press itself hides or moves the button.

- **Stuck presses:** if a button (including a subpage reference list item) is hidden while the finger is still down, the release may never arrive and the press stays high until the next full press/release. Trigger page flips, popup hides and visibility changes on the falling edge of the press.
- **WebXPanel limit:** Toolbox loads a maximum of 3 WebXPanel projects on a CP4N, but more can be added by other means.
- **CH5 redirects** from the processor root to a CH5 app are inconsistent; sometimes the panel setup pages appear instead. No pattern was identified.
- **Zoom Rooms on CP4N** require the two `jwtpublickeysource` commands; `jwtprocessor # Token validation failed` errors point at that setup.
- **AV Framework:** loading a SIMPL program does not disable AVF in slot 00. Unconfigured, it mainly registers an IP-ID for XPanel; on MPC4 a known issue has it wrongly claim the front panel, so disable it there when unused.

## Program loading, file system and deployment

When Toolbox struggles, deploy by SFTP and console; it is faster and easier to debug.

- **Manual load:** SFTP into the processor, clear `program01`, upload the `.lpz`, then run `progload -p:1` over SSH. Watch progress from a second console session.
- **Slow progload:** 20–30 s at "Looking for lpz/cpz files" is common; the program is stopped first, then unzipped. SSH and Toolbox may be unresponsive meanwhile.
- **Slow SFTP transfers:** a different client (e.g. FileZilla) can be much faster than Toolbox. Equally, some firmware has broken third-party SFTP clients, so keep both options.
- **Program folders empty after load** on the March 2026 firmware (v2.8006.00284) needed a reboot; a later release fixed "an issue found in production".
- **`CSPROJECTREMOVE -p:all` also deletes the processor's setup application.** Remove projects individually, or plan for a firmware reload/restore afterwards.
- **`user`, `User` and `USER` are different folders** on 4-series, with different contents; FTP and the `dir` command show different views. Check paths carefully when reading or writing files from code.
- **Processor-specific load failures:** a SIMPL program with a S# → S+ library started on a CP4 but not on the DIN-AP4 or MC4 it targeted, on identical firmware. Test on the actual target hardware.
- **Event Scheduler 2 v1.2.0** crashed a CP4N program at init; no working 4-series alternative was confirmed.
- **`PROGRESET` does not clear stuck CWS framework threads.** CWS `ProcessRequest` handlers run on unmanaged firmware threads. If a handler never returns (e.g. a hung SSE loop), the thread survives `PROGRESET`/`PROGLOAD` and the new program run finds the CWS path stale-registered — `Register()` returns `false`. The thread also blocks PROGLOAD from completing its "waiting for un-registration" step. Only a full `REBOOT` clears them. Prevention: always bound `ProcessRequest` execution time, and subscribe `ProgramStatusEventHandler` to call `HttpCwsServer.Dispose()` on `eProgramStatusEventType.Stopping` before PROGLOAD restarts the slot.
- **`PROGSTOP -P:<slot>` does not exist on CP4N** ("Bad or Incomplete Command"). Use `stopprog -p:<slot>` to stop a slot without restarting it, or `PROGRESET -P:<slot>` to stop and restart.

## Console command cheat sheet

Append `?` to any command to list its options; `hidhelp all` shows hidden commands.

| Command | Purpose |
| --- | --- |
| `ver -v` | Full version info, including RouterVersion on CP4N |
| `ipconfig /all` | Detailed network info (CP4N shows the internal FEC1 link speed) |
| `showarptable` | ARP table (must be typed in full on 4-series) |
| `timezone ?` / `timezone <n>` | List and set the timezone |
| `progload -p:<slot>` | Load the program already in a slot's folder |
| `stopprog -p:<slot>` | Stop a program slot without restarting (`PROGSTOP` does not exist on CP4N) |
| `progreset -p:<slot>` | Stop and restart a program slot |
| `userpageauth off` | Serve `\html` files without login |
| `CREATECsr -S` | Create a CSR with SAN (IP SANs removed on recent firmware) |
| `ssl ca` | Load or update CA certificate |
| `CERTIFICATE SELFEXPIREREBOOT FALSE` | Stop auto-reboot when the self-signed cert regenerates |
| `webinit` | Reported to stop the redirect to `/setup` |
| `flipimage` | Recover a processor stuck at the `INGOLDEN>` prompt |
| `jwtpublickeysource` | Required JWT setup for the Zoom Rooms control app |

## Firmware management

Release notes are often vague ("Fixed an issue found in production"), so test every update on a bench unit before rolling it out.

- **Expect silent behaviour changes:** removed CSR SAN options, a new OpenSSH default key exchange, and a Toolbox/firmware mismatch on SSL commands all arrived in point releases.
- **Keep the previous firmware to hand.** Downgrading was the quickest fix for the CSR change.
- **Don't deploy a release that a newer one quickly replaces** without reading what changed; one release was superseded within two weeks.
- **Failed processes can't be killed.** The console's bash shell is unprivileged, so a hung program slot or a crashed BACnet service needs a reboot (the watchdog rebooted one processor twice when BACnet died). Build programs to recover cleanly from a full restart.
- **.NET roadmap lag:** 4-series .NET support trails Microsoft's support cycle (net472, then .NET 6, then .NET 8). Weigh this if you move logic to full C# or off-box services.

## Design principles

Fix root causes and use real feedback from devices; timers and scheduled reboots hide problems rather than solving them.

- **Avoid time-based positioning** (lifts, screens, shades). Without position feedback, drift is inevitable as motors age. If you must use timing:
  - Drive into a hard mechanical limit first, then time away from it, so each move self-calibrates.
  - Time up and down separately; the motor runs at different speeds each way.
  - Add padding (e.g. 0.5 s) and provide manual jog controls so a technician can recalibrate.
- **Don't schedule reboots as a fix.** They wipe in-memory state that isn't persisted and mask the real fault. Note that -R processors already run a daily reboot check that only reboots on high memory use.
- **Persist the state you care about.** Some extension modules and devices lose state on restart; design for clean recovery after a reboot.
- **Offload heavy work wholesale.** If a task strains the processor, move all of it to a separate machine and size that machine; don't try to balance load across both. Benchmark rather than reasoning from CPU specs.
- **Don't tune CWS around thread counts.** These are low-core processors; design for efficient async handling rather than parallelism.

## Recovery and hardware maintenance

Most "bricked" processors are recoverable from the console or by cloning the SD card.

- **`INGOLDEN>` prompt** (pingable, SSH works, web UI dead): the unit booted its golden image. Run `flipimage` to return to the main image.
- **Console errors such as `Invalid Feat value` / `PUF Unknown`** on a CP4 left console unusable; download logs from the web page instead.
- **Failing SD cards (MC3/MC4):** image the card with `dd`, then write it to an industrial card (ATP is the stock brand). MC4 ships with 8 GB; MC3 with 2 GB (ATP AF2GUDI-EMI001L001), and MC3s are picky about substitutes. Larger cards of equal or better speed class work in touch panels. The MAC address must be edited on the cloned image.
- **Gather evidence before an RMA:** export error logs and, if needed, a Wireshark capture; explicit `LAN disconnected` or router errors make support cases faster.
- **License keys** can be added from the console rather than Toolbox (an example was shared in the channel).

## Sources

- CrestronProfessionals Discord, [#4-series-processors](https://discord.com/channels/314425881994985472/729826089282830407), messages from 12 Sep 2025 to 12 Oct 2026.
