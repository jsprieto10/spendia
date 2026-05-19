# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Spendia is a single-file personal finance dashboard that reads expense data from a Google Sheet and renders analytics entirely in the browser. There is no build step, no package manager, and no framework — everything lives in `dashboard.html` (inline CSS + vanilla JS + SVG charts).

## Development

Open `dashboard.html` directly in a browser. No server required for local testing, but Google OAuth will only work from an origin registered in Google Cloud Console (or `localhost` if configured).

There are no lint, test, or build commands.

## Deployment

GitHub Actions (`.github/workflows/deploy.yml`) deploys to GitHub Pages on every push to `main`. The workflow injects credentials via `sed` by replacing these placeholders in `dashboard.html` before upload:

- `__CLIENT_ID__` → `secrets.GOOGLE_CLIENT_ID`
- `__API_KEY__` → `secrets.GOOGLE_API_KEY`
- `__SHEET_ID__` → `secrets.SHEET_ID`
- `__SHEET_TAB__` → `vars.SHEET_TAB`
- `__SCOPES__` → `vars.SCOPES`

For local development, actual credential values are stored in `ignore.js` (gitignored) and must be manually placed in the `// Start config … // End Config` block inside `dashboard.html`.

## Architecture

### Data flow

1. **Auth** — `initAuth()` sets up the Google Identity Services token client. On login, the OAuth access token is stored in `sessionStorage` so it survives page refreshes.
2. **Fetch** — `loadData()` calls the Sheets API v4 (`spreadsheets.values.get`) on the `Acumulado` tab, columns A–H, using the bearer token.
3. **Parse** — `parseRows()` normalises raw 2D array values into typed `rawRows` objects `{descripcion, valor, categoria, fuente, dia, mes, anio, monthKey, dateObj}`. European decimal commas are handled here.
4. **Render** — `renderAll()` drives four independent view renderers after data loads.

### Google Sheet expected schema

Headers (case-insensitive, matched by prefix): `Descripción`, `Valor`, `Categoría`, `Fuente`, `Día`, `Mes`, `Año`. A `Fecha` (DD/MM/YYYY) column is used as fallback if the individual date columns are missing.

### View panels

| Tab | Panel ID | Key renderer |
|-----|----------|-------------|
| Overview | `panel-overview` | `renderOverview()`, `renderBars()`, `renderCategoryDistribution()` |
| Mes a mes | `panel-monthly` | `renderMonthly()`, `renderDiffChart()` |
| Categorías | `panel-categories` | `renderCategoryDetail()`, `renderCategoryTrend()` |
| Transacciones | `panel-transactions` | `renderTx()` |

Charts are hand-drawn SVGs with fixed `viewBox` dimensions; `preserveAspectRatio="none"` is used for responsive scaling. Tooltips are absolutely-positioned `<div>` overlays tied to `mousemove` events on SVG elements.

### Category colour assignment

`catColors` is built once in `parseRows()` by cycling through the 15-colour `PALETTE` array in alphabetical category order, giving stable colour-to-category mapping across all views.
