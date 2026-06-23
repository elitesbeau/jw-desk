# JW Desk — Functional Build Plan (Handoff for Review)

> **Purpose of this document:** JW Desk is currently a polished but static single-file web app. This is the plan to make it *functional* — pulling live market data, running an AI agent pipeline, and producing the daily market report dynamically. This document is a handoff for a reviewing agent. Please review for technical soundness, missing steps, cost/risk concerns, and better alternatives.

---

## 0. Current State (Context for Reviewer)

- **The app is a single file:** `/home/user/jw-desk/index.html` — all CSS, HTML, and JS are inline.
- **All data is hardcoded** in one JavaScript object: `const B = {...}` (~one giant line near the top of the `<script>` block). There is also a `const QA = [...]` array holding pre-written chat Q&A responses.
- The UI and page layout are considered **final and approved**. We do not want to redesign the frontend — we want to feed it real data.
- The app renders a "daily market report": a market regime read, a scanner funnel, a ranked list of overnight "movers" with per-name analysis, conviction "calls," a sector rotation widget, breadth, an options-play (HCM) section, and a portfolio page showing the user's book with per-position notes and news.
- The hardcoded sample data already encodes an intended **two-model agent design**: `claude-sonnet-4-6` as the per-ticker analyst, `claude-opus-4-8` as the CIO synthesizer (see the `prov` field in `B`).

**The key insight:** the frontend barely needs to change. Making the app functional means building a backend pipeline that produces the *exact same JSON shape* as the current `B` object, then swapping the hardcoded blob for a `fetch()`.

---

## 1. Target Architecture

```
Market Data APIs
      ↓
  Scanner / Screener  →  Movers list
      ↓
  Analyst Agents (Claude Sonnet, 1 per mover, run in parallel)
      ↓
  CIO Agent (Claude Opus, synthesizes everything)
      ↓
  JSON output (identical shape to current `B`)
      ↓
  Backend serves it  →  Frontend fetches it
```

All intelligence lives in the pipeline. The frontend stays a thin rendering layer.

---

## 2. Phase 1 — Data Sources (Decision Point, ~1 day)

Three categories of data are required. These are the recommended providers; reviewer should sanity-check pricing and coverage.

### 2A. Price & Market Data
**Recommended: Polygon.io** (~$29/mo Starter)
- Real-time quotes, previous close, % change, volume, RVOL, options chains, sector ETF prices.
- REST + WebSocket.
- Alternatives: Alpaca (free, US-only, no options), Tradier (good options data, free for brokerage clients), Finnhub (free tier for basics).

### 2B. Options Flow
**Recommended: Unusual Whales API** (~$50/mo)
- Dark pool blocks, sweep alerts, put/call ratios, net flow per ticker. Populates the `optionsFlow` fields.
- Alternative: compute from Polygon options chain (more engineering).
- **v1 can skip this** and stub `optionsFlow` as neutral, then add later.

### 2C. News / Catalyst Data
**Recommended: Benzinga News API** or **NewsAPI.org**
- Provides the 3 news bullets per position, and gives analyst agents a "news anchor" per mover.

---

## 3. Phase 2 — Backend Service (~1 week)

A server that runs every morning pre-market.

### 3A. Hosting
**Recommended: Vercel + cron** (free tier; serverless functions; built-in cron to trigger the pipeline ~5:30 AM ET; no server to manage). Project is already on GitHub, so Vercel is the path of least resistance.
- Alternatives: Railway ($5/mo, always-on), AWS Lambda + EventBridge (enterprise), VPS (DigitalOcean/Hetzner, full control).

### 3B. Project Scaffold
```
jw-desk-api/
  pipeline/
    scanner.js        ← finds movers
    analyst.js        ← per-ticker Claude agent
    cio.js            ← synthesis Claude agent
    portfolio.js      ← positions + notes
    dataFetch.js      ← all API calls
  api/
    brief.js          ← GET /api/brief → returns the B JSON
  cron/
    run.js            ← triggered ~5:30 AM ET
  store/
    latest.json       ← cached output (last run)
```

