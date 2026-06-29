---
name: stock-research
description: >-
  Generate an investor-grade research memo for any US public stock. Pulls SEC EDGAR filings, Yahoo Finance market data, insider signals, and news — then applies the frameworks of 8 sector-focused institutions via a Smart Money lens. Includes an interactive pre-check (preliminary conviction + data quality check before committing to full research). Outputs HTML + JSON to ~/Documents/stock-research/{ticker}.html. Use when user says "research [ticker]", "refresh [ticker]", "add [ticker] to watchlist". ~$0.28 per full memo, ~7–9 minutes. Pre-check only ~$0.02, ~30 seconds.
license: MIT (personal use)
---

# Stock Research Skill — v3.0 Spec

**Role:** Generate an investor-grade research memo for any US public stock. No hard investment philosophy gate — the tool is a neutral research instrument. Smart Money (§11) is the centerpiece: 8 sector-focused institutions' frameworks applied to the ticker. An AI Assessment section gives Claude's independent view without prescribing a trading framework.

**Scope:** US-listed public companies only (NYSE, NASDAQ). Private companies use the separate v1.2 skill.

**Budget:** ~$0.25–0.32 per Mode A run (was $0.20 in v2.0; Smart Money adds one LLM call + occasional firm scrape). ~$0.05 per Mode B run (Smart Money is sector-cached, not re-scraped per ticker).

---

## §0 Changes since v2.0

If you've read v2.0, here's what's different. If this is your first read, skip to §0.1.

- **New memo section: §11 Smart Money Perspective** — 8 firms (5 primary VC + 3 secondary public-market), with framework inference applied to the company. Inserted between §10 Trading Signals and Entry/Exit. See §6 §11 for template.
- **Memo template grew 13 → 14 sections.** Sections 11–13 in v2.0 are now §12–§14 in v2.1.
- **Per-section comment system** in HTML output. Each section has a 💬 button → expand inline textarea → auto-saves to browser localStorage. Saved comments appear in PDF exports.
- **PDF Export button** in topnav (in addition to footer). Print stylesheet auto-expands all comments.
- **Branding:** "Marisa Ma / Stock Research" replaces previous neutral branding. CONFIDENTIAL badge removed.
- **NEW: Sector → Firms Registry** at `~/.claude/skills/stock-research/sector-firms.json`. Maps sector → 8 firms (5 primary + 3 secondary) with public blog URLs. Refreshed monthly via cache. See §14.
- **NEW: Cache Maintenance** — `~/.claude/skills/stock-research/cache/firms/` stores per-firm framework summaries with 30-day TTL. See §15.

Cost / time changes:
- Mode A: ~$0.20 → ~$0.28 per run; ~6–8 min → ~7–9 min
- Mode B: ~$0.05 unchanged (Smart Money is sector-level, not per-ticker)
- Mode C: $0 unchanged

Spec length: ~1150 lines (was 872).

---

## §0.1 Design Principles (v3.0)

1. **No hard gate.** Any US public stock is valid input. The tool does not filter by drawdown threshold or enforce a trading philosophy. It surfaces data and let the user decide.

2. **Interactive pre-check before full research.** After a fast preliminary data pull (~30s, ~$0.02), the skill shows a preliminary conviction score + data quality flags and asks whether to proceed. This avoids spending 8 minutes and $0.28 on a stock the user rules out in 5 seconds.

3. **Smart Money is the centerpiece.** The §11 Smart Money section applies the frameworks of 8 sector-focused institutions to the specific ticker. This is the tool's primary differentiator — giving retail investors access to the mental models that sector specialists use.

4. **AI Assessment, not prescribed framework.** The hero and §1 Snapshot include an AI View section — Claude's independent read on whether the stock is worth further investigation. It is informational, not a gate. The user remains the decision-maker.

5. **Observability by default.** Every run logs to `~/.claude/skills/stock-research/runs.csv` (cost, tokens, status). Data quality issues are flagged inline, not silently swallowed.

---

## §0.2 Interactive Pre-Check Flow (NEW in v3.0)

This is a lightweight preliminary step that runs BEFORE the full 7–9 minute research. The goal: surface enough signal in ~30 seconds for the user to decide whether to proceed.

### Pre-check steps (fast, no Firecrawl/Apify):
1. Validate ticker exists in SEC EDGAR registry
2. Fetch Yahoo Finance quote (price, 52W high/low, market cap, P/E, short interest)
3. Run a single Claude call (~$0.02) to produce a preliminary conviction score (1–10) and a 3-bullet rationale

### Pre-check output to user:
```
RBLX — Roblox Corporation
Price: $56.09  |  52W range: $32.20–$150.65  |  Down 62.8% from high
Market cap: $9.8B  |  P/E: N/A (loss-making)  |  Short interest: 8.2%

Preliminary Conviction: 7/10

Why this might be worth researching:
• Platform with 88M DAU — user engagement has held despite stock collapse
• Down 62% from high on profitability concerns, not product failure
• Microsoft / Alphabet reportedly in partnership talks

Potential concerns:
• Still loss-making; no clear path to FCF positive in near term
• Metaverse narrative has cooled significantly
• 13–17yo demographic dependency is a long-term overhang

Data quality: ✅ SEC filings current (10-Q filed 3 weeks ago) · ✅ Yahoo data fresh

Proceed with full research? (yes / no / see more)
```

If user says **yes** → continue to full Mode A research.
If user says **no** → log pre-check to runs.csv, stop. Cost: ~$0.02.
If user says **see more** → fetch 2-3 recent news headlines and re-ask.

---

## §1 Inputs & Outputs

### Inputs
- Ticker (string, uppercase) OR company name
- Mode (string): `research` | `refresh` | `add` — auto-routed from user phrasing

### Outputs

**Mode A: Research (new)**
- `~/Documents/stock-research/{ticker}.html` — 14-section memo with 💬 comment buttons, PDF export, "Marisa Ma / Stock Research" branding
- `~/Documents/stock-research/{ticker}.json` — state file
- Notion: untouched

**Mode B: Refresh**
- Overwrites HTML (with "Since last run" diff section at top) and JSON
- Smart Money §11 reused from sector cache (not re-scraped per ticker)
- Notion: if ticker exists in Watchlist, updates Conviction, Position, Notes, Last Updated

**Mode C: Add to Watchlist**
- Notion: creates new row from JSON sidecar values
- HTML / JSON: untouched

### Success Criteria (Mode A)
- Both files written; absolute paths printed
- Hero: 7-cell metric bar (Price / Drawdown % / Mkt Cap / P/E / Setup Gate / AI Conviction / AI Position)
- All 14 sections rendered
- §3 has clear judgment + reasoning
- **§11 Smart Money** has 8 firm cards (5 primary + 3 secondary), 5 framework blocks applied to ticker, synthesis (agree/diverge), and "framework applied — not direct quotes" disclaimer
- §12 (Entry & Exit) explicit on batch sizes and exit triggers
- AI Conviction and AI Position visible in hero
- Setup Gate ✓ or ✗ shown; warning if ✗
- "Marisa Ma / Stock Research" in topnav and footer; no Confidential badge
- PDF Export button visible in topnav
- 💬 comment button on every section chip; clicking expands a yellow textarea
- Comments persist in localStorage with key `comment_{ticker}_section-{N}`
- Print stylesheet auto-expands all comments

### Success Criteria (Mode B)
- HTML re-rendered with diff section at top
- §11 Smart Money preserved (only re-synthesized if conviction or problem-type changed)
- JSON updated
- Notion Watchlist row updated if exists

### Success Criteria (Mode C)
- New row in Stock Watchlist using JSON `ai_conviction`, `ai_position`, `ai_thesis_summary`
- Entry Price left blank
- HTML / JSON untouched

---

## §2 Workflow — 3 modes

