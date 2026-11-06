# Crestron HTML5 / CH5 — Lessons Learnt

Nov 5, 2026 · @Rod Driscoll

## About this doc

Most CH5 pain comes from three things: projects load from `file://` on panels, panel Chromium is old and quirky, and WebXPanel needs the processor's certificate trusted. Get those right first.

This is distilled from about 1,700 messages in the CrestronProfessionals Discord channel #html-5-css-js between 30 Oct 2025 and 23 Oct 2026. It covers custom HTML5 UIs (React, Vue, Svelte, Angular, vanilla) more than Construct. Treat firmware and app-version specifics as true at the time they were posted and re-check them on site.

## Architecture and approach

The experienced developers in the channel keep Crestron's libraries thin and own the rest of the stack.

- **Use CrComLib for the minimum.** Several people use it only for reserved joins, panel detection (`isCrestronTouchscreen()`, `isIos()`) and the online state, then run everything else over their own WebSocket or SignalR. A common line was that CrComLib is bloat and raw bridge calls are enough.
- **Avoid Crestron's CH5 components where you can.** Using them ties you to Crestron's changes and bugs. The exception is video on panels (see Video).
- **Consider PepperDash's `@pepperdash/ch5-crcomlib-lite`.** It is a fork with the components removed and the imports fixed. Users reported it working without issues across several projects.
- **Contracts are effectively a dead end.** The Contract Editor produces invalid or empty JSON at times, and Crestron indicated in July 2026 it will not be developed further. Most people use join numbers or names, or their own protocol.
- **Publish and subscribe on the same join echoes back.** In a browser, publishing a join immediately updates its own subscription, as if input and output were wired together in SIMPL. Put feedback on a separate join.
- **The iframe wrapper pattern solves a lot.** A small ch5z holds CrComLib and loads the real app in an iframe served by the processor. The app then runs as a normal web page, not from a file. PepperDash's open-source wrapper app does this and adds a digital join to force an iframe reload.
- **Bundle the UI with the program.** Options discussed: embed the built UI in the .clz/.dll and serve it from your own web server, or add UI files to the .cpz and move them into place at startup. Either removes the "commissioner put files in the wrong folder" failure.
- **Own WebSocket vs SignalR.** Raw WebSockets work, but SignalR on ASP.NET Core (.NET 8 on 4-Series) gives reconnect logic, WSS and easy HTTP endpoints. The trade-off: it needs SIMPL#, not SIMPL+.
- **Large data over serial joins** works if you chunk it with your own start and end markers. Do not use 0x02 and 0x03 as markers; Crestron strips them.
- **Framework choice matters less than you think.** React, Vue, Svelte and Angular (up to v22) all run on panels. Svelte was reported as noticeably snappier on x60 panels. Astro and SSR frameworks are a poor fit because nothing server-side runs on the panel, and Node.js cannot run on a processor or panel.

## Build and tooling

Most "blank or black screen on the panel but fine in the browser" reports came back to one of the first three items below.

1. **Set `base: './'` in the Vite config.** On panels the project loads from `file://`, so absolute asset paths break. The same goes for `<base href>`: use `./`, not `/`.
2. **Use a memory router.** `BrowserRouter` and history-based routers change the URL, and the panel then looks for a file with that name. Use `MemoryRouter` (React) or the memory-history equivalent in Vue and Angular.
3. **Stop rgba() becoming #RRGGBBAA.** Vite and other minifiers convert `rgba()` to 8-digit hex, which panel Chromium did not render. Set `build.cssTarget: 'chrome61'`. Crestron fixed this in CH5 Renderer 1.3.0.172 (Aug 2026), but that needs app auto-update and internet access, so keep the build fix.
4. **Mind CrComLib's import change.** Between roughly 2.5 and 2.6 the import changed from `import CrComLib from …` to `import { CrComLib } from …`, without a major version bump. Older tutorials also tell you to edit CrComLib's package.json `main`; don't. If you load CrComLib and WebXPanel as UMD globals, copying them with `vite-plugin-static-copy` and initialising in `main.ts` worked well.
5. **Pin Node for ch5-cli.** `ch5-cli deploy` failed on newer Node LTS with "isDate is not a function"; Node 18 and 22 worked. A fix later allowed Node 25.
6. **`npm audit` will flag Swiper.** CrComLib 2.0.0–2.17.2 depends on a Swiper version with a critical prototype-pollution advisory and no fix.
7. **`@crestron/ch5-zoom-lib` 1.0.1 is broken as published.** Its package.json points at files that don't exist; import `dist/index.js` directly.
8. **Credentials for deploy.** The CLI can't take a password on the command line. Use environment variables or an SSH key, or script it yourself: deploy is just SFTP to the display folder and then `projectload`.
9. **vite-plugin-singlefile** helped on TSW-1060s, but was not needed (and caused issues) on 1070s.
10. **Watch npm supply-chain incidents.** The March 2026 axios compromise caught people updating old CrComLib projects; check what you pull in.