### 3C. Data Fetch Layer (`dataFetch.js`)
Functions needed:
```
fetchIndexPrices()     // SPX, NDX, RUT, VIX
fetchMoversList()      // top gainers/losers by % on volume filter
fetchTickerQuotes()    // all watchlist tickers
fetchSectorETFs()      // XLK, XLF, XLE, XLV, XLY, XLP, XLU, XLI, XLB, XLRE
fetchBreadth()         // NYSE adv/dec
fetchOptionsFlow()     // Unusual Whales
fetchNews(symbol)      // 3 headlines per ticker
fetchPortfolioData()   // user positions (see Phase 6)
```

---

## 4. Phase 3 — Agent Pipeline (~1 week)

### 4A. Scanner
```javascript
async function runScanner(tickers) {
  const quotes = await fetchTickerQuotes(tickers);
  const moved = quotes.filter(q =>
    Math.abs(q.changePct) >= 1.0 &&      // 1%+ move
    q.volume > q.avgVolume * 1.2 &&       // volume confirmation
    q.marketCap > 1_000_000_000           // $1B+ float
  );
  return {
    scanned: tickers.length,
    moved: moved.length,
    movers: moved.sort((a, b) => Math.abs(b.changePct) - Math.abs(a.changePct))
  };
}
```
The coverage universe (~50 names) lives in a config file. Seed it from the current ticker bar + portfolio names.

### 4B. Analyst Agent (per mover, cap at 12 for cost)
Call Claude Sonnet with a structured prompt; run all analysts in parallel via `Promise.all()`.

```javascript
async function runAnalyst(ticker, ctx) {
  const news = await fetchNews(ticker.symbol);
  const optionsFlow = await fetchOptionsFlow(ticker.symbol);

  const prompt = `
You are a buy-side research analyst. A name just moved significantly overnight.
Analyze it and return a structured JSON assessment.

TICKER: ${ticker.symbol} (${ticker.name})
MOVE: ${ticker.changePct}% overnight
MARKET CAP: $${ticker.marketCapB}B
VS 50D MA: ${ticker.vs50}%   VS 200D MA: ${ticker.vs200}%

NEWS HEADLINES (last 24h):
${news.map(n => `- "${n.title}" (${n.source})`).join('\n')}

OPTIONS FLOW:
${optionsFlow.summary}

BROAD MARKET CONTEXT:
SPX: ${ctx.spx.changePct}%, NDX: ${ctx.ndx.changePct}%, VIX: ${ctx.vix}

Return a JSON object with these exact fields:
{
  "catalyst": "one line or 'unclear — no headlines surfaced'",
  "read": "2-3 sentence analytical read",
  "setup": "trade setup if any, else 'none — no edge'",
  "risk": "primary risk to the thesis",
  "band": "strong|tilt|lean|none",
  "direction": "long|short|none",
  "invalidation": "what would kill this thesis",
  "relevance": 0-100,
  "isCall": true/false,
  "callThesis": "if isCall, the trade thesis",
  "callInvalid": "if isCall, the invalidation",
  "scoreBreakdown": { "catalyst": 0-25, "optionsFlow": 0-25, "priceAction": 0-25, "sectorAlign": 0-25 },
  "whyNow": "one punchy line",
  "expiresOn": "Today's open|EOW|2-3 sessions|Not actionable",
  "debate": { "bulls": 0-3, "bears": 0-3, "neutral": 0-3, "edge": "one line" },
  "optionsFlow": { "signal": "bullish|bearish|neutral|mixed", "note": "one line", "flowScore": 0-100 }
}
`;

  const response = await claude.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    messages: [{ role: 'user', content: prompt }]
  });
  return JSON.parse(extractJSON(response.content[0].text));
}
```

### 4C. CIO Agent (synthesis, one pass)
```javascript
async function runCIO(analysts, marketContext, portfolioContext) {
  const prompt = `
You are the CIO of a private wealth desk. You have received reports from ${analysts.length} analyst agents.
Synthesize everything into the morning brief.

