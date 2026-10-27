# Gloom free-tier data map (equity, timeseries, reference)

Code review of `gloomberb` at `27bce4f` (app 0.18.5, docs snapshot dated 2026-10-25). The goal: which data the free tier gives, where it comes from, and how coherent it is as an equity dataset.

The live pages at gloom.sh/datasets/docs could not be opened from the review environment, because its network policy blocks gloom.sh and api.gloom.sh. Facts about the Datasets product come from the docs snapshot bundled in `src/docs/snapshot.json`. Everything else comes from the client code and commit history. The Gloom Cloud server is closed source, so upstream attributions marked *inferred* are evidence-based guesses, not facts.

---

## 1. Bottom line

- **There is no free tier of Gloom Datasets.** It is a separate paid REST API (`api.gloom.sh/v1`, `gds_` keys; Basic $200/mo, Desk $800/mo, Enterprise from $2,000/mo, billed in credits). Only three discovery routes need no key: `/v1/datasets` (catalog), `/v1/datasets/cost` (prices a call) and `/v1/datasets/{dataset}/sample` (static example).
- **"Free" means the Gloom Cloud Free plan behind the app**, plus a free News API key with stories delayed 12 hours.
  - On Free, US equities and options are delayed 15 minutes, news 12 hours, and non-US venues are delayed on every plan.
  - Signed out you get search, delayed quotes, charts and delayed news. A free sign-in adds filings, analysts, holders and similar account-backed data.
- **Free has no supported developer API.** The REST API, MCP server and Google Sheets add-in all need Pro. On Free, the only supported programmatic route is the `gloom` CLI (`--json`, `--csv`, `--ndjson`), which calls the app's internal `/market/*` and `/cloud/*` endpoints.
- **Where the data comes from:**
  - Market data and fundamentals are vendor data relayed by Gloom Cloud. Commit history names Twelve Data, Yahoo Finance, Alpaca, OPRA and the SIP.
  - Filings, ownership, short data and macro come from public-domain primary sources (SEC EDGAR, FINRA, FRED, US Treasury, CFTC, EIA and others) that Gloom normalizes.
- **For OpenMarketDataXL:**
  - Free Gloom data is fine for prototyping and checking your schema.
  - It is not a redistributable source. The plan terms and the upstream vendor licences are unknown here, and only Datasets publishes "usage rights".
  - The coherent, legally clean path is to take the public-domain parts straight from the primary sources, using this repo's MIT-licensed parsers as a reference, and to license prices from a paid source (Gloom Datasets "Market data and fundamentals" or another vendor).

---

## 2. Products and what each gives for free

| Product | Free? | Equity / timeseries / reference | Key / access |
|---|---|---|---|
| Gloom Cloud **signed out** (app, term.gloom.sh) | Yes | Search, quotes (15 min delayed US), charts/history, fundamentals, delayed news | None |
| Gloom Cloud **Free, signed in** (verified email) | Yes | Adds filings, analysts, holders, scanners, sync, chat | Account |
| Gloom Cloud **Pro** ($, 7-day trial) | No | Real-time US equities/options, streaming, options flow, transcripts, hiring, KPIs/guidance in full, MCP, REST API, Sheets | `gloom_mcp_` key |
| **News API** free key | Yes | News only, 12 h delayed (Unlimited $99/mo is real-time) | `gloom_news_` |
| **Gloom Datasets** | **No** (paid only) | "Quotes and history, financial statements, statistics, profiles, earnings, estimates, IPOs, M&A, screener, options and gamma positioning, futures, rate path, crypto, session tape", plus filings, macro and alternative data | `gds_` |

Sources: `src/api-client/plan-rules.ts` (`CLOUD_QUOTE_DELAY_MINUTES = 15`, `CLOUD_NEWS_DELAY_HOURS = 12`), and the docs snapshot pages "Gloom Cloud", "Install", "MCP server", "Google Sheets", "News API" and "Gloom Datasets".

Datasets responses carry `meta` with the dataset's sources, attribution and usage terms, `asOf`, `historyStarts`, `coverage` and `lastUpdated`, and return `null` for anything the stored data cannot establish. The per-dataset route list, credits and coverage are only on the live site, which I could not reach.

---

## 3. Where the data comes from

### 3.1 The pipe

Since **2026-10-02** (commit `440eaeac`, "Route market data through the managed backend") **every market request goes through Gloom Cloud** (`https://api.gloom.sh`). `GloomberbCloudProvider` (`src/sources/gloomberb-cloud/index.ts`) is the only market-data provider built into the app; connected brokers (IBKR, Robinhood and others) can supply quotes too.

