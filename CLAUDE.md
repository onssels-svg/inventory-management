# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System — full-stack demo with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
- **vue-expert**: **MANDATORY** for creating or significantly modifying any `.vue` file
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **Explore**: Use for codebase exploration and pattern searches
- **general-purpose**: Use for complex multi-step tasks

### Skills
- **backend-api-test**: Use when writing or modifying tests in `tests/backend/`

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches — use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Frontend: `http://localhost:3000` | API: `http://localhost:8001`

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI + Pydantic v2 + uvicorn (port 8001)
- **Data**: JSON files in `server/data/` loaded into memory at startup via `server/mock_data.py`
- **Tests**: pytest + FastAPI TestClient (51 tests, runs in ~0.13s)

## Commands

### Backend
```bash
cd server
uv run python main.py          # Start server (http://localhost:8001)
```

### Frontend
```bash
cd client
npm install                    # Required on first run — node_modules not committed
npm run dev                    # Start dev server (http://localhost:3000)
npm run build                  # Production build → client/dist/
```

### Tests
```bash
cd tests
uv run pytest -v                                                      # All 51 tests
uv run pytest backend/test_inventory.py -v                           # Single file
uv run pytest backend/test_inventory.py::TestInventoryEndpoints::test_get_all_inventory -v  # Single test
uv run pytest --cov=../server --cov-report=html                      # With coverage
```

> **Windows**: The `scripts/start.sh` / `scripts/stop.sh` shell scripts are macOS/Linux only. Use the manual commands above.

## Architecture

### Data Flow
Vue filter state (`useFilters` composable) → `client/src/api.js` (axios, URLSearchParams) → FastAPI query params → in-memory list comprehension filtering → Pydantic response model → Vue computed properties

### Filter System
4 global filters — Time Period, Warehouse, Category, Order Status — managed by `client/src/composables/useFilters.js` as module-level refs (singleton, shared across all views). Filters are passed as query params to every API call. **Inventory endpoints do not support `month` filtering** (no time dimension in inventory data).

### Backend Pattern
All data loads from `server/data/*.json` at startup into module-level lists in `server/mock_data.py`. Endpoints in `server/main.py` filter these lists on every request — no persistence, restarts reset all changes.

### Frontend Composables
- `useFilters.js` — global filter state (singleton refs)
- `useAuth.js` — mock auth, always authenticated, hardcoded user with mock tasks
- `useI18n.js` — EN/JP translations via `client/src/locales/`, persisted to localStorage

### Styling
Global styles live in `client/src/App.vue` `<style>` (not scoped). Components use scoped styles for local rules. No CSS framework — custom utility classes.

## API Endpoints
- `GET /api/inventory` — Filters: `warehouse`, `category`
- `GET /api/orders` — Filters: `warehouse`, `category`, `status`, `month` (YYYY-MM or Q1-YYYY)
- `GET /api/dashboard/summary` — All filters
- `GET /api/demand`, `GET /api/backlog` — No filters
- `GET /api/spending/summary|monthly|categories|transactions`

## Common Issues
1. Use unique keys in `v-for` — use `sku`, `month`, etc., never array index
2. Validate dates before `.getMonth()` calls (`new Date(x)` can return `Invalid Date`)
3. Update Pydantic models in `server/main.py` when changing JSON data structure
4. Revenue goals: $800K/month (single month filter), $9.6M YTD (all months)
5. `PurchaseOrderModal` is referenced in `Dashboard.vue` but not yet implemented — causes a Vue warn on load

## Design System
- Colors: Slate/gray (`#0f172a`, `#64748b`, `#e2e8f0`)
- Status badges: green (success) / blue (info) / yellow (warning) / red (danger)
- Charts: Custom SVG + CSS Grid layouts
- No emojis in UI
