# Folio — Application Handover Memo

**Date:** 15 April 2026
**Author:** Tristan Guilbot (via Claude)
**Purpose:** Complete technical handover for continued development in Cowork

---

## 1. What is Folio?

Folio is a **personal finance manager** built as a single HTML file (~106KB, ~1280 lines). It tracks:

- **Stock portfolio** — trades, live prices, P&L, performance metrics
- **Savings accounts** — Livret A, LDDS, PEL, Compte à Terme, Fonds Euro, etc.
- **Real estate** — properties, SCPI, loans, rental income, cashflow
- **Global overview** — net worth, asset allocation, passive income

The app is designed for a **French investor** — all values are displayed in EUR (€) with French locale formatting, account types include PEA/CTO/Assurance Vie/PER, and the search covers Euronext Paris primarily.

### Key characteristics

- **Single HTML file** — no build step, no framework, no bundler. Everything (HTML + CSS + JS) lives in one `index.html`
- **Hosted on GitHub Pages** — repo: `9rftnvtbk7-cloud/folio`, URL: `https://9rftnvtbk7-cloud.github.io/folio/`
- **Firebase backend** — Google login + Firestore real-time sync across devices
- **Dual price sources** — Finnhub API (US stocks) + Yahoo Finance via Cloudflare Worker proxy (European stocks + indices)

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────┐
│                  Browser                      │
│  ┌─────────────────────────────────────────┐  │
│  │           index.html (single file)      │  │
│  │  ┌──────────┐  ┌────────────────────┐   │  │
│  │  │  CSS     │  │   JavaScript       │   │  │
│  │  │  (~170   │  │   (~1100 lines)    │   │  │
│  │  │  lines)  │  │   65 functions     │   │  │
│  │  └──────────┘  └────────────────────┘   │  │
│  └─────────────────────────────────────────┘  │
└──────────────┬─────────────┬─────────────────┘
               │             │
    ┌──────────▼──────┐ ┌────▼─────────────────┐
    │   Firebase      │ │   Price APIs          │
    │  (folio-95fd9)  │ │                       │
    │  - Auth (Google)│ │  Finnhub (US stocks)  │
    │  - Firestore    │ │  Yahoo Finance        │
    │    (7 collections│ │   via CF Worker      │
    │     per user)   │ │  ExchangeRate API (FX)│
    └─────────────────┘ └───────────────────────┘
```

### File structure

There is only ONE file: `index.html`. It contains:

1. **`<head>`** — meta tags, Google Fonts (DM Sans, JetBrains Mono)
2. **`<style>`** — all CSS (~170 lines), dark theme, CSS custom properties
3. **`<body>`** — HTML structure for all 6 tabs + auth overlay
4. **`<script type="module">`** — Firebase SDK (auth, Firestore), auth logic, sync
5. **`<script>`** — main app logic (`"use strict"`, ~1100 lines, 65 functions)

---

## 3. External Services & Credentials

### 3.1 Firebase (Authentication + Database)

| Setting | Value |
|---------|-------|
| Project ID | `folio-95fd9` |
| Auth domain | `folio-95fd9.firebaseapp.com` |
| API key | `AIzaSyBEy06ky8M4FyqP89WoaxWAkcIckSM02B0` |
| App ID | `1:638037964174:web:946b770ccb2f210bd020df` |
| Storage bucket | `folio-95fd9.firebasestorage.app` |
| Region | `eur3` (Europe) |

**Authentication:** Google Sign-In only. The authorized domain `9rftnvtbk7-cloud.github.io` must be added in Firebase Console → Authentication → Settings → Authorized domains.

**Firestore structure:**

```
users/
  {uid}/
    data/
      trades      → { data: [...] }
      savings     → { data: [...] }
      realestate  → { data: [...] }
      manual      → { data: {...} }   // manual stock prices
      names       → { data: {...} }   // ticker → company name
      fx          → { data: {...} }   // FX rates
      meta        → { data: {...} }   // ticker → metadata (sector, etc.)