Before that, the app shipped a **"Yahoo Fallback"** plugin that called Yahoo directly, with no account: `query1.finance.yahoo.com` `v8/finance/chart`, `v10/finance/quote`, `v7/finance/options`, `quoteSummary`, `ws/fundamentals-timeseries`, `v1/finance/search`, plus the `getcrumb` handshake. It is in git history at `440eaeac^:src/sources/yahoo-finance*`.

### 3.2 Upstreams, as far as the code shows

| Data | Upstream | Evidence |
|---|---|---|
| Global symbol catalogue, search, listing names | **Twelve Data** | Commit `3e1f2be3`: "Twelve Data's symbol catalogue files BAE Systems' London line … and Gloom Cloud search passes that name through". |
| Quotes / history for many venues | **Twelve Data** *(inferred)* | Request shape is Twelve Data's: `interval=1min…1month`, `outputsize`, `start_date`/`end_date`. The errors the client matches are Twelve Data's verbatim: "symbol or figi parameter is missing or invalid", "Data not found". Payloads carry `micCode` and `exchangeTimezone`. |
| Quote fields, statement line names, analyst/corporate-action fallback | **Yahoo Finance** *(Yahoo-shaped, likely still upstream for part of it)* | `marketState` enum `PRE/REGULAR/POST/PREPRE/POSTPOST/CLOSED`; statement keys such as `normalizedIncome`, `netIncomeContinuousOperations`, `gainsLossesNotAffectingRetainedEarnings`; commit `45158838` "when Cloud/Twelve Data returns no usable payload", commit `c6224d4b` (Yahoo corporate actions fallback). |
| US real-time / streaming quotes and tape | **SIP** (consolidated tape), **Alpaca** named | Commit `7ca92f90` removed "Yahoo, Alpaca, OPRA and SIP" from pane footers. Time & Sales declares "Gloom Cloud SIP trade and quote feed … SIP is delayed fifteen minutes without realtime entitlement". |
| In-progress US session volume | **IEX** | `venueVolume`: "a US session still trading, counted on IEX alone" (`src/api-client/types.ts`). |
| US options chains | **OPRA** | `OptionsChain.feed: "opra" \| "gloom" \| "deribit"`. |
| Filings, 10-K/10-Q XBRL statements, Form 4, 13F, 13D/G, proxies | **SEC EDGAR** | `/cloud/sec/*`; `src/sources/sec-edgar.ts` reads `company_tickers_exchange.json`, `data.sec.gov/submissions`, `data.sec.gov/api/xbrl/companyfacts`. |
| Short interest, daily short volume, ATS weekly, TRACE | **FINRA** | Plugin `dataRequirements`: "FINRA short interest settlements", "Gloom Cloud FINRA daily short volume", "FINRA weekly off-exchange volume". |
| Fails to deliver | **SEC** | "SEC settlement fails". |
| Macro series | **FRED**, **US Treasury** fiscaldata, **NY Fed**, **CFTC** (COT), **EIA**, **USDA**, **DTCC** (CDS) | Plugin clients and docs. |
| Non-US statement gaps | **ASX** pages | `StatementFill.source: "asx"`. |
| Fund descriptions | Issuer or exchange pages | `CompanyProfile.descriptionSource`. |

Panes are deliberately forbidden from naming providers (AGENTS.md, `pane-conventions`), so the running app won't tell you which vendor served a given value.

---

## 4. The free equity dataset in detail

### 4.1 Identity and reference data

The listing key is **`SYMBOL` + canonical venue code**: bare symbol for US venues, `SYMBOL:EXCHANGE` elsewhere (`RCI.B:TSX`, `7203:JPX`, `SAN:EPA`). Hong Kong codes are padded to four digits.