A standard Vite config shared in the channel:

```js
export default defineConfig({
  plugins: [vue()],
  base: './',
  build: { cssTarget: 'chrome61' }, // stop rgba() -> #RRGGBBAA
});
```

## Panel browser quirks

Target the panel's Chromium, not your desktop's. One developer tests in a downloaded Chromium matching the panel and gets about 97% fidelity.

| Panel | WebView (as reported in channel) | Known issues |
| --- | --- | --- |
| TSW-x60 | Roughly Chrome 95–98, Android 5 | `gap` unreliable; `<select>` fails anywhere a flexbox is used; no `:has()`; ch5-media-player unsupported; keep animations light |
| TS/TSW-x70 | Chromium 100–118 (WebView 118 on fw 3.003) | 8-digit hex colours (fixed in renderer 1.3.0.172); slow transitions on heavy pages |
| TS/TST-x80 | Chromium 122–123 | Much more fluid than x70 |
| DGE-1000 | Old Android | Video only works reliably with CH5 components |

- **Click is not hold.** A `click` only fires on a short press, so a long press registers nothing. Handle press and release yourself (pointer or touch start/end) and send high/low.
- **Fire on release where it matters.** If the state change on touch-start feels too fast, act on touch-end so users can also cancel.
- **The panel is a low-power device.** Freezes and lags seen only on panels were usually render loops or heavy reactivity. Set fixed image sizes, shrink images, avoid big mount-time logic, and test with Chrome CPU throttling.
- **Blur:** `backdrop-filter: blur()` worked on x70 and some x60 cases; Tailwind's blur classes were reported not to.
- **Pages freeze in the background.** When another app (e.g. Zoom) is in front, `visibilitychange` fires immediately and `freeze` about 5 minutes later. Messages queue and are processed on resume, so a stale "shutting down" modal can flash up. Fixes: debounce modals driven by processor state, or listen to `visibilitychange`, `freeze` and `resume` and tell the processor.
- **WebSockets pause in standby.** The TCP connection usually survives but nothing is processed, so keep-alives fail. Pause keep-alives on freeze and resync on resume, or wake the panel from the processor via the extender (`ExtenderSystemReservedSigs.BacklightOn()`).
- **A user agent of CRESTRON in Chrome DevTools** makes WebXPanel think it is on hardware (seen with Angular 22).
- **Disable iPad double-tap zoom** with `maximum-scale=1.0, user-scalable=no` in the viewport meta, and have only one viewport meta tag.

## Connectivity, WebXPanel and certificates

Certificates are the biggest recurring pain, and there is still no good answer for self-signed installs.

**Getting WebXPanel to connect**

- On a panel, CrComLib talks through the OS's CIP connection. In a browser you need the separate `@crestron/ch5-webxpanel` library, a WebXPanel defined in the program, and a WSS connection to port 49200.
- Processor settings to check: `webserver allowsharedsession on`, `ssl` on, `securewebsocketport` reports the server as running, `webinit` passes (it needs a config file in the html folder), and `userpageauth`.
- With `userpageauth` on, anything in the html folder needs a login. Turning it off does not stop `/cws/websockettoken` redirecting to `userlogin.html` when there is no session cookie.
- Running from `npm run dev` or another host needs a token: generate one on the processor (`websockettoken generate`) and pass it in.
- **Query parameters are case-sensitive:** `ipId` and `authToken`, not `pId` or `authtoken`. WebXPanel does not read them for you; parse `location.search` and feed the values in.
- One processor's secure WebSocket server stayed down until `webinit` passed and the log showed "Secure WebSocket server Successfully Created". Check `err plogcurrent`.
- `Csig` names in code use a lowercase `s` in `Csig.` (e.g. `Csig.Open_App`); `CSig` silently fails.