| Step | Mode A | Mode B | Mode C |
|---|---|---|---|
| Disambiguate ticker | ✅ | ✅ | (read JSON) |
| Read existing JSON | – | ✅ | ✅ |
| SEC EDGAR fetch | ✅ full | ✅ delta only | – |
| Yahoo Finance fetch | ✅ | ✅ | – |
| Insider + 13-F (Form 4 / Whalewisdom) | ✅ | ✅ | – |
| News + IR scrape | ✅ 5–8 articles | ✅ 3–4 (last week) | – |
| **Smart Money — sector → firms cache** | ✅ read cache | ✅ same cache | – |
| **Smart Money — LLM synthesis** | ✅ | ✅ if conviction/thesis moved; else preserve | – |
| LLM extraction (Groups 1–4) | ✅ | ✅ partial | – |
| Memo synthesis (14 sections) | ✅ | ✅ + diff section | – |
| Render HTML + JSON sidecar | ✅ | ✅ overwrite | – |
| Notion write | – | ✅ if row exists | ✅ create row |
| **Cost** | **~$0.28** | **~$0.05** | $0 |
| **Time** | 7–9 min | 2–3 min | <30 sec |

**Mode router** (skill picks mode from user phrasing):
- "research X" / "do a memo on X" → Mode A
- "refresh X" / "X after earnings" / "refresh my watchlist" / "weekly refresh" → Mode B
- "add X to watchlist" / "track X" → Mode C (requires existing JSON)

---

## §3 Step-by-Step (Mode A: Research)

### 3.1 Disambiguation

User input: `LULU`, `lulu`, `$LULU`, or `Lululemon` → strip symbols, uppercase ticker, resolve company name.

Search: `"{input}" stock ticker NYSE OR NASDAQ`. If ambiguous, stop and ask.

Resolve **CIK** from `https://www.sec.gov/files/company_tickers.json`. Left-pad to 10 digits.

### 3.2 SEC EDGAR — direct API

Required header on ALL SEC requests:
```
User-Agent: stock-research-skill {your_email}
```

Fetch:
- `https://data.sec.gov/submissions/CIK{padded_cik}.json` — filings list
- `https://data.sec.gov/api/xbrl/companyfacts/CIK{padded_cik}.json` — financial concepts

Pull these XBRL concepts (last 8 quarters + last 3 fiscal years):
- `Revenues` or `RevenueFromContractWithCustomerExcludingAssessedTax`
- `NetIncomeLoss`, `EarningsPerShareDiluted`, `OperatingIncomeLoss`, `GrossProfit`
- `Assets`, `Liabilities`, `CashAndCashEquivalentsAtCarryingValue`, `LongTermDebt`, `StockholdersEquity`
- `NetCashProvidedByUsedInOperatingActivities`

Firecrawl scrape latest 10-K and 10-Q narrative sections (Item 1A Risk Factors, Item 7 MD&A).

### 3.3 Yahoo Finance — current market data

Apify actor `cryptosignals/yahoo-finance-scraper` (free) or `kaix/yahoo-finance-scraper` (~$0.0005/stock).

Input:
```json
{ "tickers": ["{TICKER}"] }
```

Extract: current price, 52W high/low, market cap, P/E (TTM, Forward), beta, dividend yield, average volume, short interest %, analyst consensus, last/next earnings date.

Compute:
```
drawdown_pct = (52w_high - current_price) / 52w_high
setup_gate = "✓ Eligible" if drawdown_pct >= 0.50 else "✗ Below 50% threshold"
```

### 3.4 Insider + Institutional signals

- Form 4 via Firecrawl on `http://openinsider.com/screener?s={ticker}` (last 6 months)
- 13-F via Firecrawl on `https://whalewisdom.com/stock/{ticker}` (last 2 quarters)
- Short interest from Yahoo, plus `https://www.nasdaq.com/market-activity/stocks/{ticker}/short-interest`

Used as §10 tiebreakers. Don't bet thesis on these alone.

### 3.5 News + IR scrape

Search:
- `"{ticker}" earnings`, `"{ticker}" guidance`
- `"{company}" {sector} {current year}`
- `"{ticker}" {recent date}` for event-driven moves

Prioritize: WSJ, Bloomberg, Reuters, FT, Seeking Alpha, sector trade press, IR page (`https://investors.{company}.com/`).

Firecrawl full text for top 5–8 articles past 12 months. Aim for ≥1 earnings call transcript.

### 3.6 LLM extraction — 4 prompts (Groups 1–4 unchanged from v2.0)

**Group 1: Identity & Setup**
```json
{
  "company_name": "string",
  "ticker": "string",
  "one_liner": "<25 words",
  "executive_snapshot": "MANDATORY 3 sentences: business + scale, drawdown trigger, central question",
  "sector": "Tech/Software | Tech/Hardware | Healthcare/Pharma | Healthcare/Devices | Healthcare/Services | Consumer Discretionary | Consumer Staples | Financials | Energy | Materials | Industrials | Real Estate | Utilities | Communications",
  "drawdown_narrative": "2-3 sentences",
  "current_sentiment_temp": "Cold | Cool | Hot | Boiling"
}
```

**Group 2: Short-Term vs Structural Judgment**
```json
{
  "problem_type": "Short-term | Structural | Mixed | Unclear",
  "judgment_reasoning": "3-4 sentences with specific evidence",
  "recovery_window_estimate": "12-18 months | 18-36 months | 3+ years | Unknown",
  "thesis_summary": "1 sentence",
  "biggest_uncertainty": "1 sentence"
}
```

**Group 3: Financial Trajectory** — see v2.0 §3.6
**Group 4: AI Recommendation** — see v2.0 §3.6

Shared rules: literal `$`, return null if missing, cite source URLs.

### 3.7 NEW: Smart Money — sector → firms lookup + cache fetch

After Group 1 returns sector classification:

1. Open `~/.claude/skills/stock-research/sector-firms.json` (see §14 for schema)
2. Look up the 8 firms for this sector (5 primary + 3 secondary)
3. For each firm, check cache at `~/.claude/skills/stock-research/cache/firms/{firm-slug}.json`:
   - **Fresh** (< 30 days old): use cached `framework_summary` + `latest_blog_post_excerpt`
   - **Stale or missing**: Firecrawl `firm.blog_url`, extract 5 most recent posts, summarize their current sector view in 2-3 sentences, write to cache with `expires_at = now + 30 days`
4. Return 8 `firm_context` objects with: `name, tier, framework_descriptor, current_view_excerpt, last_post_date, last_post_url`

**Sector lookup → cascade fallback (NEW v2.2):**

When sector is NOT fully populated (5+3) in registry, do NOT immediately skip §11. Instead, run this cascade in order. Stop at the first level that yields ≥3 high-quality firms / artifacts; only fall through to graceful skip as last resort.

```
Sector status         → Action
─────────────────────────────────────────────────────────────────
5+3 populated         → Full mode (current behavior, see steps 1-4 above)
3+1 to 4+2 populated  → Partial mode (run with what's there + flag in §14)
0+0 (sector empty)    → ↓ Bridge Mode (§3.7.5)
   bridge fails       → ↓ Live Discovery (§3.7.6)
      live fails      → ↓ Graceful Skip (§3.7.7) — last resort
```

**If all 8 firms have scrape failures:** Degrade — use only `framework_descriptor` from registry (no fresh blog excerpts). Note in framework blocks: "Cached framework only; no recent excerpts."

### 3.7.5 NEW: Bridge Mode (sector empty, borrow from adjacent registered sectors)

When the current sector has 0+0 firms in registry, BEFORE giving up, search adjacent registered sectors for firms whose framework_descriptor is keyword-relevant to the current ticker context.

**Bridge matching logic:**

1. Collect keywords from Group 1 LLM output: ticker, company_name, sector, industry, executive_snapshot
2. For each registered sector (excluding the current empty one), score each firm's `framework_descriptor` against the keyword set
3. Keyword categories that yield strong cross-sector bridges:
   - **Platform / aggregation / network effects** → Stratechery (Tech/Software), a16z (Tech/Software), Coatue (Tech/Software), Bessemer (Tech/Software)
   - **DTC / brand-as-distribution / creator economy** → L Catterton (Consumer Discretionary), Forerunner (Consumer Discretionary)
   - **Quality compounder / reinvestment runway** → Akre Capital (Consumer Discretionary), Polen Capital (Consumer Discretionary)
   - **Activist value / cash-generative consumer** → Pershing Square (Consumer Discretionary)
   - **Subscription / recurring revenue** → Bessemer (Tech/Software, Cloud Index frameworks)
   - **Hardware / semis / capex cycles** → SemiAnalysis, Khosla, Lux (Tech/Hardware)
