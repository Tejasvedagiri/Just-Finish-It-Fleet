# Just-Finish-It-Fleet

A live web dashboard for **[Just-Finish-It](https://github.com/Tejasvedagiri/Just-Finish-It)** (JFI) sessions — every session, on every machine, on one page.

It's a small Node server (`server/master.js`) plus a framework-free frontend (`src/`). JFI sessions report to it over a WebSocket; your browser watches them over another. The master never touches a session's filesystem, so it can run on its own machine, and this repo has no dependency on JFI's Python code (or the other way round) — plain WebSocket + JSON is the only link.

## What you see

- **Fleet** — every session as a card: phase (Planner → Implement → Reviewer → Cleanup), the current stage (Architect / Lead / Task / Judge / Dev / Review / Cleanup), what it's working on, progress and tokens. Sessions waiting for an answer are pulled to the top.
- **Activity** — a feed of phase changes, prompts waiting for input and finished runs, across the fleet.
- **Session** — one session in detail, with a controls bar (**Pause / Resume**, **Stop**, and a box to **queue** a follow-up request) and three sub-tabs:
  - **Overview** — the phase timeline, progress, the prompt it's waiting on (answerable from here), the current task, token cost per task, background processes, and the live log.
  - **Plan checklist** — the plan tree with a per-leaf detail view.
  - **Task | Judge** — every plan node with the planner judge's scores side by side (the rule, Laya with its confidence, the LLM tie-break), the final verdict, who decided, the review result and the status; then the Architect's **runbook** and **design**.
- **Session DB** — browse the session's own database tables. The query runs on the session's machine; only the rows come back.

## Getting started

You need **Node.js 18+** and a JFI checkout (see [JFI's Getting started](https://github.com/Tejasvedagiri/Just-Finish-It#getting-started)).

### 1. Run the master

```bash
git clone https://github.com/Tejasvedagiri/Just-Finish-It-Fleet.git
cd Just-Finish-It-Fleet
npm install
npm run build     # builds the UI into dist/
npm run master    # serves the UI and both WebSockets on :8765
```

Open **http://localhost:8765**. After pulling changes, run `npm run build` again; the master serves whatever is in `dist/`, and the page always revalidates, so a reload picks the new build up.

### 2. Point JFI sessions at it

In JFI, sync the `master` extra (it adds the WebSocket client), listing every other extra you use too:

```bash
cd /path/to/Just-Finish-It
uv sync --extra web --extra master
```

Then set this in the `.env` of each project JFI runs in — `uv run create-env` asks for it too, and accepts just the port:

```bash
MASTER_WS_URL=ws://<master-host>:8765/report
```

Start JFI as usual; the session shows up within a second and reconnects on its own if the master restarts. A session is identified by an opaque `hostname::session-id` and its folder's name — never a filesystem path.

### Settings

| Variable | Where | Default | What it does |
|---|---|---|---|
| `MASTER_PORT` | master | `8765` | The one port for the UI, `/report` (sessions) and `/view` (browsers). |
| `MASTER_HOST` | master | `0.0.0.0` | The address to bind; `127.0.0.1` keeps it local-only. |
| `MASTER_FRONTEND_DIST` | master | `dist/` | Where the built UI is served from. |
| `MASTER_DEV_PROXY_TARGET` | `npm run dev` | `ws://localhost:8765` | The master that the dev server proxies `/view` and `/report` to. |

### Developing the UI

```bash
npm run master    # in one terminal
npm run dev       # in another: Vite with hot reload, proxied to the master
MASTER_DEV_PROXY_TARGET=ws://localhost:9988 npm run dev   # a master on another port
```

## Security

The master has **no authentication**. Anyone who can reach its port can watch every session, post fake status, and pause, stop or queue work on a session. Run it only on a network you trust (localhost, a VPN, a firewalled LAN), or bind it to `127.0.0.1` with `MASTER_HOST`.

## Layout

```
server/
  master.js             # npm run master: the WebSocket server + static host for dist/
  session-registry.js   # the fleet's in-memory state and its activity events (no ws/http dependency)
src/
  main.js               # WebSocket client, every tab's rendering, theme picker -- no framework
  themes.js             # the same 20 theme presets as JFI's terminal, expanded to this app's CSS tokens
  style.css             # component styles + a fallback token set
vite.config.js          # dev only: npm run dev, proxying /view and /report to the master
```

## Keeping in step with JFI

The two repos share no code, only a message shape. When one changes, the other usually needs the matching change:

- **Phases** — `PHASES` in `src/main.js` mirrors JFI's `runner.PHASES` (`planner`, `imp`, `reviewer`, `cleanup`).
- **The status snapshot** — what a session sends each second (JFI's `get_status_snapshot`). The Task | Judge tab reads its `plan_detail` (`rows`, `runbook`, `design`); the row keys (`#`, `Level`, `Task`, `Judge`, `Laya`, `LLM`, `Final`, `Decided by`, `Review`, `Status`) come from JFI's `plan_db_tools.plan_judge_rows`.
- **The plan checklist** — parsed from the snapshot's `plan_markdown` (`- [ ] 1.2 description` lines under `## Implementation`).
- **Session DB** — `DB_TABLES` in `src/main.js` mirrors JFI's `db_browse.table_registry()`.
- **Controls** — `pause`, `resume`, `stop`, `queue`, `answer` and `db_query` messages are handled by JFI's `socket_reporter.py`.

This repo was split out of JFI, where it used to live as `frontend/`.