```

**Security rules** should restrict read/write to authenticated user only:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

### 3.2 Finnhub API (Stock data — primarily US)

| Setting | Value |
|---------|-------|
| API key | `d6gkbt1r01qifrh0fhsgd6gkbt1r01qifrh0fht0` |
| Free tier | 60 calls/min |
| Endpoints used | `search` (symbol lookup by name/ticker/ISIN), `quote` (live price), `stock/profile2` (company info) |

**Limitation:** Finnhub free tier does NOT reliably return quotes for European stocks (.PA, .AS, .DE, etc.). It works well for US stocks and for search/lookup of any market.

### 3.3 Cloudflare Worker (Yahoo Finance CORS proxy)

| Setting | Value |
|---------|-------|
| Worker URL | `https://folio-proxy.t4ng9shfbw.workers.dev/` |
| Usage | Proxies requests to `query1.finance.yahoo.com` adding CORS headers |
| Free tier | 100,000 requests/day |

**How it works:** The browser cannot directly call Yahoo Finance due to CORS. The Worker receives `?url=<yahoo_url>`, fetches from Yahoo server-side, and returns the response with `Access-Control-Allow-Origin: *`.

**Yahoo Finance endpoint used:** `v8/finance/chart/{symbol}?interval=1d&range=5d` — returns OHLC data, current price, previous close.

### 3.4 Exchange Rate API (FX rates)

| Setting | Value |
|---------|-------|
| URL | `https://api.exchangerate-api.com/v4/latest/EUR` |
| Auth | None (free, no key) |
| Usage | Fetched on each price refresh to convert USD/GBP/CHF/etc. to EUR |

---

## 4. The 6 Tabs

### 4.1 Portfolio Tab

The main view. Shows:

- **Market indices bar** — horizontal scrollable chips showing CAC 40, Euronext 100, Euro Stoxx 50, S&P 500, Nasdaq, Dow Jones, FTSE 100, DAX with live values and day change %
- **Summary cards** — Market Value, Total P&L (€ + %), Day Change, Total Fees, Positions count
- **Filter row** — dropdowns for Sector, Industry, Broker, Account
- **Sortable table** with columns: Ticker (+ name), Shares, Avg Cost €, Break Even €, Price €, Day Chg €, Day %, MTD %, YTD %, Mkt Value €, P&L €, Fees €
- **Manual stocks** show a "MANUAL" badge and an "Update Price" button

**Position aggregation logic** (`enrichPositions` function):
- Groups trades by ticker
- Calculates average cost (weighted by shares), total fees, total shares
- Converts all values to EUR using FX rates
- Break even = (total cost in EUR + total fees) / total shares
- P&L = market value − total cost − fees

### 4.2 Dashboard Tab

10 visualization panels using custom SVG donut and bar charts:

1. By Sector
2. By Industry
3. By Asset Type (Stock, ETF, OPCVM, etc.)
4. By Geography
5. By Risk Level
6. By Exchange
7. By Currency
8. By Broker
9. By Account Type
10. Top 10 Holdings

Each chart uses a 16-color palette defined in `PALETTE`. The `donut()` function generates SVG donut charts; `barCard()` generates horizontal bar charts.

### 4.3 Ledger Tab

Trade journal showing all BUY/SELL transactions in reverse chronological order.

**Trade form** has two modes:

- **Yahoo Finance mode** — search by company name, ticker, or ISIN → Finnhub `search` endpoint → select result → auto-fills ticker, validates, fetches price. Metadata fields (sector, industry, type, geo, risk, exchange) are shown for editing.
- **Manual / Unlisted mode** — enter ticker + name + optional current price + all metadata fields manually. For stocks not on any exchange (OPCVM, PE funds, SCPI, etc.)

**Trade data structure:**

```javascript
{
  id: "t-1709123456789-1",    // unique ID
  ticker: "BN.PA",
  type: "BUY",                // or "SELL"
  shares: 100,
  price: 65.50,               // in native currency
  currency: "EUR",
  fees: 4.95,                 // in EUR
  broker: "Boursorama",
  account: "PEA",             // CTO, PEA, PEA-PME, AV, PER, Other
  date: "2026-01-15",
  notes: "Q1 reinforcement",
  manual: false,              // true if entered in manual mode
  name: "Danone SA"
}
```