**Certificates**

- Self-signed works, but every browser must accept the cert at `https://<processor>:49200`, again whenever the cert changes or a new PC arrives. Browsers also start rejecting the WebSocket after a while until it is re-accepted or the processor reboots.
- Proper fixes: load your own root CA into clients' trust stores, use CA-issued certs, or use a wildcard cert with hostnames and DNS. All need IT cooperation and are hard to sell for isolated meeting-room networks.
- TSW panels and Crestron One on iOS reject certs valid for longer than about a year, even from a trusted private CA. Desktop Chrome and Firefox accept 5-year private-CA certs.
- The processor's self-signed cert has no IP SAN, which matters for iOS WebViews.
- Safari is the strictest. For iPads, Crestron One sidesteps certificates entirely.
- Cisco devices can load self-signed pages with `xConfiguration HttpClient AllowInsecureHTTPS: True`, `xCommand WebEngine AllowInsecureHttps Add Hostname: "<ip>"` and `AllowInsecureHttps: "True"` on `WebView.Display`.

**Plain HTTP and WS are being closed off.** In March 2026 a silent CH5 app update (1.2.3.139) stopped panels making cleartext HTTP/WS connections overnight, even on panels with app auto-update reportedly off. Crestron rolled it back, but plan for HTTPS/WSS everywhere.

## Reserved joins and panel integration

Subscribe to reserved joins by their `Csig.` name, not their number. Number subscriptions to 18494/18495 never updated for one developer, while the names worked straight away.

| Need | Join (name) | Notes |
| --- | --- | --- |
| Online state | `Csig.All_Control_Systems_Online_fb` | Reliable on panel and xPanel. False if any IP-table entry is offline |
| Offline state | `Csig.Control_Systems_Offline_fb` | Reported as unreliable |
| Panel docked (TST) | `Csig.Dock_Power_Status_fb` | Flickers on transition; debounce \~500 ms |
| Eject from dock | `Csig.Wallmount_Docking_Station_Force_Panel_Release` | Pulse true then false |
| Flip to Zoom app | `Csig.Open_App` / `Csig.Hide_Opened_App` | Needs Zoom set as the panel app with page flip enabled. `Csig.Close_App` fully closes Zoom |
| Sleep / wake | `Csig.Backlight_Off` / `Csig.Backlight_On` |  |
| Video window | `Csig.video.request` / `Csig.video.response` (type `o`) | See Video |

- Some reserved joins come from the hardware layer, so an xPanel may not drive them.
- There are no reserved joins for setting SIP credentials.
- If a vital join can be undefined (e.g. after a processor fault), give it a fallback value. One UI showed a blank white screen because a lock-screen join went undefined.
- The full list is in Crestron's [Reserve Join Viewer](https://sdkcon78221.crestron.com/downloads/rjviewapp/index.html); the docs page lists only some of them.

## Video

On panels, video plays behind the web page and only shows through a "hole". Most experienced developers avoid `<ch5-video>` for network streams and serve MJPEG instead.

**If you use `<ch5-video>` (needed for DGE-1000 HDMI and SIP/door-station video)**

- Anything with a background between the video and the page covers it. Remove every background, including on `body`, or make those layers `<ch5-background>`.
- Don't nest `<ch5-video>` inside `<ch5-background>`.
- Put one `<ch5-background>` at the base of the app, before the router. Placed inside routed components, it reaches up and changes parent classes, which is why the docs warn against Angular Router.
- For HDMI use `sourceType="HDMI" url="http:hdmi"`. RTSP is supported; HLS crashed a TS-1070.
- Sizing is fragile. The `size` attribute is ignored; wrap it in a fixed-size div with `stretch` and `aspectRatio="16:9"`. Keep width and height at or below 95% or it can go black. One fix needed max-width and aspect ratio on both the element and the `.ch5-video` class.
- Native full-screen hides the whole `body`, so you can't overlay a close button. Hide `.full-screen-icon` with CSS, or override its `.hide` class to show it permanently.
- In 2.17.0, styling was ignored and door-station URLs failed where 1.2.0 worked.
- Some found the HDMI input only worked once Tailwind, Skeleton and Svelte were removed and the CH5 template used.

**Rolling your own video object** (more control, used in production)

