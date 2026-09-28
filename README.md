# Just-Finish-It-Fleet

The fleet dashboard for [Just-Finish-It](https://github.com/Tejasvedagiri/Just-Finish-It) (JFI) sessions: a small Node WebSocket server (`server/master.js`) plus a framework-free JS frontend (`src/`), meant to run on its own machine (or just a different terminal) and watch *multiple* JFI sessions anywhere over the network. A session and the master only ever talk over one WebSocket — never a shared filesystem — so this repo has zero dependency on JFI's Python code or vice versa; the wire protocol (plain WebSocket + JSON) is the only thing connecting the two.

This was previously a subdirectory (`frontend/`) inside the `Just-Finish-It` repo and has been split out as its own standalone project.

## Run it

```bash
npm install
npm run build     # builds the fleet UI once, into dist/
npm run master    # serves it on :8765 (MASTER_PORT to change)
```

Open `http://localhost:8765` (or whatever host/port `master.js` is bound to) for the fleet UI.

For local development with hot reload against a master already running elsewhere:

```bash
npm run dev
# or, if the master isn't on the default port 8765:
MASTER_DEV_PROXY_TARGET=ws://localhost:9988 npm run dev
```

## Pointing a JFI session at this dashboard

On each JFI session you want this dashboard to watch (any machine that can reach the master's port):

```bash
MASTER_WS_URL=ws://<master-host>:8765/report ./jfi
```

A session identifies itself to the master with an opaque `hostname::session-id` string and a short repo-directory label, never a filesystem path — the master has no way to read a remote session's disk.

## Layout

```
server/
  master.js             # npm run master — the fleet WebSocket server + static file host for ../dist/
  session-registry.js   # SessionRegistry/deriveEvents: master.js's in-memory fleet state, no dependency on ws/http (unit-testable alone)
src/
  main.js                # WebSocket client, all rendering, tab switching, theme dropdown — no framework
  themes.js               # the same 20 theme presets JFI's own terminal supports, expanded into this app's full CSS token set
  style.css                # component styles + a static-fallback token set (themes.js overrides these live via main.js)
vite.config.js           # dev-only: npm run dev + hot reload, proxying /view + /report to the master (MASTER_DEV_PROXY_TARGET)
```

See `Just-Finish-It`'s own README ("Fleet dashboard" section) for the full tab-by-tab walkthrough of what the UI shows.
