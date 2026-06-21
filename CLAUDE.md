# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A single-file Progressive Web App (PWA) for personal annual income/expense tracking. The UI is in Traditional Chinese (zh-TW) and tracks NT$ (New Taiwan Dollar) finances over a 12-month period (2026/06–2027/05). It is installable on mobile via the Web App Manifest.

## Running the app

There is no build step or package manager. Open `index.html` directly in a browser, or serve it with any static file server:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

After changing `service-worker.js` or any cached asset, bump `CACHE_NAME` in `service-worker.js` (e.g. `income-expense-race-v2`) so browsers pick up the new version.

## Architecture

**Everything lives in `index.html`**: CSS in a `<style>` block in `<head>`, markup in `<body>`, and all application logic in a `<script>` block at the end of `<body>`. There are no external JS files, no framework, and no bundler.

Supporting files:
- `manifest.json` — PWA install metadata
- `service-worker.js` — cache-first offline strategy; caches all assets on install, cleans old caches on activate

### Data layer

All state is persisted in `localStorage` under two keys:

| Key | Shape | Purpose |
|-----|-------|---------|
| `raceEntries` | `Array<{month, type, desc, amount, payMethod}>` | Every income/expense entry |
| `debtPayments` | `{mox: number, sis: boolean[], kris: boolean[]}` | Debt repayment state |

`loadEntries()` / `saveEntries()` and `loadDebtPayments()` / `saveDebtPayments()` are the only persistence functions. Both modules guard against `JSON.parse` errors.

### Rendering model

`render()` is the single re-render function — it reads `entries` and `debtPayments`, recomputes all derived values, and writes to the DOM. It is called after every mutation (`addEntry`, `deleteEntry`, `toggleInstallment`, `resetAllData`). There is no virtual DOM or reactive framework; DOM IDs are the stable interface between HTML and JS.

### Financial constants (top of `<script>`)

All monetary targets are hardcoded constants:

```js
ANNUAL_INCOME_TARGET = 1_800_000  // NT$
ANNUAL_RENT_BUDGET   =   800_000
MONTHLY_LIVING       =    25_000
MONTHLY_SAVINGS      =    10_000
MONTHLY_STOCK        =     5_000
DEBT_MOX_TOTAL       =   125_000  // lump-sum debt
DEBT_SIS_TOTAL       =    60_000  // 12 installments of NT$ 5,000
DEBT_KRIS_TOTAL      =    60_000  // 12 installments of NT$ 5,000
TRACK_START_DATE     = 2026-06-16
TRACK_END_DATE       = 2027-05-31
```

### Entry types

`type` field on each entry maps to one of these string values:

- `income` — income (shown green / olive)
- `living`, `transport`, `social`, `utilities`, `other_expense` — variable expenses
- `rent`, `savings`, `stock`, `debt` — fixed / investment expenses

The `render()` function groups entries by type using `categorySum(type)`.

### Key UI components and their DOM IDs

| Component | Role | Key IDs |
|-----------|------|---------|
| Journey bar (TW→CH) | Visualises income % vs time-elapsed % | `journeyMarker`, `journeyProgLine`, `journeyPaceLine`, `journeyStatus` |
| OODA badge | Status: Observe/Orient/Decide/Act based on pace delta | `oodaObserve`…`oodaAct`, `oodaMsg` |
| Duel bar | Income vs total expense tug-of-war | `duelIncomeSeg`, `duelExpenseSeg`, `duelResult` |
| Income race | Progress toward annual income target | `incomeFill`, `incomePct`, `incomeCurrent` |
| Expense race + mini tracks | Total spend + per-category breakdowns | `expenseFill`, `mtLiving`…`mtOther` |
| Debt lanes | Mox (lump-sum) + Sis/Kris (installment dots) | `debtMoxFill`, `debtSisDots`, `debtKrisDots` |
| Entry form | `addEntry()` on button click | `entryType`, `entryMonth`, `entryDesc`, `entryAmt`, `entryPayMethod` |
| Log list | Reverse-chronological list, delete per entry | `logList` |
