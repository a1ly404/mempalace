# Derby Stack — Project Context

> **Last updated:** 2026-04-10
> **Purpose:** Persistent context for AI assistants across Zed, Cursor, and Claude Code sessions.
> Add this repo to any workspace so new threads can read this file and get up to speed immediately.

---

## Architecture Overview

```
CRG ScoreBoard (Java, port 8000/8002)
    │ WebSocket
    ▼
derby-scoreboard-api (Python/FastAPI, port 5001)
    │ REST: GET /live, /health, /raw
    ├──────────────────────┬──────────────────────┐
    ▼                      ▼                      ▼
derby-stat-tracker     derby-scoreboard-      live-bridge
(React monorepo,       display (overlays      (Node.js poller
 port 5173/5175)       in CRG custom/view/)   → Supabase)
```

## Repositories

### derby-scoreboard-api
- **Repo:** github.com/a1ly404/derby-scoreboard-api
- **Tech:** Python, FastAPI, async WebSocket client
- **Key files:** `main.py`, `client.py`, `models.py` (source of truth for API types), `proxy.py`
- **Run:** `python main.py` (default: ws://localhost:8000, serves on :5001)
- **Test:** `pytest`
- **CRITICAL:** `models.py` is the single source of truth. Never hand-edit the generated TypeScript types downstream.

### derby-stat-tracker
- **Repo:** github.com/a1ly404/derby-stat-tracker
- **Tech:** TypeScript, React, Vite, npm workspaces, Tailwind v4
- **Workspaces:**

| Package | Path | Port | Purpose |
|---------|------|------|---------|
| @derby/web | apps/web | 5173 | Main web app (manual stat tracking, auth, dashboard) |
| @derby/live-tracker | apps/live-tracker | 5175 | Live scoreboard tracker (polls API) |
| @derby/live-frontend | packages/live-frontend | — | Shared React components |
| @derby/live-bridge | services/live-bridge | — | Node.js poller → Supabase persistence |

- **Run:** `npm run dev` (web), `npm run tracker:dev` (live-tracker), `npm run bridge:dev` (bridge)
- **Test:** `npm run test` (unit), `npm run e2e` (Playwright)
- **Type sync:** `npm run sync:api-types` (requires API running on :5001)
- **Auto-generated file:** `apps/live-tracker/src/types/scoreboard-api.ts` — NEVER hand-edit
- **CI:** GitHub Actions — unit tests, build, Playwright E2E, type-check, Vercel deploy
- **Ruleset:** "keep prod safe" on main — requires PR, CodeQL, Vercel Preview deploy

### derby-scoreboard-display
- **Repo:** github.com/a1ly404/derby-scoreboard-display
- **Purpose:** Houses overlay submodules/folders, Playwright overlay test suite
- **Overlays (in custom-overlays/):**
  - `eod-custom-overlay/` — broadcast overlay (team bars, scores, lead flash, WCAG)
  - `eod-commentator-overlay/` — full-screen roster for commentator booth
- **Test:** `npx playwright test` (overlay E2E, 191+ tests)
- **Install overlays:** Copy desired folder to CRG `html/custom/view/`

### eod-custom-overlay (standalone repo)
- **Repo:** github.com/a1ly404/eod-custom-overlay
- **Structure:** Being restructured into `broadcast/` and `commentator/` subfolders (PR #10)
- **Key features:**
  - WCAG AA lead-jammer flash (two-colour pulse, peak >= 4.5:1, trough >= 3.0:1)
  - Input validation (`isValidHex` guard)
  - Same-colour conflict auto-adjustment (lightens/darkens Team 2)
  - CSS custom properties for flash colours (`--teamN-flash-peak`, `--teamN-flash-trough`)

### scoreboard
- **Local CRG ScoreBoard install** (Java)
- **Run:** `java -jar lib/crg-scoreboard.jar --port=8002`
- **Custom overlays live at:** `html/custom/view/<overlay-name>/`

---

## Conventions

### API Fields
- `snake_case` for all API field names (Python/Pydantic)
- `camelCase` for TypeScript/React local variables
- Clock values in **milliseconds** (suffix `_ms`), parallel human-readable `str` fields
- Boolean flags: present tense (`in_jam`, `in_box`, `jam_running`, `in_lineup`)
- Nullable fields: `Optional[T] = None` (API), `T | null` (TypeScript)

### Overlay URLs
```
/custom/view/eod-custom-overlay/index.html?home=%23HEX&away=%23HEX
```
- Path is under `html/custom/view/`, NOT `html/custom/`
- `#` in hex colours must be URL-encoded as `%23`

### Type Sync Flow
1. Change `derby-scoreboard-api/models.py`
2. Run the API: `python main.py`
3. In derby-stat-tracker: `npm run sync:api-types`
4. Generated file: `apps/live-tracker/src/types/scoreboard-api.ts`

---

## Current State (as of 2026-04-10)

### Recently Merged
- **derby-stat-tracker #11** — Full monorepo restructure, E2E coverage, live pipeline, database setup, Vercel deployment
- **eod-custom-overlay #9** — WCAG lead flash CSS properties, input validation, same-colour conflict adjustment

### Open PRs
- **eod-custom-overlay #10** — Restructure repo into `broadcast/` and `commentator/` subfolders

### Closed/Superseded
- **eod-custom-overlay #1** — Old commentator overlay PR (was labelled DONT MERGE, closed in favor of #10)

### Known Issues / Next Steps
- Live-tracker `[object Object]` bug was fixed (team names were objects not strings) — now reads `team1.name`
- Default API URL set to `http://localhost:5001/live`
- Live-tracker auto-connects and polls by default
- Consider adding localStorage persistence for saved API URL and poll interval
- CodeQL workflow exists but does not always trigger on PR branches — may need investigation
- `packages/live-frontend` still pins Vite ^5.4.1 (rest of monorepo is ^7.2.6)
- Template files in live-tracker need cleanup (Spark README, GitHub SECURITY.md, LICENSE attribution)

---

## Local Dev Quick Start

```bash
# Terminal 1 — CRG ScoreBoard
cd scoreboard
java -jar lib/crg-scoreboard.jar --port=8002

# Terminal 2 — Scoreboard API
cd derby-scoreboard-api
python main.py --scoreboard-port 8002

# Terminal 3 — Live Tracker
cd derby-stat-tracker
npm run tracker:dev

# Terminal 4 — Web App (optional)
cd derby-stat-tracker
npm run dev
```

### URLs

| Service | URL |
|---------|-----|
| CRG ScoreBoard | http://localhost:8002 |
| Scoreboard API | http://localhost:5001 |
| API live state | http://localhost:5001/live |
| Live Tracker | http://localhost:5175 |
| Web App | http://localhost:5173 |
| Broadcast Overlay | http://localhost:8002/custom/view/eod-custom-overlay/index.html?home=%23cc1100&away=%231f3264 |
| Commentator Overlay | http://localhost:8002/custom/view/eod-commentator-overlay/index.html |

---

## MemPalace Integration

This context file lives in the [mempalace](https://github.com/milla-jovovich/mempalace) repo
so it can be added to any AI-assisted workspace (Zed, Cursor, Claude Code) for persistent
cross-session context.

### Zed
Add the `mempalace` folder to your Zed project (File → Add Folder to Project).
Any new assistant thread will see `derby/CONTEXT.md` in its project tree.

### Cursor
Add mempalace as an MCP server in `~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "mempalace": {
      "command": "python3",
      "args": ["-m", "mempalace.mcp_server"],
      "cwd": "/Users/ally.beaumont/Documents/derby/mempalace"
    }
  }
}
```
Or simply add the mempalace folder to your Cursor workspace for file-based context.

### Claude Code
```bash
claude mcp add mempalace -- python -m mempalace.mcp_server
```

### Keeping This File Updated
After significant changes to the derby stack (merged PRs, new repos, architecture changes),
update the "Current State" section and bump the "Last updated" date at the top.