- Send an object of type `o` to `Csig.video.request`: `action` (`start`, `resize` or `stop`), a unique `id`, `source` (`{type:'HDMI', url:'http:hdmi'}`), `location` (top, left, width, height in px), credentials and timing. Listen on `Csig.video.response` for `status` (`started` = video OK; `connecting` or `error` = none).
- Always send `stop` when the window closes, or video stops working after a few days.
- Draw the background on a canvas and `clearRect` a hole where the video sits.

**Alternatives**

- Transcode RTSP, HLS or NDI to MJPEG with FFmpeg and show it in an `<img>` tag. FFmpeg can't be installed on a processor, so it runs on VC-4, a server or an appliance.
- For NVX previews, have the processor fetch the images via the NVX REST API and re-host them from memory on your own web server path. This also bridges control and LAN subnets and avoids CORS. One install ran about 100 endpoints this way from 2023 without complaints.
- Camera MJPEG with `user:pass@host` URLs is blocked by Chromium; proxy it through the processor.

## Platforms

Each host loads your project differently. Detect where you are at startup and initialise accordingly.

| Platform | Key lessons |
| --- | --- |
| **Crestron One (iOS/Android)** | Avoids certificate pain on iPads and runs custom HTML. Load the ch5z as a **mobile** project; uploading it as a web project makes only the first page work. Adds white bars top and bottom that you can't style. One CP3/AP3 issue opened the project only once on iOS (Crestron known issue). Images from the processor's /html folder may not render. Reconnect reliability was criticised |
| **Zoom Rooms (Crestron Control App)** | Configured separately from the older Zoom Room Controls JSON profile; set the JSON profile to status only if you run both. The app won't load from a web browser; needs an Android or iOS controller. `runsInContainerApp()` returns false here, so pass a flag such as `?zoomRoom=true`. Initialise `ch5zoomsdk` with CrComLib and WebXPanel before `WebXPanel.initialize`. The viewport is smaller than 1280×800. Self-signed certs on iPad were "a complete PITA". Custom (non-Construct) apps are possible but poorly documented |
| **Teams UC Engine** | Projects without CrComLib crashed Room Controls to a black screen; add CrComLib even to a demo. Load projects over CTP on port 41797 via XMODEM, or with Toolbox over SSL using the UC credentials. The user agent reports desktop, so hover states misbehave |
| **VC-4** | `ch5-cli deploy` doesn't work; upload the ch5z with the program in the Program Library (pencil icon), then restart rooms |
| **Crestron Virtual Panel** | Host, IP ID and room ID reach the page only through a GET of `./assets/data/project-config.json`. Custom apps must fetch that relative path on every load. The auth token arrives as `?authtoken=` |
| **TSW in General Web mode** | Needs valid HTTPS (FQDN and real cert). Not available on TSW-1060 (EMS is equivalent) |
| **Cisco Room Navigator / Touch** | Use the AllowInsecureHttps settings for self-signed processors (see Connectivity) |
| **Firefox** | WebXPanel failed until the :49200 cert was accepted |

A connection-config gotcha: a ch5z that reads its own `panel-config.json` with a `ws://` address works on an MTR, which runs it from local files, but a TSW never reads that file. Pass the processor address another way.

## Deployment, caching and updates

Panels update their CH5 app independently of firmware, and that can break working systems overnight.

- **Turn off application auto-update** (`appauenable off`) on deployed panels unless you control what it pulls. It is often on because it's needed at first boot to fetch the app. Check the actual CH5 app version with `ver -v`.
- **Recovering from a bad app push:** switch the panel's app mode away from User Project and back. In March 2026 this reverted CH5 from 1.2.3.139 to 1.2.3.130. With 350 panels affected, one integrator had to do it on every panel.
- **Offline sites can't get renderer fixes** without hosting the update files locally and editing the device manifest.
- **Stale cached content:** panels kept serving old HTML and JSON. The same User Project → Scheduling → User Project flip cleared it. `no-cache` was reused; `no-store` or versioned URLs forced fresh requests. Version your asset URLs.
- **Reloading after an update:** restart the program, pulse a reload join (if your app has one) or use `projectreload` on the panel.
- **ch5-cli vs Toolbox:** mobile projects that wouldn't deploy with the CLI went fine through Toolbox or SFTP plus `projectload`. Crestron One's "project is incomplete or missing files" error was once caused by an invalid remote address in the connection settings.
- **Avoid magic auto-refresh timers** as a fix for white screens. Find the root cause with proper debugging instead.
- **Use the right Zoom/Teams flows:** see Platforms; standard panel loading doesn't work on UC Engines.

