# CLAUDE.md — Trader Theorem

AI assistant guide for working on the Trader Theorem codebase.

---

## Project Overview

**Trader Theorem** is a client-side trading journal web application built as a single React component. It is designed to run in a browser with no server, no build step, and no database — all persistence is handled via `localStorage` and browser file APIs.

The project is a personal/learning project and is acknowledged as "vibe coded." The next iteration is called **Nexus Terminal**.

---

## Repository Structure

```
Trader-Theorem/
├── README.md              # Project overview and roadmap
├── CLAUDE.md              # This file
└── Trading Journal VX     # The entire application (1351-line React component)
```

There is **no `package.json`**, **no build system**, **no test suite**, and **no CI/CD**. The single source file contains the complete application.

---

## The Main File: `Trading Journal VX`

This is a React functional component exported as default (`TradingJournal`). It is 1,351 lines of JavaScript and contains:

- All UI rendering (JSX)
- All state management (React hooks)
- All business logic
- All styling (inline via the `S` constants object)
- All CSV parsing
- All localStorage persistence

### External Dependencies (imported, not installed locally)

| Library | Purpose |
|---|---|
| `react` | UI framework (hooks: `useState`, `useMemo`, `useEffect`, `useRef`) |
| `papaparse` | CSV parsing (`Papa.parse`) |
| `recharts` | Charts (`LineChart`, `AreaChart`, `BarChart`, and related components) |

These must be provided by the host environment (e.g., CDN links or a parent React project).

---

## Architecture & Key Patterns

### State Management

The component manages 20+ `useState` hooks. Key state:

| State variable | Type | Purpose |
|---|---|---|
| `rawTrades` | Array | Individual trade executions parsed from CSV |
| `allTrades` | (memoized) | Consolidated trades (entries matched to exits) |
| `trades` | (memoized) | Date-filtered view of `allTrades` |
| `dateRisks` | Object | Risk amount per trading day (`sortKey -> number`) |
| `tradeTags` | Object | Tags per trade (`tradeId -> string[]`) |
| `savedTags` | Array | All unique tags for the dropdown |
| `activeTab` | String | Current tab: `'calendar'`, `'overview'`, `'r'`, `'filter'`, `'trades'` |
| `dateMode` | String | Date filter: `'all'`, `'single'`, `'range'` |
| `importedFiles` | Set | Filenames already imported (prevents re-import) |
| `isWatching` | Boolean | Whether folder watch is active |

### Memoized Computations (`useMemo`)

Heavy computation is memoized to avoid re-renders:

- `allTrades` — groups executions into matched entry/exit trade pairs
- `trades` — applies date range filter to `allTrades`
- `symbolSummary` — per-symbol P&L aggregation
- `tradesByDate` — groups trades by `sortKey` for calendar view
- `calGrid` — 6-week calendar layout array
- `stats` — overall performance statistics (win rate, avg win/loss, etc.)
- `perfByDow` — performance by day of week (Mon–Fri)
- `perfByHour` — performance by entry hour
- `rStats` — R-multiple risk-adjusted statistics

### Trade Data Model

**Execution** (raw, from CSV):
```js
{
  date: Date,
  dateStr: string,      // e.g. "Mar 1, 2026"
  sortKey: string,      // "YYYY-MM-DD"
  symbol: string,
  side: string,         // "SS" | "B" | "MARGIN" | "S"
  qty: number,
  price: number,
  direction: 'LONG' | 'SHORT'
}
```

**Consolidated Trade** (after `allTrades` memoization):
```js
{
  id: string,           // "SYMBOL-YYYY-MM-DD"
  symbol: string,
  direction: 'LONG' | 'SHORT',
  sortKey: string,
  dateStr: string,
  date: Date,
  pnl: number,
  totalQty: number,
  executions: Execution[],
  entryPrice: number,
  exitPrice: number,
  avgEntry: number,
  avgExit: number,
  entryValue: number,
  exitValue: number,
  entryTime: string,
  exitTime: string
}
```

### P&L Calculation

- **Short trade**: `pnl = (avgEntry - avgExit) * totalQty`
- **Long trade**: `pnl = (avgExit - avgEntry) * totalQty`

### CSV Format Expected

Files must be named with a date pattern: `M D YY` (e.g., `3 1 26.csv` → March 1, 2026).

Expected CSV columns:
```
Time | Symbol | Qty | Price | Side
```