| Item | Route | Fields | Notes |
|---|---|---|---|
| Search | `/market/search`, `/market/search/batch` | symbol, name, exchange, type, currency, primaryExchange, popularity, searchAliases (trade names), searchKeywords (themes), formerName, formerIssuer, delisting, homeListing | Catalogue from Twelve Data. |
| Issuer grouping | `/market/issuer-listings` | `issuer` key (e.g. `cik:733099`), name, ISO country, every listing (symbol, venue, primary flag) | Links share classes, ADRs and dual listings. |
| Ticker changes | `/market/ticker-changes`, `…/batch` | hops `from → to` on date, `moved`, `formerly`, `formerIssuer` (merger successor, with SEC source) | **US only, history from April 2017, served signed out.** |
| Delisting | in quote, history and search | `status` delisted/inactive, `lastTradeDate`, `historyStart` | US listings. |
| Quote metadata | `/market/quote` | instrumentType, currency, exchangeName, fullExchangeName, listing and routing exchange, `providerPriceDivisor` (100 for GBp) | |
| Depositary receipts | quote and fundamentals | `isDepositaryReceipt`, `adrRatio`, `depositaryPair` (receipt, ratio, home lines) | |
| Company profile | `/market/financials` | description (+ source), sector, industry, employees, website, **SIC + description**, **fiscal year end** | SIC and FYE for US filers. |
| Venues | `/market/venues` | MIC, name, country, city, timezone, lat/long, open/close timers | |
| Index / ETF members | `/cloud/members/{ETF}` | dated holdings, member returns, breadth | Survivorship-biased breadth; **no Nasdaq-100**. |
| Venue coverage | `src/sources/listing-symbols.ts` | About 70 venue codes: US, Canada, Japan, HK, China (SSE/SZSE), Taiwan, Korea, Singapore, Indonesia, India, Australia/NZ, SE Asia, London, Germany, Euronext, Italy, Spain, Switzerland, Nordics, CEE, Turkey, Israel, Gulf, South Africa, Brazil, Mexico, Argentina, Chile, Egypt | Full per-venue coverage is on gloom.sh/data-coverage. |

**Missing:** payloads carry no **ISIN, CUSIP or FIGI**. The only cross-reference key is the issuer's **SEC CIK**, and only US filers have one. Build your own security master that maps `SYMBOL:VENUE` and CIK to ISIN and FIGI, for example with OpenFIGI.

### 4.2 Quotes (`/market/quote`, `/market/quotes/batch`)

`Quote` (`src/types/financials.ts`):

- **Price and move:** price, change, changePercent, previousClose (+ date), regularClose (+ session date), open/high/low, 52-week high/low, volume, marketCap, bid/ask (+ sizes), lastTradePrice/Time.
- **Extended hours:** pre-market, post-market and overnight price/change, `marketState`.
- **Freshness:** `dataSource` live/delayed, `lastUpdated`, `stale`.

| Asset | Free | Pro |
|---|---|---|
| US stocks and ETFs | 15 min delayed | Real-time, streaming |
| Non-US venues | Delayed | Delayed |
| Crypto | Real-time | Real-time |
| US options, FX | 15 min delayed | Real-time |

### 4.3 Price timeseries (`/market/history`)

**Parameters:** `symbol`, `exchange`, `interval` (`1min 5min 15min 30min 45min 1h 1day 1week 1month`), `outputsize`, `startDate`, `endDate`, `session=extended` (US intraday only).

**Bar fields:** date, OHLC, volume. Optional flags:
- `venueVolume`: IEX-only count while a US session is still trading; never treat it as market volume.
- `partial`: the bar's period hadn't ended when served.
- `filled`: rebuilt from shorter bars.
- `unsettled`: futures session closed but settlement not confirmed.

**Response metadata:**
- `coverage`: firstBarDate, lastBarDate, barCount, verifiedLineageStart, firstAllowedBarDate, inceptionDate, note.
- `currency`.
- `omittedSessions` (e.g. non-positive closes).
- `delisting`.
- `historySession`: US session contract.
- `servedResolution`: `1day` when a listing has no intraday bars.

**Depth by resolution** (`CLOUD_RESOLUTION_SUPPORT`):

| Resolution | Max range |
|---|---|
| 1m | 1W |
| 5m | 1M |
| 15m | 3M |
| 30m | 6M |
| 1h | 1Y |
| 1d | 5Y |
| 1wk | 5Y |
| 1mo | ALL (up to 50 years) |

- The server can also declare an intraday **retention limit** (`HISTORY_RETENTION`), which the client accepts up to 730 days.
- One CLI request returns daily bars for windows up to 5 years, weekly up to 20 years and monthly beyond. For long daily series, request consecutive windows of 5 years or less (`getDetailedPriceHistory` takes exact start and end dates). Whether the server serves daily bars before the last 5 years is not established client-side.

**Adjustment:**
- Prices are **split-adjusted**, not dividend-adjusted. The docs say "price history is split-adjusted the same way" as restated EPS.
- Total return is computed **client-side**: cash distributions are reinvested at the ex-date close (`src/time-series/total-return.ts`), from the dividend history.

**Units:**
- History is in **major currency units** unless the response declares a subunit.
- Quotes can be in subunits (`GBp`, with `providerPriceDivisor`).
- Normalize both before you combine them.

**Survivorship:** delisted US listings keep their daily history up to `lastTradeDate`. Index breadth uses current members only.