**Edit functionality:** Each trade row has an edit button (pencil icon). Clicking it opens the form pre-filled with all trade data including metadata, with the submit button reading "Update Trade".

**Sell validation:** The form prevents selling more shares than currently held for a given ticker.

### 4.4 Savings Tab

CRUD for savings/cash accounts.

**Fields:** Account Name, Account Type (Compte Courant, Livret A, LDDS, LEP, PEL, CEL, Livret Jeune, Compte à Terme, Fonds Euro AV, Other), Bank (with autocomplete), Balance (€), Interest Rate (%/yr), Notes.

**Display:** Cards showing account details + computed Annual Interest (balance × rate / 100).

### 4.5 Real Estate Tab

CRUD for properties, SCPI shares, etc.

**Fields (12):** Property Name, Property Type, Purchase Price (€), Current Value (€), Remaining Loan (€), Loan Rate (%/yr), Monthly Payment (€), Rented? (yes/no), Monthly Rent (€), Annual Charges (€), Annual Tax (€), Notes.

**Computed values displayed:**
- P&L: current value − purchase price
- Equity: current value − remaining loan
- Gross yield: (annual rent / value) × 100
- Net yield: ((annual rent − charges − tax) / value) × 100
- Annual cashflow: rent×12 − charges − tax − monthly payment×12

### 4.6 Global Overview Tab

Read-only dashboard aggregating all asset classes:

- **Net Worth banner** — stocks market value + savings total + real estate equity
- **Summary cards** — Total Assets, Total Liabilities, Net Worth, Annual Passive Income
- **Asset Allocation** — two donut charts (by equity / by gross value)
- **Stocks section** — 4 cards (Market Value, Total Cost, P&L, Positions)
- **Savings section** — 4 cards (Total Balance, Accounts, Annual Interest, Avg Rate)
- **Real Estate section** — 6 cards (Gross Value, Loans, Equity, Properties, Annual Rent, Annual Cashflow)

---

## 5. Price Fetching Flow

### 5.1 On ticker search/selection (trade form)

```
User types in "Search" field
  → debounce 350ms
  → Finnhub /search?q={query}     (accepts name, ticker, or ISIN)
  → dropdown shows results with exchange names
  → user clicks result
  → selectSearchResult():
      1. Accept ticker immediately (tvValid=true)
      2. Show info card with known metadata
      3. Auto-set currency from metadata
      4. Background: try Finnhub quote → if fails → try yahooQuote()
      5. If price found, display it in info card
```

### 5.2 On portfolio refresh (`refreshPrices`)

```
Pass 1: Finnhub (all non-manual tickers)
  → fhFetch("quote", {symbol: ticker}) for each
  → 120ms delay between calls (rate limiting)
  → successful: store in prices[ticker]
  → failed: add to failed[] array

Pass 2: Yahoo Finance fallback (failed tickers only)
  → yahooQuote(ticker) for each
  → tries CORS_PROXY first, then corsproxy.io, then allorigins, then codetabs
  → 200ms delay between calls

In parallel: fetchFxRates() from exchangerate-api.com
```

### 5.3 Yahoo Finance proxy (`yahooQuote` function)

```javascript
yahooQuote(symbol) → tries multiple CORS proxies in order:
  1. CORS_PROXY (Cloudflare Worker) — most reliable
  2. corsproxy.io
  3. allorigins
  4. codetabs

Each proxy wraps: https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval=1d&range=5d

Response parsing:
  - price = meta.regularMarketPrice || last closing price
  - prevClose = meta.chartPreviousClose
  - change = price - prevClose
  - changePct = (change / prevClose) * 100
```

### 5.4 Market indices (`fetchIndices`)

Fetches 8 indices via `yahooQuote()`: ^FCHI (CAC 40), ^N100 (Euronext 100), ^STOXX50E (Euro Stoxx 50), ^GSPC (S&P 500), ^IXIC (Nasdaq), ^DJI (Dow Jones), ^FTSE (FTSE 100), ^GDAXI (DAX).

---

## 6. Data Persistence

### 6.1 localStorage keys