4. If ≥3 firms match across 1-2 source sectors → render §11 in **Bridge Mode**:
   - Render firm cards with explicit "BRIDGED · `{source_sector}`" tier label (replaces "PRIMARY VC")
   - Add a callout at the top of §11: "Smart Money bridged from `{source_sector}` registry — `{current_sector}` sector not yet populated. Per spec §3.7.5."
   - Run §3.8 synthesis using the bridged firms; framework_blocks include explicit "Applied to `{TICKER}` (cross-sector framework — `{firm}` is registered under `{source_sector}` but `{specific_thesis_relevance}`)"
5. If <3 firms match → fall through to §3.7.6 Live Discovery

**Cost impact:** Bridge mode cost ≈ Full mode cost (~$0.05 in §3.8 synthesis). No additional Firecrawl calls if cache is fresh for the borrowed firms; otherwise per-firm cache refresh as in §3.7 step 3.

### 3.7.6 NEW: Live Discovery (Firecrawl authoritative public sources)

When neither full-mode nor bridge mode yields ≥3 firms, attempt live discovery of authoritative public sources for the ticker / sector. This pulls REAL public artifacts (not curated registry entries) and applies framework inference based on what's found.

**HARD CAPS (enforced in code, not optional):**

```
LIVE DISCOVERY HARD CAPS
- Firecrawl calls: max 5 per memo
- Total Firecrawl spend per memo: max $0.10 USD
- LLM synthesis input: max 30 KB context (auto-truncate longest sources first)
- Stop conditions: ≥3 Tier A/B sources gathered → proceed to §3.8 synthesis
                  <3 Tier A/B sources after 5 calls → fall through to §3.7.7 graceful skip
```

**5-call cascade (each call uses Firecrawl `search` mode where possible to maximize effective sources per call):**

| Call # | Target | Tier | Expected effective sources |
|---|---|---|---|
| 1 | Whalewisdom top 13-F page (`https://whalewisdom.com/stock/{ticker}`) | A | 10-20 institutional holders + position changes |
| 2 | Firecrawl search: `"{ticker}" {sector} analysis 2026 site:stratechery.com OR site:matthewball.co OR site:semianalysis.com OR site:endpts.com OR site:gorozen.com` (filtered to authoritative trade-press whitelist) | B | 3-8 articles |
| 3 | Already-registered firms' tag pages for `{ticker}` (e.g. `https://stratechery.com/?s={ticker}`, `https://a16z.com/?s={ticker}`) — limit to 2 firms most relevant to sector | B | 2-6 firm-specific articles |
| 4 | SEC EDGAR recent filings: `https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK={cik}&type=8-K` (last 6 months) | A | 5-15 8-K events with dates |
| 5 | Yahoo "Wall Street's view" aggregation page (`https://finance.yahoo.com/quote/{ticker}/analyst-insights`) | A | Sell-side PT distribution + recent rating actions |

**Source quality tiers (used for inclusion in framework_blocks):**

```
Tier A (always include if available):
  - SEC filings (13-F, 8-K, S-1, DEF 14A) — legally required disclosure
  - Whalewisdom 13-F aggregations (real money positions, current quarter)
  - Yahoo / Bloomberg consensus analyst PT distributions
  - Bessemer Cloud Index / Sequoia / a16z official annual reports

Tier B (include if Tier A is thin):
  - Established sector trade press whitelist:
    Tech/Software/Communications: Stratechery, Matthew Ball, Sacra, Stratecology
    Tech/Hardware: SemiAnalysis (Dylan Patel), Fabricated Knowledge (Doug O'Laughlin)
    Healthcare: Endpoints News, STAT, Fierce Biotech
    Energy: Goehring & Rozencwajg (gorozen.com), JTC Energy
    Fintech: Fintech Brainfood (Simon Taylor), Andreessen fintech
    Consumer: Trung Phan, Modern Retail, Retail Dive (analytical pieces only)
  - Public investor letters (Pershing Square, Akre Capital, Saga Partners, etc.) when freely accessible

Tier C (include only as supplementary, never as sole source):
  - Investor-Relations company pages (self-reported, conflict of interest)
  - Yahoo / Seeking Alpha Editor's Picks (curated only, NOT random retail authors)

❌ DO NOT cite:
  - Random Substack authors, Reddit, Twitter/X posts (not source-quality)
  - Seeking Alpha retail authors, blog posts without named author
  - LinkedIn posts, Medium articles without firm affiliation
```

**Render in §11 (Live Discovery mode):**

- Add callout at top: "Smart Money via Live Discovery — `{current_sector}` sector empty in registry; cited public artifacts gathered live with [Firecrawl call count]/5 calls within $[Firecrawl spend]/$0.10 cap."
- Render firm cards with sources cited inline (firm name + tier badge + URL + date)
- Run §3.8 synthesis using the live-gathered material; each framework_block cites a specific article URL + date with "Source pulled live `{YYYY-MM-DD}`" disclaimer

**Cost impact:** Live discovery mode cost ≈ Full mode cost + $0.05-0.10 (5 Firecrawl calls) + ~2 min wall-clock. Total Mode A budget: $0.18-0.28 → $0.23-0.38 with live discovery active.

### 3.7.7 Graceful Skip (last resort)

Only reached when: sector 0+0 in registry AND bridge match <3 firms AND live discovery <3 Tier A/B sources.

Skip §11. In §14 Data Gaps, write: "Smart Money §11 skipped after exhausting cascade (Bridge: <3 cross-sector matches; Live Discovery: <3 Tier A/B sources within 5-call / $0.10 budget). Add firms to `~/.claude/skills/stock-research/sector-firms.json` under `'{sector}'` to enable full mode for future runs."

### 3.8 NEW: Smart Money LLM synthesis (5th LLM call, ~$0.05)

Prompt:

```
You are applying 8 investor frameworks to a US public stock under contrarian-drawdown analysis.

Ticker: {TICKER}
Company: {COMPANY_NAME}
Sector: {SECTOR}
Current setup: {DRAWDOWN_PCT}% drawdown from 52W high
AI judgment: Problem type = {PROBLEM_TYPE}, Conviction = {AI_CONVICTION}/10
Thesis summary: {THESIS_SUMMARY}

The 8 firms surveyed for this sector:
{firms_array_with_context}

For 5 firms (selected by you for relevance to this specific ticker — prioritize firms with direct portfolio exposure, recent sector commentary, or framework that makes a strong claim about this name), produce a 2–4 sentence "framework applied" inference. Each must:
- Reference the firm's actual framework or public view
- Apply it to this ticker's specific situation
- End with a position-implication ("would buy", "would pass", "tactical position", etc.)
- NOT claim the firm has actually said anything about this stock unless directly cited from `current_view_excerpt`

Then synthesize:
- "Where they agree" — 4 points (cite "X/8 firms" tally)
- "Where they diverge" — 4 points organized by camp ("Quality compounders X / Brand specialists Y / etc.")
- "Read" — 1–2 sentences on what the agree/diverge pattern implies

CRITICAL: Always end every framework block with "Framework applied — not a direct quote" disclaimer.
DO NOT invent firm quotes. DO NOT misrepresent firm positions. If a firm has no relevant view, omit it from the 5.

Return JSON:
{
  "framework_blocks": [
    { "firm_name": "...", "applied_text": "...", "position_implication": "..." },
    ...5 entries
  ],
  "agree_points": ["4 points with X/8 tallies"],
  "diverge_camps": ["4 points by investor camp"],
  "synthesis_read": "1-2 sentences"
}
```

### 3.9 14-section memo synthesis (1 LLM call, larger context)

See §6 for template. Memo synthesis prompt MUST:
- Generate all 14 sections (was 13 in v2.0)
- §11 Smart Money body comes from §3.8 output, not re-synthesized
- §11 disclaimer present
- §2, §3 Verdict, §6, §11 framework blocks, §13 (Bull/Bear), §10 Red Flags marked `[AI-generated]`
- §12 Entry & Exit explicit on batch sizing
- Use literal `$`, no `\$`
- Total length 3000–4500 words (longer than v2.0 due to §11)

### 3.10 Render HTML

1. Read template: `~/.claude/skills/stock-research/memo-template.html`
2. Replace placeholders (see §7)
3. Write to `~/Documents/stock-research/{ticker}.html` (overwrite if exists)
4. Print absolute path

### 3.11 Write JSON sidecar

See §9 for schema. Includes new `smart_money` block.

---

## §4 Step-by-Step (Mode B: Refresh)