MARKET CONTEXT:
${JSON.stringify(marketContext, null, 2)}

ANALYST REPORTS:
${analysts.map(a => `${a.symbol}: ${a.read} [score: ${totalScore(a)}]`).join('\n\n')}

PORTFOLIO CONTEXT:
${portfolioContext.positions.map(p => `${p.symbol}: ${p.shares} shares @ ${p.price}, +${p.changePct}%`).join('\n')}

Return JSON with:
{
  "regime": "2-3 sentence tape read",
  "headline": "one crisp line, the desk's single take",
  "topRisk": "the one thing that could make today worse than it looks",
  "noiseNote": "which movers are pure beta noise — explicit list",
  "watch": [3 items: { "symbol", "why" }],
  "convictionCalls": [up to 3: { "symbol", "direction", "whyNow", "expiresOn", ... }],
  "regimeRead": "sector rotation / macro regime in one line",
  "qa": [
    { "q": "How did my book do overnight?", "a": "..." },
    { "q": "Anything worth trading today?", "a": "..." },
    { "q": "What should I worry about?", "a": "..." }
  ]
}
`;

  const response = await claude.messages.create({
    model: 'claude-opus-4-8',
    max_tokens: 4096,
    messages: [{ role: 'user', content: prompt }]
  });
  return JSON.parse(extractJSON(response.content[0].text));
}
```

### 4D. Portfolio Analyst (per holding, one-sentence note)
```javascript
async function analyzePosition(position, news, marketContext) {
  const prompt = `
You are a portfolio analyst reviewing a client's overnight book.
Position: ${position.shares} shares of ${position.symbol} (${position.name})
Overnight move: ${position.changePct}%   Day P&L: $${position.dayChg.toLocaleString()}
Unrealized gain: ${position.unrealPct}%

NEWS (last 24h):
${news.map(n => n.title).join('\n')}

BROAD MARKET: ${marketContext.regime}

Write ONE concise sentence (max 40 words): is the move catalyst-driven or noise,
and is action warranted? Be direct.
`;
  // ...returns the `note` field on each position
}
```

---

## 5. Phase 4 — Assemble `B` and Serve It (~2 days)

```javascript
async function buildBrief() {
  const [indexPrices, quotes, sectors, breadth, optFlow] = await Promise.all([
    fetchIndexPrices(), fetchTickerQuotes(COVERAGE_UNIVERSE),
    fetchSectorETFs(), fetchBreadth(), fetchOptionsFlowSummary()
  ]);

  const { movers, scanned, moved } = await runScanner(quotes);

  const analyses = await Promise.all(
    movers.slice(0, 12).map(m => runAnalyst(m, { indexPrices, optFlow }))
  );

  const cio = await runCIO(analyses, { indexPrices, sectors, breadth }, portfolioData);
  const portfolio = await buildPortfolioSection(portfolioData);

  const B = {
    dateStr: formatDateStr(),
    context: indexPrices,
    funnel: { scanned, moved, agents: analyses.length, calls: cio.convictionCalls.length },
    regime: cio.regime,
    headline: cio.headline,
    topRisk: cio.topRisk,
    noiseNote: cio.noiseNote,
    watch: cio.watch,
    movers: analyses,
    ticker: quotes.map(q => ({ s: q.symbol, p: q.changePct })),
    breadth,
    sectors,
    portfolio,
    hcm: buildHCM(cio.convictionCalls, analyses),
    performance: loadHistoricalPerformance(),
    regimeHistory: cio.regimeHistory,
    prov: { analyst: 'claude-sonnet-4-6', cio: 'claude-opus-4-8' }
  };

  await saveToStore(B);
  return B;
}
```

API endpoint:
```javascript
// GET /api/brief
export default async function handler(req, res) {
  const B = await loadFromStore(); // cached JSON, ~instant
  res.json(B);
}
```

---

## 6. Phase 5 — Connect the Frontend (~half a day)