| Key | Content |
|-----|---------|
| `folio-trades` | Array of trade objects |
| `folio-savings` | Array of savings account objects |
| `folio-realestate` | Array of real estate property objects |
| `folio-manual` | `{TICKER: {price, currency}}` for manually-priced stocks |
| `folio-names` | `{TICKER: "Company Name"}` map |
| `folio-fx` | `{USD: 0.92, GBP: 1.17, ...}` FX rates EUR-based |
| `folio-meta` | `{TICKER: {sector, industry, type, geo, risk, exchange, currency}}` |

### 6.2 Firestore sync

- On login: `loadFromFirestore(uid)` reads all 7 collections into localStorage
- On any save: `window.__folioSaveToCloud(key, data)` writes to Firestore
- Real-time: `startListeners(uid)` sets up `onSnapshot` listeners on all 7 collections for cross-device sync
- On logout: listeners are unsubscribed

### 6.3 Hardcoded ticker metadata (`TM` object)

~47 tickers have pre-defined metadata (sector, industry, currency, exchange, geography, risk, type). This serves as fallback when Finnhub doesn't return metadata. Examples: BN.PA, MC.PA, AAPL, MSFT, ASML.AS, etc.

The `getMeta(ticker)` function checks: tickerMeta (user-saved) → TM (hardcoded) → defaults.

---

## 7. Key Functions Reference

### Core rendering

| Function | Purpose |
|----------|---------|
| `render()` | Master render — calls enrichPositions, builds portfolio table, updates summary cards, triggers sub-renders |
| `enrichPositions()` | Aggregates trades into positions with P&L, applies FX conversion, sorting, filtering |
| `renderDash(positions)` | Builds all 10 dashboard charts |
| `renderSavings()` | Renders savings tab cards |
| `renderRe()` | Renders real estate tab cards |
| `renderGlobal()` | Renders global overview tab |

### Price & data fetching

| Function | Purpose |
|----------|---------|
| `refreshPrices()` | Two-pass price fetch: Finnhub then Yahoo fallback |
| `yahooQuote(symbol)` | Fetches price from Yahoo Finance via CORS proxy chain |
| `fhFetch(endpoint, params)` | Finnhub API wrapper |
| `fetchFxRates()` | Fetches EUR-based FX rates |
| `fetchIndices()` | Fetches 8 market indices via yahooQuote |

### Search & validation

| Function | Purpose |
|----------|---------|
| `doSearch(q)` | Debounced search via Finnhub (name/ticker/ISIN) |
| `selectSearchResult(symbol, name)` | Handles search result click — fills form, fetches price in background |
| `validateTicker(tk)` | Multi-fallback validation: profile2+quote → search+quote → TM lookup |
| `showTvResult(r)` | Renders validation result card (green check or red X) |

### Trade management

| Function | Purpose |
|----------|---------|
| `submitTrade()` | Creates or updates a trade (handles editingTradeId) |
| `editTrade(id)` | Opens form pre-filled with existing trade data for editing |
| `hideForm()` | Closes and resets trade form |
| `applyMode()` | Toggles between Yahoo Finance and Manual mode |

### Persistence

| Function | Purpose |
|----------|---------|
| `saveTrades()` / `loadTrades()` | localStorage + Firestore sync for trades |
| `saveMeta()` / `loadMeta()` | Ticker metadata persistence |
| `saveManual()` / `loadManual()` | Manual stock prices |
| `saveNames()` / `loadNames()` | Stock name map |

### Utilities

| Function | Purpose |
|----------|---------|
| `toEur(amount, currency)` | Converts any amount to EUR using fxRates |
| `eurFmt(val)` | Formats number as "1 234,56 €" |
| `getMeta(ticker)` | Returns metadata with fallback chain |
| `getName(ticker)` | Returns company name from stockNames map |
| `isManualTicker(ticker)` | Checks if ticker uses manual pricing |
| `getSharesHeld(ticker)` | Calculates net shares held (buys − sells) |

---

## 8. CSS / Design System

### Theme (dark mode only)