### Single-ticker refresh
User says: `Refresh LULU` or `Refresh LULU after earnings`

1. Read existing JSON (last_run, last price, last conviction, last problem_type, last sector)
2. Fetch deltas: Yahoo (current price), SEC (filings since last_run), news (past 7 days)
3. **Smart Money: read sector cache (NO re-scrape).** If sector hasn't changed, **preserve** existing framework_blocks from previous JSON unless conviction or problem_type changed materially. If changed (>2 conviction points OR problem_type flipped), re-run §3.8 synthesis with new context.
4. Re-run Group 2 (problem-type) and Group 4 (conviction) prompts
5. Generate "Since last run" diff section
6. Re-render full memo with diff at top
7. Update JSON + Notion Watchlist row (if exists)

### Batch refresh
User says: `Refresh my watchlist` or `Weekly refresh`

1. Query Stock Watchlist DB: rows where Position ∈ {Considering, Held, Trim, Replace}
2. For each ticker, run single-ticker refresh
3. Output summary table at end

Cost: ~$0.05 per ticker × N. For 15 tickers, ~$0.75.

---

## §5 Step-by-Step (Mode C: Add to Watchlist)

User says: `Add LULU to my watchlist`

1. Read `~/Documents/stock-research/lulu.json` — must exist (else: "Run `Research LULU` first")
2. Notion data source ID: `e4efdc03-93e6-4834-9b27-ce2960f9df98`
3. Create row:
   - Ticker: `ticker`
   - Conviction: `ai_conviction`
   - Position: `ai_position`
   - Notes: `ai_thesis_summary`
   - Entry Price: blank (user fills manually)
4. Confirm to user

---

## §6 Memo Template — 14 sections

Each section: section chip (number + label) + h2 title + content. Comment button (💬) is added by HTML template's JavaScript on every section, so memo prompt does NOT need to add it.

```markdown
# {Company Name} ({TICKER})

> {one-liner}

[Hero — see §7 for HTML structure]

[IF setup_gate is "✗ Below 50%": WARNING CALLOUT]

[IF mode == refresh: §0 Since Last Run diff section]

## §0 Since Last Run [Mode B only]
*[only present in refresh runs]*

[Diff table: metric / last run / now / Δ for: Price, Drawdown, Conviction, Problem Type]

**What changed:** [3-4 sentences]
**Recommendation change:** [if conviction or position moved]

---

## §1 Snapshot
3-5 bullets: setup, judgment, conviction, position rec, biggest uncertainty.

## §2 The Setup *[AI-generated]*
3-4 paragraphs telling the drawdown story chronologically.
End with: "The market is pricing this as: [X]"

## §3 Short-Term or Structural? *[AI-generated — central question]*
### Bull case (Short-term) [4-5 numbered points]
### Bear case (Structural) [4-5 numbered points]
### Verdict box: Problem Type / Confidence / Biggest uncertainty

## §4 Financial Trajectory
### Revenue table (5+ periods, with YoY growth and segment breakdown)
### Margins table (3+ years: Gross / Operating / FCF margin)
### Balance Sheet: cash, debt, net position, FCF, buybacks

## §5 Valuation
### Current vs 5y history table (P/E TTM, Fwd, EV/EBITDA, EV/Sales, FCF Yield)
### Peers table (3-4 peers)
### Read: priced for structural decline / cyclical recovery / fair?

## §6 Business & Moat *[AI-generated]*
What the company does. Moat components + vulnerabilities. Replicability test.

## §7 Macro & Sector
Sector dynamics, tailwinds, headwinds, regulatory, currency, tariff exposure.

## §8 Capital Structure & Returns
Debt, buybacks, dividend, M&A, capex.
Read: capital structure as tailwind/headwind for thesis.

## §9 Catalysts (Next 12 Months)
3-5 dated catalyst blocks with priority (P0/P1/P2):
- [Title] [Priority] [Date] — [body explaining the catalyst]

## §10 Trading Signals
| Signal | Reading | Tiebreaker |
| Insider activity | ... | +1/0/-1 |
| Institutional flow | ... | +1/0/-1 |
| Short interest | ... | +1/0/-1 |
| Volume pattern | ... | +1/0/-1 |
| Sentiment temperature | ... | +1/0/-1 |
**Aggregate read:** [confirms / contradicts / neutral]

## §11 Smart Money Perspective *[AI-generated — framework inference]*
[NEW in v2.1]

**Disclaimer up front (REQUIRED — render as a styled callout box before the framework blocks):**
> "The source links below point to each institution's **public research pages** — not to specific articles about [TICKER]. The frameworks are **inferred** by applying each firm's known investment thesis (from their published writing) to [TICKER]'s current situation. This is how a sector specialist might think about the stock, not a record of what they've actually said. Always verify before citing. **Frameworks applied — not direct quotes.**"

Source links in firm cards must point to the institution's actual research/insights page (e.g. `a16z.com/games/`, `bvp.com/atlas`). Do NOT fabricate specific article URLs — if no verified article about this ticker exists, link only to the research index page and note "search for [TICKER] on this site."

### The 8 firms surveyed for [SECTOR]
Prose intro: monthly-refreshed cache. Selected for sector authority + public communication.

[Firms grid — 8 cards from sector-firms.json]:
For each firm:
- Tier label ("PRIMARY VC" or "SECONDARY (Public-Market)")
- Firm name
- AUM + sector descriptor
- Framework / thesis (from registry)
- Public communication note (blog / letters / reports)

### How these frameworks read [TICKER]
[5 framework blocks from §3.8 LLM synthesis output]:
For each (5 of 8 firms most relevant to this ticker):
- Firm name (h4)
- 2-4 sentences applying the firm's framework to ticker
- Bold "Applied to {TICKER}:" leading the position-implication sentence
- Disclaimer: "Framework applied — not a direct quote"

### Synthesis
**Where they agree:** [4 bullets, each with X/8 tally]
**Where they diverge:** [4 bullets organized by investor camp]

### Read
1-2 sentences. The agree/diverge pattern itself is informative.

## §12 Entry & Exit Plan *[AI-generated]*
[was §11 in v2.0]
Setup gate met: ✓ / ✗
Position recommendation: Watch / Considering / Avoid

### If Considering:
- Batch sizes: 30-40% / 30-40% / 20-40%
- Add triggers (3-5 specific)
- Final fill triggers

### Exit triggers (any 1):
- Thesis flip
- 60%+ unrealized gain
- Better risk/reward + lowest conviction
- Ticker-specific exits

### What would change conviction
Up scenarios + Down scenarios

## §13 Bull / Bear Case *[AI-generated]*
[was §12 in v2.0]
🐂 Bull: 3-4 numbered arguments
🐻 Bear: 3-4 numbered arguments

## §14 Data Gaps & Sources
[was §13 in v2.0]
### Official Filings
- 10-K, 10-Q, 8-K, DEF 14A, latest earnings transcript

### Market Data
- Yahoo Finance, Whalewisdom, OpenInsider, IR page

### News & Analysis
- 3-5 articles with publication, link, date

### Data Gaps
- Anything unresolved, source conflicts, sector firms registry coverage notes
```

### LLM system prompt for memo synthesis

```
You are writing a contrarian investment memo for a US public stock.

CRITICAL CONSTRAINTS:
- Framework: drawdown-driven contrarian. Setup gate: ≥50% drawdown.
- §3 must classify as Short-term / Structural / Mixed / Unclear with reasoning.
- §11 Smart Money body is provided pre-synthesized by upstream call (do NOT regenerate).
- §11 firm cards come from sector-firms.json (do NOT invent firms).
- §12, §13, §6, §3 verdict, §10 red flags, §11 framework blocks must carry [AI-generated] markers.
- §12: explicit batch entry plan (30-40% / 30-40% / 20-40%) and exit triggers.
- Use literal $ (never \$).
- Total length 3000-4500 words.
- Cite source URLs for material claims.

Structured data: {extracted_json}
Source material: {full_context}
Smart Money output (already synthesized): {smart_money_json}
Sector firms registry slice (8 firms for {sector}): {firms_array}
Mode: {mode}
Previous JSON (refresh only): {prev_json}
```

---

## §7 HTML Template Reference

Skill expects template at `~/.claude/skills/stock-research/memo-template.html`.

### Hero placeholders