## Debugging toolkit

You can't open DevTools on a panel, so build a console into your workflow before you need it.

- **[remotejs.com](https://remotejs.com/):** paste its script tag into `index.html` **above** your app bundle, so you see errors even when the framework never boots. The panel needs internet access.
- **[eruda](https://www.npmjs.com/package/eruda):** an on-screen console, better for inspecting in context on the panel. Works without internet.
- **No errors in the console** means it isn't throwing. Add `console.log` on each press to see whether presses even register during a freeze.
- **Reproduce panel conditions on desktop:** a Chromium build matching the panel version, plus CPU throttling.
- **Processor side:** `err plogcurrent`, `webinit`, `securewebsocketport`, `ver -v` (shows the forced-auth mode).
- **Network:** Proxyman on an iPad showed one app never even requesting the processor.
- **CH5 emulator:** call `run()` once, not on every render, or toggles and triggers misbehave.
- **Read the source:** CH5 components are on GitHub, though the npm package doesn't always match the repo.

## Checklist and references

**Before loading a new project to a panel**

- [ ] `base: './'` and relative asset paths
- [ ] Memory router
- [ ] `cssTarget: 'chrome61'` (or verified hex colours)
- [ ] Tested in a Chromium matching the panel (x60 ≈ 95–98, x70 ≈ 100–118, x80 ≈ 122)
- [ ] CrComLib present, even if barely used
- [ ] Reserved joins subscribed by `Csig.` name
- [ ] eruda or remotejs wired in
- [ ] Page visibility / freeze handling and a resync on resume
- [ ] HTTPS/WSS everywhere, no cleartext calls
- [ ] Panel app auto-update set deliberately
- [ ] Versioned asset URLs

**Before a WebXPanel goes live**

- [ ] `allowsharedsession`, `ssl`, `webinit`, secure WebSocket server running
- [ ] Decision on `userpageauth` and how users get the token
- [ ] `ipId` / `authToken` parsed from the URL
- [ ] Certificate plan agreed with IT (self-signed acceptance, private CA or wildcard)

**Reference links shared in the channel**

- [Crestron HTML5 UI docs — control system settings](https://sdkcon78221.crestron.com/sdk/Crestron_HTML5UI/Content/Topics/Platforms/X-CS-Settings.htm)
- [Web XPanel docs](https://sdkcon78221.crestron.com/sdk/Crestron_HTML5UI/Content/Topics/Platforms/Web-XPanel.htm)
- [Reserve Join Viewer](https://sdkcon78221.crestron.com/downloads/rjviewapp/index.html)
- [Crestron Control App for Zoom Rooms — deployment](https://docs.crestron.com/en-us/9421/Content/Topics/Installation/Deployment.htm)
- [CH5ComponentLibrary source (ch5-video)](https://github.com/Crestron/CH5ComponentLibrary/tree/master/src/ch5-video)
- [@pepperdash/ch5-crcomlib-lite](https://www.npmjs.com/package/@pepperdash/ch5-crcomlib-lite)
- [PepperDash touchpanel wrapper app (iframe pattern)](https://github.com/PepperDash/mobile-control-touchpanel-wrapper-app)
- [PepperDash React app core (own WebSocket)](https://github.com/PepperDash/mobile-control-react-app-core)
- [use-join-hook for React](https://github.com/nathan-hello/use-join-hook)
- [Vue 3 / Vite WebXPanel template](https://github.com/pkopysci/crestron-ch5-webxpanel-vue3)
- [Vue TS template](https://github.com/jphillips-eng/ch5-vue-ts-template)
- [CrestronCommunity KnowledgeBase](https://github.com/CrestronCommunity/KnowledgeBase)
- [CertManager scripts](https://github.com/ProfessorAire/CertManager)
- [Vite `build.cssTarget`](https://vite.dev/config/build-options#build-csstarget)

Source: CrestronProfessionals Discord, #html-5-css-js, messages from 30 Oct 2025 to 23 Oct 2026. The links above were shared in that channel and haven't been checked separately.
