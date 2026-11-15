hey can you implement webblox? it's roblox running in the browser as wasm. first try to pull the webblox vendor from roblox so we can get the actual player files

Reference notes (what we found from a HAR and from studying a working deployment, so you don't need either):

- Roblox ships an official browser client: binary type `WebClientApp`, an Emscripten build.
  - Current version: `GET https://clientsettings.roblox.com/v2/client-version/WebClientApp` → `clientVersionUpload` (e.g. `version-f807b21caba84809`).
  - Files: `https://setup.rbxcdn.com/webclient/<version>-RobloxWebClient/`. Core: `WasmPlayer.js`, `WebClient.js`, `WebClient.wasm` (~100 MB), `WebClient.metadata.json`, `audio-processor-fast.js`. The metadata JSON lists every content file (`{path, size}`, ~2900 files) at the same base. Directory listing is 403, so take paths from the metadata.
  - Flags: `https://clientsettingscdn.roblox.com/v2/settings/application/WebClientApp`. Pin them next to the engine version so a flag push can't break an older engine.
- `WasmLoader.js` and `stub.js` are not on the CDN. They're Roblox's small host-side loaders, and you can replace them. `WasmPlayer.js` only needs `window.__wasmLoaderContext = { engineBase, channel: null, server: "prod" }` set before it loads. If the engine is served from the page's own origin, no worker stub is needed.
- The host page needs the DOM ids `WasmPlayer.js` looks up:
  - `canvas`, `input-container`, `input-textarea`, `input-input`, `server`, `proxyenabled`;
  - the loading screen: `loading-overlay`, `status`, `.loading-error-panel`, `loading-error-title`/`-message`/`-code`/`-leave`, `loading-cancel`, `loading-experience-*`.
- The page must be cross-origin isolated (COOP `same-origin` + COEP `require-corp`), because the engine runs on pthreads over a shared `WebAssembly.Memory`. So it must be a top-level page in its own tab, not an iframe in the app.
- The engine needs `?placeId=`. Without one it logs "no startup place id; exiting", so build a small launcher: game search via `apis.roblox.com/search-api/omni-search`, plus pasting a game link.
- Networking:
  - All the engine's HTTP goes to `*.roblox.com` / `*.rbxcdn.com` via fetch/XHR, from the page and from its workers. A service worker scoped to the player catches those cross-origin requests and hands them to the page. The page sends them over Wisp with epoxy (TLS ends in the browser) and keeps a cookie jar in localStorage.
  - Synthetic responses need CORP `cross-origin` plus ACAO / ACAC / expose-headers, so the engine can read `x-csrf-token`.
  - Game traffic is WebTransport straight to `*.qdmux.roblox.com`, not through Wisp.
- Join quirks (from a working deploy):
  - POST to `gamejoin.roblox.com` with `rbx-user-agent: Roblox/Darwin`.
  - If `joinScript.gameFqdn` is missing, build it from the join response: `<netstack token hex>-<MachineAddress hex>-<NetStackPort hex>.<UdmuxEndpoints[0].Address with last octet +1, hex>.qdmux.roblox.com`.
  - For other Roblox requests, rewrite `rbx-user-agent` "Roblox/Web … Desktop" → "Roblox/WinInet" and drop "Browser/…".
  - To join a specific server, rewrite `/v1/join-game` to `/v1/join-game-instance` with `gameId`.
  - Joining needs a `.ROBLOSECURITY` session. Without one you get error 524 "Not authorized".
- Sign-in and launching:
  - Users sign in to roblox.com inside the app's Scramjet browser (email one-time code; audio CAPTCHA). Scramjet 2's controller saves its cookie jar in IndexedDB (`__scramjet_controller` → store `state` → key `cookies`, a JSON dump keyed `domain@path@name`). The player page, on the same origin, reads `.ROBLOSECURITY` from there.
  - A Scramjet frame plugin catches roblox.com launching the desktop app and opens the player instead. That covers `roblox-player:` / `roblox://` URIs set as a hidden iframe `src` (by property or `setAttribute`), passed to `window.open` or used as a link, and `/games/start?placeId=` (by link, `window.open`, or top-level navigation).
  - For "Join" on a friend, the RequestFollowUser launch carries a userId; look up their presence (`presence.roblox.com/v1/presence/users`) to get the place and server.