| Placeholder | Source |
|---|---|
| `{{TICKER}}` | input |
| `{{COMPANY_NAME}}` | extraction |
| `{{ONE_LINER}}` | extraction |
| `{{CURRENT_PRICE}}` | Yahoo |
| `{{DRAWDOWN_PCT}}` | computed |
| `{{52W_HIGH}}`, `{{52W_LOW}}` | Yahoo |
| `{{PRICE_52W_PERCENTILE}}` | computed: `(price − low) / (high − low) × 100`, display as "Nth %ile 52W" |
| `{{MARKET_CAP_DISPLAY}}` | Yahoo, formatted |
| `{{MARKET_CAP_SIZE}}` | computed: Nano <$300M · Micro $300M–$2B · Small $2B–$10B · Mid $10B–$50B · Large $50B–$200B · Mega >$200B |
| `{{PE_TTM}}` | Yahoo |
| `{{SETUP_GATE_STATUS}}` | computed |
| `{{AI_CONVICTION}}` | LLM 1-10 |
| `{{AI_POSITION}}` | LLM Watch/Considering/Avoid |
| `{{PROBLEM_TYPE}}` | LLM |
| `{{GENERATED_DATE}}` | today |
| `{{MODE_LABEL}}` | "Research" / "Refresh" |
| `{{SECTIONS_CONTENT}}` | rendered §0–§14 HTML |
| `{{MARKDOWN_SOURCE}}` | full markdown for "Copy as Markdown" |

### Visual structure (matches sarea-invest-skill style, stocks-flavored)

**Topnav:**
- Left: `Marisa Ma / Stock Research` (no Confidential badge)
- Right: `⎘ Copy as Markdown` button + `⌥ Export PDF` button (calls `window.print()`)

**Hero:**
- Eyebrow: ticker tag + sector + mode tag (e.g. "LULU · NASDAQ · Consumer Discretionary · RESEARCH MODE")
- H1: company name (navy, serif, 64px)
- One-liner sub
- Horizontal metric bar — 7 cells: Price | Drawdown (red) | Mkt Cap | P/E | AI View | Conviction (purple) | Position
  - Price cell: show `{{CURRENT_PRICE}}` + sub-line `{{PRICE_52W_PERCENTILE}}th %ile 52W` in small muted text
  - Mkt Cap cell: show `{{MARKET_CAP_DISPLAY}}` + sub-line `{{MARKET_CAP_SIZE}}` in small muted text
  - `.metric-value` CSS must NOT use `white-space: nowrap` — AI View label can be multi-word ("Mixed signals", "Short-term dip") and must wrap rather than overflow into adjacent cells
- Below hero bar: one-line data source attribution in 11px mono: "Fundamentals: SEC EDGAR 10-K/10-Q · Market data: Yahoo Finance · Insider: OpenInsider Form 4 · Institutional: Whalewisdom 13-F · News: IR + trade press + earnings call"

**Section structure (each of §0–§14):**
```html
<section class="memo-section" id="section-N">
  <div class="section-chip">
    <span class="section-chip-num">NN</span>
    <span>LABEL</span>
  </div>
  <h2 class="section-title">Title</h2>
  [body]
</section>
```

JavaScript on page load:
1. Wraps each `.section-chip` and a 💬 button in `.section-with-comment-btn` flex row
2. Inserts `.comment-panel` (collapsible textarea) after the wrapper
3. textarea auto-saves to `localStorage[`comment_${ticker}_section-${N}`]` on input
4. Button has class `has-comment` when textarea non-empty (purple highlight)
5. Print stylesheet:
   - Hides `.btn-comment`, `.btn-copy`, `.topnav`, `.toc`
   - Forces `.comment-panel.open` to display (so non-empty comments print)
   - Forces colors with `print-color-adjust: exact` on chips, callouts, firm-cards, framework-blocks

**Footer:**
- Left: "Marisa Ma" (large) + "Stock Research · Contrarian Memo Series" (small)
- Right: Export PDF / Copy Markdown / JSON sidecar links

**Special body components — CSS classes:**

| Component | Class |
|---|---|
| Bullet list w/ em-dash | `<ul class="memo-list">` |
| AI callout | `<div class="callout callout-ai">` |
| Warning callout | `<div class="callout callout-warning">` |
| Verdict box (§3) | `<div class="verdict-box">` w/ verdict-main, verdict-confidence, verdict-uncertainty |
| Bull/bear grid | `<div class="bullbear-grid">` w/ `<div class="case-card bull">` and `<div class="case-card bear">` |
| Trajectory tables | `<table class="trajectory">` |
| Valuation table | `<table class="valuation">` |
| Trading signals table | `<table class="signals">` w/ `.aggregate-read` callout |
| Catalyst block | `<div class="catalyst-block">` w/ catalyst-priority p0/p1/p2 |
| Entry/Exit cards | `<div class="entry-exit-grid">` w/ `<div class="plan-card">`, `.batch-line`, `.trigger-list.add-triggers/.exit-triggers` |
| **Firms grid (§11)** | `<div class="firms-grid">` w/ `<div class="firm-card primary">` or `.secondary` (8 cards total) |
| **Framework blocks (§11)** | `<div class="framework-block">` w/ firm name, framework-text, framework-disclaimer |
| **Synthesis grid (§11)** | `<div class="synthesis-grid">` w/ `.synthesis-card.agree` / `.diverge` |
| Sources grid | `<div class="sources-grid">` w/ `.source-group` |
| Diff table (§0) | `<table class="diff-table">` |

**Section chip mapping:**

| § | Label |
|---|---|
| 1 | SNAPSHOT |
| 2 | THE SETUP |
| 3 | CORE QUESTION |
| 4 | FINANCIAL TRAJECTORY |
| 5 | VALUATION |
| 6 | BUSINESS & MOAT |
| 7 | MACRO & SECTOR |
| 8 | CAPITAL STRUCTURE |
| 9 | CATALYSTS |
| 10 | TRADING SIGNALS |
| 11 | **SMART MONEY** ← new |
| 12 | ENTRY & EXIT |
| 13 | BULL / BEAR CASE |
| 14 | SOURCES |

---

## §8 Notion Watchlist Schema (unchanged from v2.0)

**Page:** https://www.notion.so/611ab11fdaa1412bb64db53e2a3e48a7
**Data source ID:** `e4efdc03-93e6-4834-9b27-ce2960f9df98`

| Field | Type | Notes |
|---|---|---|
| Ticker | title | Uppercase |
| Conviction | number | 1-10 |
| Position | select | Watch / Considering / Held / Trim / Replace / Exited / Avoid |
| Entry Price | number ($) | User-only field |
| Notes | rich_text | One-line judgment |
| Last Updated | last_edited_time | Auto |

Skill rules:
- Mode A: never writes Watchlist
- Mode B: only updates existing rows
- Mode C: only creates new rows
- Skill never writes Entry Price

---

## §9 JSON Sidecar Format

```json
{
  "schema_version": "2.1",
  "ticker": "LULU",
  "company_name": "Lululemon Athletica Inc.",
  "cik": "0001397187",
  "last_run": "2026-04-25T14:32:00Z",
  "mode_last_run": "research",

  "market_snapshot": {
    "price": 215.30,
    "52w_high": 449.80,
    "52w_low": 198.50,
    "drawdown_pct": 0.521,
    "market_cap_usd": 26200000000,
    "pe_ttm": 11.2,
    "pe_forward": 13.5,
    "as_of": "2026-04-25"
  },

  "setup_gate": "eligible",
  "sector": "Consumer Discretionary",

  "ai_judgment": {
    "problem_type": "Short-term",
    "confidence": "Medium",
    "thesis_summary": "...",
    "biggest_uncertainty": "..."
  },

  "ai_recommendation": {
    "conviction": 7,
    "position": "Considering",
    "first_batch_size_pct": 35,
    "add_triggers": [...],
    "exit_triggers": [...]
  },

  "smart_money": {
    "sector_used": "Consumer Discretionary",
    "firms_used": [
      {"name": "L Catterton", "tier": "primary", "slug": "l-catterton"},
      {"name": "Forerunner Ventures", "tier": "primary", "slug": "forerunner"},
      ...8 entries
    ],
    "cache_age_max_days": 12,
    "framework_blocks_synthesized": 5,
    "agree_count": 4,
    "diverge_count": 4,
    "synthesis_read": "Smart money would split. Quality compounders see setup; brand specialists see structural risk..."
  },

  "key_dates": {
    "next_earnings": "2026-06-04",
    "last_10k": "2026-03-21",
    "last_10q": "2026-03-21"
  },

  "sources": {
    "10k": "https://...",
    "10q_latest": "https://...",
    "ir_page": "https://...",
    "earnings_transcript_latest": "https://..."
  }
}
```