**Quality checks you get for free:** OHLC contradiction flags, gap and jump detection, and a coverage start (CLI `history`).

### 4.4 Fundamentals and statements (`/market/financials`, `…/batch`, `statementHistory=extended`)

**Statistics (`Fundamentals`):**
- **Valuation:** marketCap (+ currency), trailing and forward P/E (+ forwardEps), PEG, EV (with an `enterpriseValueCheck` reconciliation against cap + debt − cash), EV/revenue.
- **Income and cash:** revenue, netIncome, EPS (TTM from the last four filed quarters, with `trailingEpsSource`), margins, growth rates, operating and free cash flow.
- **Dividends:** dividend yield and rate (forward or trailing basis), exDividendDate.
- **Shares:** sharesOutstanding (+ as-of date from the 10-K/10-Q cover), floatShares, split basis.
- **Short interest:** sharesShort, shortPercentOfFloat and shortRatio, dated by FINRA settlement.
- **Ownership and risk:** insider and institution percentages, beta, 1-year and 3-year returns.

**Statements (`FinancialStatement`):**
- About 150 Yahoo-style line items across income, cash flow and balance sheet, annual and quarterly.
- Per row: `currency`, `availableAt`, per-field `fieldAvailability` (publication dates), `fieldSources` (SEC concept and accession), `dateEvidence`, `unavailableFields` (declared gaps), `shareBasis`, `epsBasis` (SEC split-adjusted).
- `statementHistory=extended` asks for deep SEC XBRL history: "deep" means at least 5 annual or 8 quarterly rows.

**Point-in-time:**
- Publication dates are given per field.
- Values are the **latest snapshot**: restatements overwrite earlier figures and historical vintages are not reconstructed (`docs/research-data.md`).
- The fourth quarter is derived as the year minus nine months, with a restatement guard.

### 4.5 Corporate actions and events

| Data | Route | Fields |
|---|---|---|
| Dividends | `/market/corporate-actions`, `/market/dividends` | Paid `{exDate, amount}`; declared (declaration, record and pay dates); `declaredCash` in the declaring currency |
| Splits | same | date, ratio, from/to factor (spin-off adjustments look the same) |
| Earnings | same; `/cloud/earnings/calendar`, `/cloud/earnings/history` | date, BMO/AMC, EPS estimate and actual, surprise, `epsBasis` gaap/adjusted, `dateStatus` reported/confirmed/projected |
| IPO calendar | `/cloud/ipo/calendar` | Merged from each exchange's own listings |
| M&A | `/cloud/mna/deals` | 7 days delayed on Free |

### 4.6 Ownership, analysts and short-side data

All free, some needing sign-in:

- **13F holders:** SEC index, quarterly.
- **13D/G:** four years of filings.
- **Insiders:** Form 4.
- **Short interest:** FINRA, twice a month.
- **Daily short volume:** FINRA.
- **Fails to deliver:** SEC.
- **ATS:** FINRA, weekly.
- **Analysts:** ratings history, price targets, recommendation trend, EPS and revenue estimates; free sign-in.
- **Estimate revisions:** cloud snapshots, free sign-in.

### 4.7 Cross-sectional universe: the equity screener (`EQS`, `/cloud/equity-screener`)

- A **daily snapshot, statements as filed**, paged 200 rows at a time with a cursor.
- **Universe stats:** covered/skipped, per-field known/missing/stale counts, currencies, sectors, exchanges.
- **Categories:** symbol, exchange, currency, sector, industry.
- **Metric groups** (about 70 in all):
  - price/volume, valuation, margins, growth, returns, 52-week position;
  - technicals (RSI, SMA distance, MACD/golden cross, Bollinger);
  - ownership (shares, float, short % of float, 13F %);
  - short interest, insider and institutional flows;
  - IV/HV, analyst upside and EPS revisions, beta, ROE, EV/EBITDA, P/B, dividend payout and growth;
  - social attention.
- Each metric comes with `asOf`, `availableAt`, `source`, `state` and a percentile.
- **KPI, guidance, hiring and app fields are withheld on Free** ("Needs Gloom Pro").
- This is the quickest way to get a universe list plus a reference snapshot.

### 4.8 Macro alongside equities

All free:
- **FRED** series with no sign-in (`gloom fred --list`), Treasury yields (`GC`), the economic calendar and central-bank rates.
- **Positioning and commodities:** CFTC COT, EIA, USDA.
- **Credit:** CDS/CDX from DTCC (delayed), TRACE.

### 4.9 Pro-only or preview (not usable as a free dataset)