`Side` values:
- `SS` = Short sell entry
- `B` = Short cover exit (buy to cover)
- `MARGIN` = Long entry (buy on margin)
- `S` = Long exit (sell)

### Persistent Storage

All data is stored in `localStorage` under the key `'journal-data'` as JSON:
```js
{
  rawTrades,
  dateRisks,
  tradeTags,
  savedTags,
  uploadedDates,
  importedFiles  // serialized from Set to Array
}
```

Saves are **debounced 500ms** after any state change. Load happens once on mount.

### Folder Watch

Uses the browser **File System Access API** (`window.showDirectoryPicker`). Polls every **5 seconds** for new `.csv` files not yet in `importedFiles`.

---

## Styling Conventions

All styles are defined in a single top-level `S` object (CSS-in-JS):

```js
const S = {
  root, wrap, row, card,
  btn, btnActive, btnDanger,
  input, select,
  table, th, thR, td, tdR,
  green, red, dim, mid,
  label, big, hidden
};
```

**Color palette:**
- Background: `#111`
- Card: `#1a1a1a`
- Border: `#2a2a2a` / `#333`
- Text: `#e5e5e5`
- Green (profit/active): `#22c55e`
- Red (loss/danger): `#ef4444`
- Dim text: `#666` / `#777`
- Mid text: `#888` / `#999`

**Helper formatting functions:**
- `fmt(v)` → `"$123.45"` or `"-$123.45"`
- `fmtR(v)` → `"+1.50R"` or `"-0.75R"`
- `pnlColor(v)` → returns `S.green`, `S.red`, or `S.mid`

---

## Application Tabs

| Tab key | Label | Contents |
|---|---|---|
| `'calendar'` | Calendar | Monthly grid + DOW/hourly bar charts |
| `'overview'` | Overview | Equity curve (line + area), W/L area chart |
| `'r'` | R-Multiple | Risk-adjusted equity curve, R-stats |
| `'filter'` | Filter | Tag-based trade filtering and analysis |
| `'trades'` | Trades | Full trade list with tagging/selection UI |

---

## Development Guidelines

### No Build Step

This is a raw `.jsx`-style file with no transpilation. To use it:
- Drop it into a React project with the 3 dependencies available, OR
- Serve it with a CDN-based React setup (e.g., via `<script>` tags with Babel standalone)

### No Tests

There are currently no tests. If adding tests, they would need a test runner (e.g., Vitest or Jest) and React Testing Library to be introduced as a new dependency.

### Modifying the File

Because the entire app is one file:
- Keep the `S` styles object at the top for all new styles
- All new state goes in the `TradingJournal` component body alongside existing `useState` calls
- New memoized data goes in the `useMemo` section
- New UI sections go at the bottom of the JSX return

### Code Style

The existing code uses:
- Short/abbreviated variable names (`p`, `w`, `l`, `d`, `t`, `s`, etc.) throughout
- Terse arrow functions, often single-line
- No TypeScript — plain JavaScript only
- No comments except section headers (`// ===== SECTION =====`)
- Inline styles everywhere (no CSS classes, no CSS files)

When adding code, match the existing compact style.

### localStorage Key

Do not rename or restructure the `'journal-data'` key without migrating existing data. Users have real trade data stored under this key.

### File Naming

Avoid introducing a `.jsx` extension to `Trading Journal VX` without verifying the host environment supports it. The space in the filename is intentional.

---

## Git Workflow

- Main branch: `master`
- Feature work: `claude/*` branches
- Remote: configured locally (proxied)
- No CI/CD is configured

Commit message style: plain English, no strict convention observed in history.

---

## Roadmap (from README)

Planned features not yet implemented:
- Backend server with OAuth 2.0
- Trade notes (600 chars + 2 PDFs per trade)
- Google Sheets / Polygon / Charles Schwab API sync
- Backtesting tab
- Watchlists with presets
- EDGAR scraper agent for company DD
- UI/UX consistency pass across all tabs

---

## Quick Reference

| Thing | Where |
|---|---|
| Main component | `Trading Journal VX` (line 33) |
| Styles | `Trading Journal VX` (lines 5–27) |
| localStorage load | `Trading Journal VX` (line 66) |
| localStorage save | `Trading Journal VX` (line 91) |
| CSV parsing | `processFile()` function |
| Trade consolidation | `allTrades` useMemo |
| Folder watch | `pickWatchFolder()` / `pollFolder()` |
| Date filter logic | `trades` useMemo |
| Stats computation | `stats` useMemo |
