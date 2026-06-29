# Stock Research — Claude Code Skill (v3.0)

A Claude Code skill that generates investor-grade research memos for US public stocks. Pulls live data from SEC EDGAR, Yahoo Finance, insider filings, and news — synthesized via a Smart Money lens that applies 8 sector-focused institutional frameworks to each ticker.

## What it does

1. **Validates input** — resolves ticker to SEC CIK, confirms public company
2. **Pulls SEC EDGAR data** — 8 quarters of XBRL financials (revenue, net income, FCF, balance sheet)
3. **Fetches market data** — price, 52W range, market cap, P/E, short interest via Yahoo Finance
4. **Scrapes insider signals** — Form 4 filings via OpenInsider, 13-F aggregations via Whalewisdom
5. **Reads the news** — 5–8 articles prioritized from WSJ, Bloomberg, FT, earnings call transcripts
6. **Smart Money analysis** — applies 8 sector-specific investor frameworks (5 primary VC + 3 public-market) to the ticker; synthesizes where they agree and diverge
7. **Synthesizes 14-section memo** — hypothesis, financials, valuation, moat, entry/exit, bull/bear cases
8. **Outputs HTML + JSON** — `~/Documents/stock-research/{ticker}.html` with comment system and PDF export

**Cost:** ~$0.28 per full memo · **Time:** ~7–9 minutes

## Smart Money (v3.0 centerpiece)

The §11 Smart Money section is the tool's primary differentiator. For each sector, a curated registry (`sector-firms.json`) maps 8 institutional investors and analysts — 5 primary (VC / growth investors) + 3 secondary (public-market specialists). Their frameworks are applied to the specific ticker:

- What would this investor think about this setup?
- Does the company fit their investment criteria?
- Where do the 8 frameworks agree vs. diverge?

Frameworks are **inferred from public writing** — blog posts, letters, annual reports — not invented quotes. Every block ends with a disclaimer.

## Sample memos

Generated with this skill (v3.0 format):

| Ticker | Company | Sector | Drawdown | AI Conviction |
|--------|---------|--------|----------|---------------|
| [RBLX](memos/rblx.html) | Roblox | Tech / Consumer Platform | −62.8% | 7/10 |
| [HIMS](memos/hims.html) | Hims & Hers Health | Healthcare / Consumer | −46.2% | 6/10 |
| [NKE](memos/nke.html) | Nike | Consumer Discretionary | −54.3% | 5/10 |

## File structure

```
SKILL.md              ← Claude Code skill spec (v3.0)
memo-template.html    ← HTML output template with {{PLACEHOLDERS}}
sector-firms.json     ← Sector → 8-firm registry with blog URLs
memos/
  rblx.html           ← Generated memo: Roblox
  hims.html           ← Generated memo: Hims & Hers
  nke.html            ← Generated memo: Nike
```

## How to install

1. Clone this repo
2. Copy `SKILL.md`, `memo-template.html`, and `sector-firms.json` into `~/.claude/skills/stock-research/`
3. In Claude Code, run: `/stock-research` then type a ticker

Requires Claude Code with an Anthropic API key. The skill also uses Apify (Yahoo Finance scraper, ~$0.0005/ticker) and Firecrawl (news scraping, ~$0.05 per run) — both are optional; the skill degrades gracefully if unavailable.

## v3.0 vs v2.1

| | v2.1 | v3.0 (this repo) |
|--|------|-----------------|
| Entry gate | ≥50% drawdown required | No hard gate — any US public stock |
| Pre-check | None | Fast 30s preliminary check (~$0.02) before committing |
| Hero metric | Setup Gate (✓/✗) | AI View label |
| Observability | None | `runs.csv` cost log + 5 data quality checks |
| Smart Money | Full §11 | Full §11 (unchanged) |

## Tech stack

- **Claude Code** — skill orchestration, LLM calls (Claude Sonnet 4.6)
- **SEC EDGAR XBRL API** — direct HTTP, no API key required
- **Apify** — Yahoo Finance scraper actor
- **Firecrawl** — news and IR page scraping

---

Built by [Marisa Ma](https://marisama.xyz) · [Website showcase](https://marisama.xyz/tools/stock-research)