| Status | Functions |
|---|---|
| Pro only | `RISK` (risk factors), `EK` (8-K events), `CALLS` (transcripts), `JOBS`, `SRCH` (document search), `FLOW` (options flow) |
| Preview | `KPIS`/`GUIDE` (latest three), `SEG` (first two revenue lines), `EXEC` (total pay only), `DIAG`, `CRDOC`/`COVN`, `CATL`/`LITI`, `AWARDS`, `SPLC`, `EXPO`, `HIRE`/`APPS`, `PERP`, `POWER`, `GPU`, `ATTN` |

The full matrix is in `src/cli/pane-functions/function-help.ts` (`FUNCTION_HELP[*].access` and `.data.free`).

---

## 5. Getting free data out programmatically

The supported free route is the CLI (`bin/gloom`). It works signed out, and `gloom` login adds the sign-in tier.

```sh
gloom search "berkshire hathaway" --json
gloom quote AAPL MSFT 7203:JPX --json
gloom history --file universe.txt --from 2021-01-01 --to 2025-12-31 --csv > bars.csv   # 8 symbols in flight
gloom fundamentals NVDA --json          # statistics + profile
gloom financials MSFT --json            # annual/quarterly statements
gloom events KO --json                  # dividends, splits, earnings
gloom holders AAPL --json ; gloom 13f AAPL --json ; gloom insider AAPL --json
gloom filings AAPL --json
gloom fn EQS --json                     # screener snapshot (see `gloom help fn`)
gloom fred DGS10 --start 2000 --csv
```

`history --csv` always uses one header: Date, OHLCV, Flag, Partial, Filled, Unsettled, volume unit/note/venue, status, symbol_or_series, currency_or_unit, interval, source, as_of.

**Caveats:**
- The `/market/*` and `/cloud/*` routes behind it are an **internal, undocumented** API of a closed hosted service.
- Expect rate limits (`429`/`retryable_error`).
- Don't build a product that scrapes them. The documented developer surfaces are Pro (REST/MCP/Sheets) or Datasets (paid).

---

## 6. Coherence checklist for OpenMarketDataXL

1. **Security master:** key on `SYMBOL:VENUE` plus the `issuer` key (CIK for US filers) from `/market/issuer-listings`. Add ISIN and FIGI yourself. Use ticker-change hops (US, since April 2017) and the `delisting` and `formerIssuer` fields for point-in-time symbol mapping.
2. **Mixed freshness:** US is 15 min delayed and non-US delayed by different amounts; daily bars are final only after the close plus 15 min. Store `asOf` and `dataSource` with every value.
3. **Adjustment consistency:** prices are split-adjusted; statements are as reported but EPS is put on a split-adjusted basis. Keep a raw splits table and build total return from dividends yourself.
4. **Units:** normalize subunits (GBp, ZAc, ILA) through `currency` and `providerPriceDivisor`. Statement `currency` can differ from listing currency, and `financialCurrency` and `ttmConversion` tell you when TTM figures were converted.
5. **Depth:** daily bars come 5 years per request (use windowed requests); intraday is capped by resolution (1-minute bars cover 1 week); statements go deep only via SEC XBRL (US filers).
6. **Point-in-time fundamentals:** use `availableAt` and `fieldAvailability`, and accept that restatements overwrite.
7. **Licensing:**
   - SEC, FINRA, FRED, Treasury, CFTC and EIA data is public. Ingest it directly; this repo's `src/sources/sec-edgar.ts` (companyfacts → statements, submissions → filings, `company_tickers_exchange.json` → ticker↔CIK) is MIT and no longer used at runtime, so it is a ready reference. SEC requires a descriptive `User-Agent` with contact details (`SEC_USER_AGENT`).
   - Vendor-derived prices, fundamentals statistics, analyst data and the symbol catalogue need a licence that allows redistribution in your product: Gloom Datasets (check its "usage rights" on the pricing page) or a vendor directly.

---

## 7. Not verified here

- The live Datasets catalog: dataset ids, per-route credits, rate limits, usage rights, and `historyStarts`/`coverage` per dataset. With network access, call `curl https://api.gloom.sh/v1/datasets` and `/v1/datasets/{id}/sample`; neither needs a key.
- The per-venue coverage page (gloom.sh/data-coverage).
- Server-side mapping of each field to an upstream vendor. The attributions in §3.2 are from client code and commit messages.
- Whether the server serves daily bars older than 5 years in windowed requests.
- Which `/market/*` routes refuse signed-out callers. The client doesn't declare it; the docs say filings, analysts and similar panes need a free sign-in.
