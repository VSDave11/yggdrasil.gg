# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

```bash
node index.js
```

Runs on `http://localhost:3000`. No build step. The app deploys to Render.com — the `GOOGLE_CREDENTIALS` env var is used in production instead of `credentials.json`.

## Architecture

Everything lives in a single file: **`index.js`** (~2370 lines). It is a Node.js/Express server that serves a single-page HTML dashboard. There is no frontend framework — all HTML, CSS, and client JS are generated as template strings inside route handlers and sent via `res.send()`.

**`public/`** — Static assets (login page `index.html`, `style.css`, images). The login page is served at `/`, the dashboard at `/dashboard`.

### Data layer

Google Sheets is the database. The `google-spreadsheet` + `google-auth-library` (JWT) packages read/write sheets.

- **`Schedule - <Month Year>`** sheets (e.g. `Schedule - March 2026`) — shift planner grid. Columns map to products via `productMapping` (hardcoded column offsets, e.g. Valhalla Cup A starts at col 2). Dates are in column 0 as Google Sheets serial numbers or Czech-format strings (`6.4.2026`).
- **`ManualShifts`** sheet — manually added/edited shifts, with columns `Date, Name, Trading, Product, Start, End, Note`.
- **`AuditLog`** sheet — event log with columns `Timestamp, Jmeno, Event, Detail`. Events: `LOGIN`, `ADD_SHIFT|name|product|date`, `EDIT_SHIFT|name|product|date`.
- **`uzivatele`** sheet — user accounts (email, password, role, jmeno, location).

**Cache:** `_shiftsCache` + `_shiftsCacheTime` — 2-minute in-memory cache for all shifts. Invalidated by any write operation. The `/dashboard?sync=1` query forces a refresh.

### Key server-side functions (module-level)

- `convertCzechDate(val)` — normalises any date format (serial number, `D.M.YYYY`, ISO) to `YYYY-MM-DD`.
- `timeToPercent(timeStr)` — converts `HH:MM` → 0–100% of 24 hours.
- `calculateDuration(start, end)` — returns shift duration in hours, handles overnight.
- `getProductColor(tradingName, productName)` — returns per-product color from `productColors`, falls back to category color from `tradingHierarchy`.

### Dashboard rendering (`GET /dashboard`)

The entire page is built server-side in one large template. Key data structures declared inside the route:

- `peopleHierarchy` — groups of people with display color and weekly target hours.
- `tradingHierarchy` — trading categories with sub-products (e.g. FIFA → Valhalla Cup A/B/C).
- `productColors` — per-product hex colors (defined at module level, line ~138).
- `personColors` — per-person hex colors (~58 entries, defined at module level, line ~78).

**Views** — controlled by `?view=` query param:
- `timeline` (default) — horizontal 7-day scrollable grid. Each person/product is a row (`user-row` / `product-row`). Shifts are `position:absolute` pills with `left`/`width` as percentages of the 960px-per-day grid (960px = 24h, 40px = 1h).
- `week` — vertical calendar grid, 7 columns × 24 rows (40px/hour).
- `list` — flat chronological list grouped by day.
- `agenda` — Google Calendar-style with date sidebar.

**Overnight shifts** — detected when `startPct > endPct && endPct > 0`. Rendered as two pills: Pill 1 (start → midnight), Pill 2 (midnight → end, next day). A pre-pass loop handles Sunday→Monday continuation for shifts that started the previous week.

**Sidebar filter** — `applyAllFilters()` in client JS shows/hides rows by toggling `hidden-row` CSS class. Pills in week/list/agenda have both `user-row` and `product-row` classes; the filter uses OR logic (show if person OR product matches).

### API endpoints

| Route | Purpose |
|-------|---------|
| `POST /login` | Auth against `uzivatele` sheet |
| `GET /export-csv` | Download all shifts as CSV (Admin/TL only), supports `?name=` filter |
| `POST /add-shift` | Adds row to `ManualShifts` |
| `POST /update-shift` | Edits row in `ManualShifts` by row index |
| `POST /delete-shift` | Deletes row from `ManualShifts` |
| `POST /exchange-shift` | Swaps two people's shifts |
| `POST /delete-month` | Clears all ManualShifts for a given month |
| `GET /api/shift-history` | Returns created/edited audit entries for a specific shift |
| `GET /api/schedule-sheets` | Returns sorted list of `Schedule - *` sheet names |

### Client JS (embedded in dashboard template)

Serialized at render time: `pColors` (person colors), `pRoles` (person→group), `tColors` (trading category colors), `pColorsProduct` (per-product colors).

Key client functions: `openViewModal()`, `applyAllFilters()`, `toggleSelect()`, `toggleProduct()`, `saveSelection()` (persists sidebar state to `localStorage` keys `ygg_sel_names` / `ygg_sel_prods`).