---

## §10 Source Priority (unchanged from v2.0)

| Field | Priority |
|---|---|
| Revenue / EPS / Margins | SEC EDGAR XBRL > 10-Q narrative > earnings press release > Yahoo |
| Current price / market cap | Yahoo only |
| Insider transactions | OpenInsider > SEC Form 4 |
| Institutional holdings | Whalewisdom > SEC 13-F |
| News | WSJ/Bloomberg/FT > Reuters/Axios > Seeking Alpha > smaller blogs |
| Earnings transcripts | Company IR > Motley Fool / SA mirrors |
| **Smart Money firm view** | **firm's own blog (cached) > our framework_descriptor in registry** |

When SEC and news disagree on financials: SEC wins, note discrepancy.

---

## §10.5 Observability (NEW in v3.0)

### Cost & run log

After every run (including pre-check), append one row to `~/.claude/skills/stock-research/runs.csv`:

```
date,ticker,mode,status,input_tokens,output_tokens,estimated_cost_usd,error_note
2026-06-29,RBLX,research,success,42100,8300,0.28,
2026-06-29,HIMS,precheck,success,3200,610,0.02,
2026-06-29,TGT,research,failed,18400,0,0.09,Yahoo Finance 403 at step 3
```

**Estimated cost formula:**
- Claude Sonnet 4.6: $3/M input tokens + $15/M output tokens
- Log cumulative tokens per run by summing across all LLM calls
- Round to 2 decimal places

### Data quality flags

Before passing data to any LLM call, validate these conditions. If any fail, surface a visible warning in the pre-check output AND in §14 Data Gaps:

| Check | Pass condition | Flag text |
|---|---|---|
| SEC filings current | Most recent 10-Q or 10-K filed within 6 months | "SEC filings stale — last filing {date}" |
| Revenue data present | At least 4 quarters of revenue in XBRL | "Insufficient revenue history — only {N} quarters available" |
| Price data reasonable | 52W high > current price > $0.50 | "Suspicious price data — verify manually" |
| Market cap present | marketCap field non-null and > 0 | "Market cap unavailable — Yahoo data incomplete" |
| Ticker resolves to public company | CIK found in SEC EDGAR | "Ticker not found in SEC registry — private company or delisted?" |

### Input validation (pre-research, always runs)

1. Strip whitespace and `$` prefix. Uppercase. Must match `^[A-Z]{1,5}$` — else stop and ask.
2. Resolve CIK from `https://www.sec.gov/files/company_tickers.json`. If not found → stop, report.
3. Check that `quoteType` from Yahoo is `EQUITY` (not ETF, mutual fund, crypto). If not → warn and ask user to confirm.

---

## §11 Error Handling