| Variable | Value | Purpose |
|----------|-------|---------|
| `--bg` | `#0f1117` | Page background |
| `--surface` | `#161922` | Card backgrounds |
| `--card` | `#1c1f2e` | Elevated surfaces |
| `--border` | `#2a2d3a` | Borders |
| `--text` | `#a0a4b0` | Body text |
| `--bright` | `#e8eaf0` | Emphasized text |
| `--dim` | `#5a5e6a` | Muted text |
| `--accent` | `#6ee7b7` | Mint green (primary accent) |
| `--green` | `#6ee7b7` | Positive values |
| `--red` | `#f87171` | Negative values |

### Typography

- **Primary font:** DM Sans (headings, body)
- **Monospace font:** JetBrains Mono (prices, numbers, tickers)

### Layout

- Max width: 1440px, centered
- Responsive grid for forms: `repeat(auto-fit, minmax(160px, 1fr))`
- Dashboard grid: `repeat(auto-fit, minmax(340px, 1fr))`
- Indices bar: horizontal scroll, no scrollbar

---

## 9. Development History

| Session | Date | Key additions |
|---------|------|---------------|
| 1 | Feb 24 | Initial build: React portfolio table, trade ledger, live prices via Anthropic API |
| 2 | Feb 25 | Converted to single HTML file, added Dashboard tab with 10 chart panels |
| 3 | Feb 26 | EUR base currency, FX conversion, broker/account tracking, transaction fees |
| 4 | Feb 27 AM | Break-even price, historical performance (day/MTD/YTD %), expanded columns |
| 5 | Feb 27 PM | Manual/unlisted stock support, stock names, Savings tab, Real Estate tab, Global Overview |
| 6 | Mar 3 | Firebase auth + Firestore sync, Finnhub API migration, manual metadata fields, GitHub Pages deploy |
| 7 | Mar 3 (cont.) | Company search by name/ISIN, Yahoo Finance fallback via CF Worker, market indices bar, trade editing |

---

## 10. Known Issues & Limitations

1. **European stock prices depend on Cloudflare Worker** — if the Worker goes down, only US stocks get prices. The fallback public proxies (corsproxy.io, etc.) are unreliable.

2. **No historical data** — MTD % and YTD % columns exist but Finnhub free tier doesn't provide month-ago/year-ago prices. These always show 0%.

3. **FX rates are snapshot** — fetched once per refresh, not continuously updated. For precise EUR conversion, rates may lag.

4. **Single-file constraint** — at ~1280 lines, the file is getting large. A refactor into separate files (with a build step) may be needed for further expansion.

5. **No dividend tracking** — the app tracks capital gains only, not dividend income.

6. **No import/export** — no CSV import for bulk trade entry, no data export.

7. **Search dropdown z-index** — on mobile, the dropdown may partially overlap with other form fields. The `form-card` has `overflow:visible` and `z-index:50` to mitigate this.

8. **Finnhub metadata gaps** — for many .PA stocks, Finnhub returns incomplete profile data (no sector, no industry). Users must edit trades to add metadata manually.

---

## 11. Suggested Next Steps

- **CSV import/export** — bulk trade import from broker exports (Boursorama, Degiro, etc.)
- **Dividend tracking** — add dividend transaction type, display yield
- **Historical performance chart** — portfolio value over time using trade history
- **Alerts / targets** — price alerts for positions
- **Better mobile responsive design** — the table is wide, needs horizontal scroll or card view on mobile
- **Split into modules** — separate CSS, JS modules with a simple build step (esbuild or Vite)
- **Add more CORS proxy fallbacks** or consider a self-hosted proxy for full reliability
- **Automated price refresh** — periodic refresh (e.g., every 5 minutes when tab is active)

---

## 12. How to Deploy Changes

1. Download the latest `index.html` from this conversation
2. Open the GitHub repo: `https://github.com/9rftnvtbk7-cloud/folio`
3. Upload/replace `index.html`
4. GitHub Pages auto-deploys within 2-3 minutes
5. Visit `https://9rftnvtbk7-cloud.github.io/folio/`

**To test locally:** Simply open the HTML file in a browser. Firebase auth won't work from `file://`, but you can add `data-folio-noauth` attribute to the `<body>` tag to bypass auth (uses localStorage only).

---

## 13. File Attachment

The current production file is `folio.html` (also deployed as `index.html` on GitHub Pages). All code, styles, and markup are contained within this single file.
