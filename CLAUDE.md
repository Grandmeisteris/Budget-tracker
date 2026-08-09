# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, client-side monthly budget tracker web app, written in Lithuanian (`lang="lt"`). The entire application — markup, styles, and logic — lives in one file: `budget-tracker.html`. There is no build step, no package manager, and no server; it's opened directly in a browser (or hosted via GitHub Pages) and persists all data in `localStorage`.

## Development

There is no build/lint/test tooling in this repo. To work on it:

- **Run it**: open `budget-tracker.html` directly in a browser, or serve the directory with any static file server.
- **Verify changes**: since there's no test suite, check syntax and behavior manually, e.g.:
  ```bash
  # Quick syntax check of the inline script
  node -e "
  const fs = require('fs');
  const html = fs.readFileSync('budget-tracker.html', 'utf8');
  const script = html.match(/<script>([\s\S]*)<\/script>/)[1];
  new Function(script);
  console.log('OK');
  "
  ```
  For functional verification, drive the page with Playwright/Chromium (`/opt/pw-browsers/chromium` in this environment) — fill the add-entry modal, switch tabs, and read back `innerText`/`textContent` to confirm totals and rendered rows.
- **Deployment**: the app is deployed via GitHub Pages serving `main` from the repo root; the live file is `budget-tracker.html` directly (no `index.html`).

## Architecture

Everything is in `budget-tracker.html`, structured in three parts:

1. **`<style>`** — a dark-themed, mobile-first CSS design (CSS custom properties in `:root` for colors). Layout is built around a sticky header, a bottom tab-bar nav, and bottom-sheet modals (`.modal-overlay` / `.modal`).
2. **Static shell HTML** — the header, empty containers (`#summaryArea`, `#mainContent`) that get their content replaced by JS, the bottom nav, and the add-entry modal (`#addModal`).
3. **`<script>`** — all application logic, no framework, no modules.

### Data model & persistence

- All transaction data is stored in `localStorage`, **keyed per calendar month**: `storageKey(d)` produces `bt_{year}_{month}` (e.g. `bt_2026_6`). `getData()`/`saveData()` read/write the object `{ expenses: [], income: [] }` for `currentDate`'s month. Switching months (`changeMonth`) just changes `currentDate` and re-renders — there is no cross-month aggregation beyond what `renderReport`'s month-over-month comparison does (see below).
- Each entry (expense or income) has: `id` (timestamp via `Date.now()`), `cat`, `amount`, `desc`, `date` (`YYYY-MM-DD`).
- Categories are hardcoded constants: `EXP_CATS` (expenses) and `INC_CATS` (income). `BUDGET_PLAN` and `INCOME_PLAN` hold the planned monthly amount per category, used by the Biudžetas tab to compute over/under-budget status. These plans are compile-time constants, not currently editable through the UI.

### Rendering model

There's no virtual DOM or diffing — every render call does a full `innerHTML` replacement of its target container. The single entry point is `render()`, which always updates `#monthLabel` and `#summaryArea` (via `renderSummary()`), then dispatches to one of the per-tab render functions based on `currentPage`:

- `renderExpenses()` / `renderIncome()` — transaction lists for the current month, newest first.
- `renderBudget()` — per-category planned vs. actual spend, computed by summing `data.expenses` grouped by `cat`.
- `renderReport()` — month summary, income-by-source, top expense categories, and CSV export. This is also where `generateInsights(data)` (see below) is invoked and rendered as a list of Lithuanian-language observations.

Any state change (add/delete entry, change month, switch tab) calls `render()` to redraw. There's no partial update — always re-derive from `localStorage` and rebuild HTML strings via template literals.

### Monthly analysis (`generateInsights`)

`renderReport()` calls `generateInsights(data)`, which compares the current month against the previous month (via `getMonthData(prevDate)`, a read-only helper distinct from `getData()`/`currentDate`) to generate a list of natural-language observations in Lithuanian: overall spend trend, the category with the biggest increase, budget overruns (against `BUDGET_PLAN`), dominant category share, largest single transaction, average transaction size, and a savings-rate verdict. Each rule pushes a string into an `insights` array only when its threshold condition is met (e.g. trend deltas under 5% are considered "no significant change" and get a neutral message instead). When extending this, follow the existing pattern: one self-contained `if`/threshold block per observation, appended to `insights`.

### Modals

The add-entry modal (`#addModal`) is shared between expenses and income; `openModal(type)` swaps its title and category `<select>` options based on `modalType`, and `saveEntry()` pushes into `data.expenses` or `data.income` accordingly. Modals are shown/hidden via the `.open` class on `.modal-overlay`, and clicking the overlay background (not the sheet itself) closes them (see the `addEventListener('click', ...)` guard at the bottom of the script).

### Styling conventions

Colors are centralized as CSS custom properties (`--bg`, `--surface`, `--green`, `--red`, etc.) in `:root`. Per-category colors for chart dots/tags are a separate hardcoded map in `getColor(cat)` in JS — when adding a category, add its color there too or it falls back to gray (`#888`).