| Failure | Response |
|---|---|
| Ticker not resolved | Stop, ask user to clarify |
| Ticker is private company / not in SEC registry | Stop: "This tool covers US public companies only." |
| SEC EDGAR rate-limited | Retry with 30s backoff. After 3 fails, proceed Yahoo-only and flag in §14 |
| Yahoo Finance fails | Try alternative actor, then Firecrawl direct |
| Drawdown metric unavailable | Note in hero, proceed |
| **Sector partially populated (3+1 to 4+2)** | **Run with available firms. Flag coverage gap in §14 Data Gaps.** |
| **Sector empty (0+0) — cascade level 1** | **Bridge Mode: search adjacent registered sectors for keyword-relevant firms (see §3.7.5). If ≥3 match → render with "BRIDGED · `{source_sector}`" tier label. Cost ≈ Full mode.** |
| **Sector empty + bridge fails (<3 matches) — cascade level 2** | **Live Discovery: Firecrawl 5 authoritative public sources with hard caps (max $0.10 spend, max 5 calls; see §3.7.6). If ≥3 Tier A/B sources gathered → render with "LIVE · `{date}`" labels and source URLs cited inline.** |
| **Sector empty + bridge fails + live discovery <3 sources — cascade level 3 (last resort)** | **Graceful Skip §11 per §3.7.7. §14 Data Gaps notes cascade exhaustion + suggests registry population.** |
| **All 8 firm scrapes fail** | **Degrade §11: render firm cards using only `framework_descriptor` from registry, no fresh excerpts. Log "Cached frameworks only" warning** |
| **Smart Money synthesis LLM call fails** | **Retry once. If still fails, render only firm cards (8 cards) and skip the "How these frameworks read X" + Synthesis subsections. Note in §14** |
| **Live Discovery Firecrawl call fails** | **Skip that call (don't retry within memo). If <3 calls succeed total → fall through to graceful skip.** |
| **Live Discovery $0.10 spend cap hit before 5 calls complete** | **Stop calling Firecrawl. Synthesize §11 with whatever was gathered if ≥3 sources; else fall through.** |
| LLM API error (other groups) | Retry 3x. Persistent → stop, report |
| HTML write fails | Print attempted path + rendered HTML to stdout |
| Notion write fails (Mode B/C) | Continue. Log warning. HTML/JSON are source of truth |

---

## §12 Test Cases

### Primary test: full research run

Pick any mid-cap or larger US public ticker. Validate after run:
- [ ] Pre-check output shown before full research begins
- [ ] `runs.csv` has a new row with correct ticker, tokens, cost estimate
- [ ] HTML hero shows metric bar (Price / 52W Drawdown / Mkt Cap / P/E / AI Conviction / AI View / Sentiment)
- [ ] No "Setup Gate" cell (removed in v3.0)
- [ ] §1 Snapshot has AI View paragraph (not a trading instruction)
- [ ] **§11 Smart Money has 8 firm cards (5 primary + 3 secondary)**
- [ ] **5 framework blocks present, each with disclaimer**
- [ ] **Synthesis (agree/diverge) present**
- [ ] **No invented firm quotes (random spot-check)**
- [ ] AI Conviction visible in hero
- [ ] §10 trading signals populated
- [ ] Each section has 💬 button
- [ ] Clicking a 💬 button reveals textarea; typing saves to localStorage
- [ ] `⌥ Export PDF` button in topnav works
- [ ] No `\$` anywhere; only literal `$`
- [ ] JSON sidecar has `smart_money` block with `firms_used` (8 entries)
- [ ] Data quality flags in §14 if any checks failed

### Pre-check abort test

Research a ticker, say "no" at pre-check. Validate:
- [ ] Full research does NOT run
- [ ] `runs.csv` has a row with `mode=precheck, status=aborted`
- [ ] Cost logged as ~$0.02

### Refresh test (Mode B)

After first run, wait ≥1 day, then `Refresh [TICKER]`. Validate:
- [ ] HTML has "Since last run" diff at top
- [ ] §11 Smart Money preserved (NOT re-synthesized) if conviction unchanged
- [ ] If conviction moved ≥2 points, §11 re-synthesized
- [ ] JSON `last_run` updated
- [ ] `runs.csv` updated with refresh row

### Negative test: sector not in registry

Force sector to one not in registry (e.g. user-defined "Crypto/Web3" before registry is populated). Validate:
- [ ] §11 absent or shows registry-miss message
- [ ] §14 Data Gaps notes the missing sector
- [ ] Memo still generates without §11

---

## §13 Roadmap

### v2.2 (next)
- **Comment export to Markdown.** "Copy as Markdown" button includes any non-empty comments inline as `> [my note: ...]` quotes
- Earnings calendar integration (skill flags upcoming earnings)
- Auto-detect 8-K filings on watchlist tickers and trigger event-based refresh

### v2.3
- Sector firms registry pulled from public source (so users don't maintain manually)
- Per-firm signal weight (configurable in registry)
- Deeper firm scrape: not just blog, but recent letters / podcasts

### v3.0 — Vercel public deployment
- Same workflow logic, wrapped as Vercel serverless function
- HTML-only output (no Notion sync — that's user-private)
- Vercel KV cache (7-day TTL on memo, 30-day TTL on firms)
- Public URL: any user enters ticker → memo
- No accounts, no payment (free public utility)
- ~80% of skill code reused

---

## §14 NEW: Sector → Firms Registry

### File location
`~/.claude/skills/stock-research/sector-firms.json`

### Schema
```json
{
  "schema_version": "1.0",
  "last_updated": "2026-04-25",
  "sectors": {
    "Consumer Discretionary": {
      "primary": [
        {
          "name": "L Catterton",
          "slug": "l-catterton",
          "aum_usd_billion": 34,
          "tier_descriptor": "Consumer brand specialist",
          "framework_descriptor": "Premium athleisure / DTC consumer brand. Active in Vuori, Honest Company, Peloton, Tarte. Thesis: brand advantage durable but requires reinvention every 5-7 years.",
          "blog_url": "https://www.lcatterton.com/insights",
          "letters_url": null,
          "public_communication_note": "Direct competitor exposure · sector reports public"
        },
        {
          "name": "Forerunner Ventures",
          "slug": "forerunner",
          "aum_usd_billion": 2,
          "tier_descriptor": "Consumer DTC pioneer",
          "framework_descriptor": "Glossier, Bonobos, Warby Parker, Hims early-stage. Thesis: brand-as-distribution structurally won by specialists, not legacy incumbents.",
          "blog_url": "https://forerunnerventures.com/blog",
          "letters_url": null,
          "public_communication_note": "Public blog · regular thought-leadership posts"
        },
        {
          "name": "Lerer Hippeau",
          "slug": "lerer-hippeau",
          "aum_usd_billion": 1,
          "tier_descriptor": "Lifestyle & commerce",
          "framework_descriptor": "Allbirds, Casper, Warby Parker. View: brand fatigue real but slower than market prices in.",
          "blog_url": "https://www.lererhippeau.com/articles",
          "letters_url": null,
          "public_communication_note": "Newsletter on consumer cycles"
        },
        {
          "name": "Verlinvest",
          "slug": "verlinvest",
          "aum_usd_billion": 5,
          "tier_descriptor": "Cross-stage consumer",
          "framework_descriptor": "Vita Coco, Hims, Tony's Chocolonely, Oatly. Hybrid public + private. Believes premium pricing power = compounding moat when category healthy.",
          "blog_url": "https://www.verlinvest.com/news",
          "letters_url": null,
          "public_communication_note": "Sector white papers · quarterly thesis posts"
        },
        {
          "name": "Maveron",
          "slug": "maveron",
          "aum_usd_billion": 1.5,
          "tier_descriptor": "Consumer specialist",
          "framework_descriptor": "Allbirds, Trupanion, eBay (early). Howard Schultz-affiliated. Lens: consumer trust + repeat purchase > one-time hype.",
          "blog_url": "https://maveron.com/news",
          "letters_url": null,
          "public_communication_note": "Periodic sector framework essays"
        }
      ],
      "secondary": [
        {
          "name": "Pershing Square",
          "slug": "pershing-square",
          "aum_usd_billion": 15,
          "tier_descriptor": "Activist long-only",
          "framework_descriptor": "Bill Ackman. Quality consumer at distress: Chipotle, Restaurant Brands, Howard Hughes. Framework: simple model + cash-generative + activist lever + brand moat.",
          "blog_url": null,
          "letters_url": "https://pershingsquareholdings.com/letters/",
          "public_communication_note": "Quarterly investor letters publicly available"
        },
        {
          "name": "Akre Capital",
          "slug": "akre",
          "aum_usd_billion": 15,
          "tier_descriptor": "Quality compounders",
          "framework_descriptor": "Long-term holds in Mastercard, Constellation Software, Moody's. 'Three-legged stool' framework: business + management + reinvestment runway.",
          "blog_url": "https://www.akrecapital.com/insights/",
          "letters_url": null,
          "public_communication_note": "Thought-piece essays on quality & compounding"
        },
        {
          "name": "Polen Capital",
          "slug": "polen",
          "aum_usd_billion": 50,
          "tier_descriptor": "Quality growth",
          "framework_descriptor": "Concentrated quality growth book. Has historically held LULU. Framework: high ROIC + brand moat + secular tailwind.",
          "blog_url": "https://www.polencapital.com/insights",
          "letters_url": null,
          "public_communication_note": "Quarterly commentary · sector deep dives"
        }
      ]
    },

    "Tech/Software": {
      "primary": [
        {
          "name": "Bessemer Venture Partners",
          "slug": "bessemer",
          "aum_usd_billion": 20,
          "tier_descriptor": "Cloud / SaaS thought leader",
          "framework_descriptor": "Cloud Index publisher, State of the Cloud reports, deep public-comp visibility. Frameworks on NDR, magic number, rule of 40.",
          "blog_url": "https://www.bvp.com/atlas",
          "letters_url": null,
          "public_communication_note": "Industry-leading SaaS analytics published quarterly"
        },
        {
          "name": "a16z",
          "slug": "a16z",
          "aum_usd_billion": 45,
          "tier_descriptor": "Generalist with deep tech bench",
          "framework_descriptor": "Marc Andreessen / partners. Thesis-driven essays on infrastructure, AI, dev tools.",
          "blog_url": "https://a16z.com/articles/",
          "letters_url": null,
          "public_communication_note": "Frequent essays + podcast"
        },
        {
          "name": "Index Ventures",
          "slug": "index",
          "aum_usd_billion": 15,
          "tier_descriptor": "Enterprise SaaS specialist",
          "framework_descriptor": "Stripe, Slack, Datadog, Confluent early. Frameworks on bottom-up adoption, developer tools.",
          "blog_url": "https://www.indexventures.com/perspectives",
          "letters_url": null,
          "public_communication_note": "Periodic perspective articles"
        },
        {
          "name": "ICONIQ Growth",
          "slug": "iconiq",
          "aum_usd_billion": 80,
          "tier_descriptor": "Growth-stage software",
          "framework_descriptor": "Growth-stage SaaS / enterprise. Publishes benchmark reports.",
          "blog_url": "https://www.iconiqcapital.com/growth/insights",
          "letters_url": null,
          "public_communication_note": "Quarterly benchmark reports on SaaS metrics"
        },
        {
          "name": "Battery Ventures",
          "slug": "battery",
          "aum_usd_billion": 13,
          "tier_descriptor": "B2B / vertical software",
          "framework_descriptor": "Vertical SaaS, infrastructure, dev tools. Multi-stage approach.",
          "blog_url": "https://www.battery.com/insights",
          "letters_url": null,
          "public_communication_note": "Insights blog with sector deep-dives"
        }
      ],
      "secondary": [
        {
          "name": "Coatue Management",
          "slug": "coatue",
          "aum_usd_billion": 70,
          "tier_descriptor": "Crossover hedge fund",
          "framework_descriptor": "Public + private tech. Quarterly market reports. Thesis-led, growth-quality balance.",
          "blog_url": null,
          "letters_url": "https://www.coatue.com/research",
          "public_communication_note": "Public-facing research occasionally"
        },
        {
          "name": "Lone Pine Capital",
          "slug": "lone-pine",
          "aum_usd_billion": 18,
          "tier_descriptor": "Quality growth long/short",
          "framework_descriptor": "Steve Mandel / successors. Quality growth focus. Holdings include Microsoft, Meta, Amazon at various points.",
          "blog_url": null,
          "letters_url": null,
          "public_communication_note": "Limited public commentary; positions visible via 13-F"
        },
        {
          "name": "Stratechery (Ben Thompson)",
          "slug": "stratechery",
          "aum_usd_billion": null,
          "tier_descriptor": "Independent analyst",
          "framework_descriptor": "Aggregation theory, platform economics. Most-read public tech analyst in the industry.",
          "blog_url": "https://stratechery.com",
          "letters_url": null,
          "public_communication_note": "Daily updates; publicly indispensable for tech-PM thinking"
        }
      ]
    },

    "Tech/Hardware": {
      "primary": [
        {"name": "Khosla Ventures", "slug": "khosla", "framework_descriptor": "Deep tech / chips / climate. Vinod Khosla.", "blog_url": "https://www.khoslaventures.com/insights/", "tier_descriptor": "Deep tech / hardware"},
        {"name": "Lux Capital", "slug": "lux", "framework_descriptor": "Frontier tech, semis, robotics, defense.", "blog_url": "https://www.luxcapital.com/news/", "tier_descriptor": "Frontier hardware"},
        {"name": "Founders Fund", "slug": "founders-fund", "framework_descriptor": "Peter Thiel-affiliated. Hardware (Anduril, Palantir).", "blog_url": "https://foundersfund.com/the-prepared-mind/", "tier_descriptor": "Hardware-friendly generalist"}
      ],
      "secondary": [
        {"name": "SemiAnalysis (Dylan Patel)", "slug": "semianalysis", "framework_descriptor": "Independent semis analyst. Datacenter, GPU, AI infra deep technical.", "blog_url": "https://www.semianalysis.com", "tier_descriptor": "Independent semis analyst"},
        {"name": "Fabricated Knowledge", "slug": "fabricated-knowledge", "framework_descriptor": "Doug O'Laughlin. Semis, hardware capex cycles.", "blog_url": "https://www.fabricatedknowledge.com", "tier_descriptor": "Independent semis analyst"}
      ],
      "_note": "Hardware sector has fewer 5+3. Skill should accept partial registries (3+2 minimum)."
    },

    "Healthcare/Pharma": {
      "primary": [
        {"name": "RA Capital", "slug": "ra-capital", "framework_descriptor": "Biotech specialist crossover.", "blog_url": null, "letters_url": null, "tier_descriptor": "Biotech crossover"},
        {"name": "OrbiMed", "slug": "orbimed", "framework_descriptor": "Healthcare-only. Public + private.", "blog_url": "https://www.orbimed.com/news/", "tier_descriptor": "Healthcare specialist"},
        {"name": "Bessemer Healthcare", "slug": "bessemer-health", "framework_descriptor": "Bessemer's healthcare arm. Healthcare Atlas reports.", "blog_url": "https://www.bvp.com/atlas/healthcare", "tier_descriptor": "Healthcare thought leader"}
      ],
      "secondary": [
        {"name": "Endpoints News", "slug": "endpoints", "framework_descriptor": "Industry trade publication with strong analytical voice.", "blog_url": "https://endpts.com", "tier_descriptor": "Industry analyst"}
      ]
    },

    "Fintech": {
      "primary": [
        {"name": "Ribbit Capital", "slug": "ribbit", "framework_descriptor": "Fintech specialist. Robinhood, Nubank, Coinbase early.", "blog_url": null, "tier_descriptor": "Fintech specialist"},
        {"name": "QED Investors", "slug": "qed", "framework_descriptor": "Fintech-only. Frank Rotman.", "blog_url": "https://qedinvestors.com/insights/", "tier_descriptor": "Fintech specialist"}
      ],
      "secondary": [
        {"name": "Fintech Brainfood (Simon Taylor)", "slug": "fintech-brainfood", "framework_descriptor": "Independent weekly fintech analysis.", "blog_url": "https://fintechbrainfood.com", "tier_descriptor": "Independent fintech analyst"}
      ]
    },

    "Energy": {
      "primary": [],
      "secondary": [
        {"name": "Goehring & Rozencwajg", "slug": "gnr", "framework_descriptor": "Natural resources / energy specialists. High-quality public commentary.", "blog_url": "https://gorozen.com/research/", "tier_descriptor": "Energy + resources specialist"},
        {"name": "Andurand Capital", "slug": "andurand", "framework_descriptor": "Pierre Andurand. Public + private commentary on oil cycles.", "blog_url": null, "tier_descriptor": "Commodity/energy hedge fund"}
      ],
      "_note": "Energy has very few public-facing VCs. Use 0 primary + 2-3 secondary for now."
    },

    "Healthcare/Devices": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Healthcare/Services": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Consumer Staples": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Financials": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Materials": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Industrials": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Real Estate": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Utilities": { "primary": [], "secondary": [], "_note": "Populate as encountered." },
    "Communications": { "primary": [], "secondary": [], "_note": "Populate as encountered." }
  }
}
```

### Maintenance philosophy
- **Don't over-build.** Sectors only need population when you encounter a stock in them.
- **Quality over completeness.** Better to have 3 high-signal firms than 8 noise firms.
- **Public communication is the bar.** A firm with no blog / letters / podcast is not in the registry, even if great.
- **Refresh quarterly.** When a firm's blog has been silent for 6+ months, demote or replace.

---

## §15 NEW: Cache Maintenance

### Cache structure
```
~/.claude/skills/stock-research/cache/firms/
  ├── l-catterton.json
  ├── forerunner.json
  ├── lerer-hippeau.json
  ├── verlinvest.json
  ├── maveron.json
  ├── pershing-square.json
  ├── akre.json
  ├── polen.json
  ├── bessemer.json
  ├── a16z.json
  └── ... (one per firm in registry)
```

### Cache file schema
```json
{
  "firm_slug": "l-catterton",
  "scraped_at": "2026-04-15T10:00:00Z",
  "expires_at": "2026-05-15T10:00:00Z",
  "framework_summary": "Their public 2025 commentary frames premium athleisure as a category where heritage brands face share erosion from specialist newcomers...",
  "latest_blog_post_url": "https://www.lcatterton.com/insights/2026/04/athleisure-q1",
  "latest_blog_post_title": "Q1 2026: The Premium Athleisure Tipping Point",
  "latest_blog_post_excerpt": "...key points from the post...",
  "all_recent_posts": [
    {"title": "...", "url": "...", "date": "..."},
    ... 5 most recent
  ],
  "scrape_status": "success",
  "scrape_errors": null
}
```

### Refresh commands

**Manual single-firm refresh:**
```
Refresh sector firms cache for L Catterton
```
→ Skill scrapes firm.blog_url, summarizes, updates cache file. Cost ~$0.005.

**Manual sector refresh:**
```
Refresh sector firms cache for Consumer Discretionary
```
→ Refreshes all 8 firms in the sector. Cost ~$0.04.

**Manual full refresh:**
```
Refresh all sector firms cache
```
→ Refreshes everything in registry. Cost varies by populated sectors. Can take 5-15 min.

**Auto-refresh:** During Mode A, if a firm cache is stale (`expires_at` past), scrape that firm before Smart Money synthesis. This means the user rarely needs to run manual refresh — natural usage keeps cache fresh.

### Cache miss policies
- Stale (>30 days): silently refresh during Mode A. User doesn't see anything.
- Missing entirely: scrape on first encounter. Note in log.
- Scrape failure: use `framework_descriptor` from registry only. Note "Cached frameworks only" in framework block.
- Persistent scrape failure (3 attempts): mark firm as `scrape_status: "broken"` in cache. Skill skips the firm for next 7 days.

---

## §16 Test sequence (recommended for first-time setup)

1. **Verify file structure:**
   ```
   ~/.claude/skills/stock-research/
   ├── SKILL.md (this spec)
   ├── memo-template.html (HTML template, derived from lulu-stock-research-demo.html)
   ├── sector-firms.json (registry)
   └── cache/
       └── firms/ (auto-created on first run)
   ```

2. **Run a Consumer Discretionary ticker** (sector fully populated):
   ```
   Research [TICKER] using the stock-research skill
   ```
   Validate against §12 checklist.

3. **Run a Tech/Software ticker** (sector fully populated):
   Same validation.

4. **Run a sector NOT in registry** (e.g. Utilities):
   - Validate Smart Money skipped gracefully
   - Validate §14 Data Gaps notes the missing sector

5. **Open generated HTML, test:**
   - Click 💬 on a section → write text → reload page → comment persists ✓
   - Click `⌥ Export PDF` → preview shows comments included ✓
   - Verify branding: "Marisa Ma / Stock Research" ✓
   - Verify NO Confidential badge ✓

6. **Run Mode B refresh** on the same ticker after ≥1 day:
   ```
   Refresh [TICKER]
   ```
   Validate Smart Money preserved (not re-synthesized).

7. **Add to Watchlist:**
   ```
   Add [TICKER] to my watchlist
   ```
   Verify Notion row created with AI Conviction + Position.

---

## End of v2.1 spec

To install as a Claude Code skill:
1. Save this file as `~/.claude/skills/stock-research/SKILL.md`
2. Save the HTML template (derived from `lulu-stock-research-demo.html`) as `~/.claude/skills/stock-research/memo-template.html`
3. Save the registry JSON as `~/.claude/skills/stock-research/sector-firms.json` (copy from §14)
4. Restart Claude Code
5. Run: `Research [TICKER] using the stock-research skill`