In `index.html`, replace the hardcoded `const B = {...}` with:
```javascript
let B = null;
async function init() {
  const res = await fetch('https://your-api.vercel.app/api/brief');
  B = await res.json();
  initBoard();          // existing init code runs after B loads
}
init();
```
Add a loading state (skeleton/spinner on the cHead widget) while fetching. Everything else in the frontend stays identical. The `QA` array becomes `B.qa` (or similar) from the CIO output.

---

## 7. Phase 6 — Portfolio Live Data (~1 week, optional but recommended)

Replace hardcoded positions with real broker data. Easiest → most powerful:
1. **Manual `portfolio.json`** — you maintain positions by hand; pipeline reads it. Zero API work. **Start here.**
2. **Schwab API** (Schwab/TD Ameritrade) — official OAuth, refresh token, live positions.
3. **Interactive Brokers** — TWS local API or IBKR Web API. Powerful, more setup.
4. **Alpaca** — cleanest API if paper/live trading there.

---

## 8. Phase 7 — Agent Ruleset Refinement (ongoing)

This is where quality comes from. Iterate on prompts after the pipeline runs end-to-end.

### Conviction Band Criteria (encode in analyst prompt)
```
strong (score 75+):  Confirmed catalyst + options flow alignment + counter-tape strength
tilt   (50-74):      Real catalyst but mixed flow OR strong flow with unclear catalyst
lean   (30-49):      Weak/indirect catalyst, no confirmation
none   (<30):        Pure beta, noise, or uninterpretable
```

### Call Criteria (`isCall: true` requires ALL of)
```
- relevance >= 50
- band == "strong" or "tilt"
- expiresOn != "Not actionable"
- direction != "none"
- catalyst != "unclear"
```

### CIO Override Rules
```
- Never more than 3 conviction calls
- If VIX > 30: all calls downgraded one band
- If breadth < 30% advancing: no long calls
- If a position is in the book AND flagged as a call: mention it in the portfolio note
```

---

## 9. Recommended Build Order

| Step | What | Estimate |
|------|------|----------|
| 1 | Sign up for Polygon.io + get API key | 1 hour |
| 2 | Build `dataFetch.js`, test all data calls | 2-3 days |
| 3 | Build analyst agent prompt, test on 1 ticker | 1-2 days |
| 4 | Build CIO prompt, test with mock analyst outputs | 1-2 days |
| 5 | Wire orchestrator end-to-end, compare to hardcoded B | 1 day |
| 6 | Deploy to Vercel, set up 5:30 AM cron | half day |
| 7 | Update frontend to `fetch()` | half day |
| 8 | Iterate prompts until quality matches/exceeds sample data | ongoing |
| 9 | Wire portfolio (manual JSON first, broker later) | 1-3 days |

**Total to first live brief: ~2 weeks of focused work.**

---

## 10. Key Decisions To Lock Before Starting

1. **Market data provider** — Polygon.io recommended.
2. **Backend hosting** — Vercel recommended (already on GitHub).
3. **Portfolio data** — start manual JSON, upgrade to broker API later.
4. **Options flow** — Unusual Whales best; can stub in v1.
5. **Node vs Python** — Node is easier (frontend is already JS); Python has better data-science libs if needed later.

---

## 11. Open Questions for the Reviewer

- Is the single CIO synthesis pass enough, or should there be a debate/critique round between analysts before synthesis?
- Cost ceiling per daily run? (12 Sonnet calls + 1 Opus + ~10 portfolio notes — estimate and bound it.)
- How is **historical performance** (`performance`, hit-rate tracking) computed? It implies storing past calls and grading them against outcomes — needs a small datastore and a separate end-of-day grading job not yet specified in this plan.
- Should the brief regenerate intraday or strictly once pre-market?
- Error handling / fallback: if a data API or an agent call fails mid-run, does the pipeline serve a partial brief, retry, or serve yesterday's cached B?
- Secrets management for the API keys (Anthropic, Polygon, Unusual Whales, broker) on the host.
- Compliance: the app shows an "internal research note — not investment advice" disclaimer; confirm that framing holds once data is live and a real book is connected.
