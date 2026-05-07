# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

BBterminal is a two-process local app: a Python data backend (OpenBB Platform) and a React/Vite frontend. They run on separate ports; the UI proxies `/api` to the backend.

```
Browser  ──▶  Vite dev :5173  ──/api──▶  openbb-api :6900  ──▶  Yahoo / FRED / SEC / …
```

- **Backend**: `openbb-api` (FastAPI, installed into `.venv` by `setup.sh`). Provides `/api/v1/...` endpoints. Not in this repo — the binary lives at `.venv/bin/openbb-api` after install. There is no Python source to edit.
- **Frontend**: everything in `app/`. Vite + React 18 + TypeScript + Tailwind. State is Zustand (`workspaceStore`), data fetching is TanStack Query, charts are TradingView `lightweight-charts`. The Vite proxy in `app/vite.config.ts` forwards `/api` → `127.0.0.1:6900`.

### How a function (screen) works

The terminal is a tabbed workspace where each tab is a *function* identified by a 2–6 letter code (`CC`, `INTEL`, `GP`, `OMON`, …). The flow:

1. User types a command in `CommandBar`. `parseCommand` in `app/src/lib/functions.ts` turns free-form input (`AAPL`, `TSLA INTEL`, `WEI`, `KEY TSLA`) into `{ code, symbol? }`. Bare ticker → `INTEL`; bare function with `needsSymbol: true` reuses `activeSymbol` from the store.
2. `workspaceStore.openTab` either focuses an existing tab (id = `${code}:${symbol ?? "_"}`) or appends a new one. The store is persisted to localStorage under `bbterminal-workspace`, so tabs survive reloads.
3. `App.tsx` looks up the active tab in the `SCREENS` map and renders the corresponding component from `app/src/functions/`. Each function component is responsible for its own data fetching via React Query, calling typed wrappers in `app/src/lib/api.ts`.

**Adding a new function** requires updating four places: register it in `FUNCTIONS` (`lib/functions.ts`) and the `FunctionCode` union, add the component under `functions/`, wire it into the `SCREENS` map in `App.tsx`, and (if it makes new API calls) add a fetcher to `lib/api.ts`.

### Data layer (`app/src/lib/api.ts`)

All HTTP goes through one `get<T>` helper that returns `body.results` from the OpenBB envelope. On non-OK responses it parses `detail` and detects the `credential 'foo_api_key'` substring, throwing `ApiError` with a `needsKey` field — UI components surface this as a "needs API key" hint instead of a generic error. Most fetchers default to `provider: "yfinance"` because that's the only no-key path; treasury rates use `federal_reserve`. Don't change providers without checking that the new one works keyless, or the function will 401 in fresh installs.

### Signals engine (`app/src/lib/signals.ts`)

The `INTEL` scorecard is a set of pure-function rules: each `sigX(...)` returns `{ level: "bull" | "bear" | "neutral" | "na", label, detail }`. `tally(signals)` aggregates them into a verdict (Bullish / Bearish / Mixed / Sparse) using a 40%-net-of-informative threshold. These are **heuristics, not predictions** — when tweaking thresholds, keep them transparent and auditable; don't bury them in a component.

### Styling

Tailwind with a custom `term.*` palette in `app/tailwind.config.js` (amber-on-black Bloomberg aesthetic). Reusable component classes (`.panel`, `.panel-header`, `.grid-data`, `.up`, `.down`, `.num`) are defined in `app/src/index.css`. Use these instead of re-styling. The `@/` import alias maps to `app/src/`.

## Commands

All development commands run from `app/` unless noted. The repo-root scripts (`setup.sh`, `start.sh`, `stop.sh`) manage both processes together.

```bash
# First-time setup (from repo root) — creates .venv, installs OpenBB + npm deps
./setup.sh

# Launch both servers + open browser (idempotent — skips already-running ports)
./start.sh

# Kill both
./stop.sh

# Frontend-only dev (assumes API is already running on :6900)
cd app && npm run dev

# Type-check + production build
cd app && npm run build

# Preview the built bundle
cd app && npm run preview
```

There is **no test suite and no linter configured** — `tsc -b` (run as part of `npm run build`) is the only static check. If you add tests or lint, update this file.

Logs after `./start.sh`: `/tmp/bbterminal-api.log` and `/tmp/bbterminal-ui.log`. OpenAPI explorer: `http://localhost:6900/docs`.

## Things to know before changing code

- **Provider choice matters.** Anything other than `yfinance` (or `federal_reserve` for treasury) likely needs an API key from `~/.openbb_platform/user_settings.json`. The UI handles 401s gracefully via `ApiError.needsKey`; preserve that path.
- **Polling, not streaming.** Quotes refresh via `refetchInterval` on React Query. Don't introduce WebSockets — OpenBB's Yahoo provider doesn't support them.
- **Known dead symbols.** A handful of Yahoo symbols (`^N225`, `^HSI`, `^AXJO`, `^TWII`, some crypto) return empty; UI shows "—". Don't "fix" these by swapping providers without confirming the replacement is keyless.
- **Workspace persistence.** `workspaceStore` is persisted, so changes to `Tab` shape or the `tabId` scheme will collide with old localStorage state. Bump the persist `name` if you change the schema.
- **`OpenBB/` is gitignored** — it's the upstream repo cloned for reference only. Don't add code there expecting it to ship.
