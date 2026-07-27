# jjmow-claude-plugins

Personal Claude Code / Cowork plugin marketplace.

| Plugin | What it does |
|--------|--------------|
| [`auto-coverage`](plugins/auto-coverage) | One-input equity research orchestrator — routes a ticker, a holdings list, or a sector/theme through the Anthropic FSI equity-research, market-researcher, earnings-reviewer, and model-builder plugins, end to end, without pausing between tasks |

---

## Install

### 1. Install the four dependency plugins first

`auto-coverage` orchestrates other plugins — it contains no research logic of its own and
**will not function** unless all four are installed. Claude Code does not hard-enforce
cross-plugin dependencies, so this step is on you.

```bash
claude plugin marketplace add anthropics/financial-services
```

```bash
claude plugin install equity-research@claude-for-financial-services
```

```bash
claude plugin install market-researcher@claude-for-financial-services
```

```bash
claude plugin install earnings-reviewer@claude-for-financial-services
```

```bash
claude plugin install model-builder@claude-for-financial-services
```

### 2. Install auto-coverage

```bash
claude plugin marketplace add egger-meow/jjmow-claude-plugins
```

```bash
claude plugin install auto-coverage@jjmow-claude-plugins
```

### 3. Data sources — nothing to set up

**No paid terminal, no API key, no connector is required to start running.** Taiwan-listed
tickers are the primary use case; US tickers are supported via SEC EDGAR.

**Taiwan chain** (`.TW` / `.TWO`, or any TWSE/TPEx-listed name):

| Tier | Source | Feeds |
|------|--------|-------|
| 0 | `twse-mcp` connector — **optional**, see below | Quotes, financials, ESG via TWSE + TPEx + TAIFEX |
| 1 | Official keyless OpenAPI — `openapi.twse.com.tw/v1/` (上市) and the TPEx equivalents (上櫃) | Material announcements (重大訊息), monthly revenue (月營收), 5%+ shareholder filings, insider holdings |
| 2 | MOPS direct — `mops.twse.com.tw` | Full financial statements and annual reports — the primary feed for financial modeling |
| 3 | `mis.twse.com.tw` stock info endpoint | Current and historical prices (degrades to web search if unreachable) |
| 4 | Company IR pages + web search | Business description, management, competitive landscape |

**US chain** (non-Taiwan tickers): SEC EDGAR filings and the XBRL financial data API for
financials, company IR pages and web search for qualitative context, public feeds for
price history.

Tier 1 is rate-capped at **3 requests per 5 seconds**, which matters most in holdings mode
— `auto-coverage` paces its calls across the whole loop rather than per ticker.

**No fabricated numbers.** If a figure can't be obtained through any tier, it is marked
`[UNSOURCED]` inline and listed in a Data Gaps note at the end of that task's deliverable —
never estimated, inferred from a comparable, or filled in from memory. A task halts only
when *every* applicable tier fails to produce a required input, not merely because an
optional connector is absent.

### Optional: Taiwan market MCP connector

`plugins/auto-coverage/.mcp.json` ships a config for `twse-mcp`
([TWSEMCPServer](https://github.com/twjackysu/TWSEMCPServer)), which exposes TWSE, TPEx,
and TAIFEX data through one interface as Tier 0.

- **This is an upgrade, not a prerequisite.** Tiers 1–4 work with zero setup, and
  `auto-coverage` drops silently to Tier 1 when the connector isn't reachable.
- **To enable:** nothing further is needed once `.mcp.json` is present and Claude Code /
  Cowork picks it up. Confirm with `/mcp` that `twse-mcp` shows as connected.
- **Fair-use cap:** the public hosted endpoint is rate-limited, not unlimited. For heavy or
  commercial use, self-host instead:

```bash
docker run -i --rm --pull=always -e MCP_STDIO=1 ghcr.io/twjackysu/twsemcpserver:latest
```

---

## Usage

```bash
/auto-coverage TSM
```

The command takes one argument and detects what to do with it.

| Input | Mode | What runs |
|-------|------|-----------|
| `TSM`, `2330.TW`, `Nvidia` | **ticker** | `initiating-coverage` Tasks 1–5, end to end |
| `TSM, ASML, 2330.TW` or `./portfolio.csv` | **holdings** | Per-ticker triage, sequentially |
| `"AI power infrastructure"`, `"semis"`, empty | **explore** | `market-researcher` sweep → top 3 candidates → full pipeline each |

Examples:

```bash
/auto-coverage 2330.TW, ASML, NVDA
```

```bash
/auto-coverage AI power infrastructure
```

### Output language

**Default: Traditional Chinese (繁體中文).** Every prose deliverable — research narrative,
valuation write-up, note text, DOCX report body — is written in Traditional Chinese unless
told otherwise. File names, Excel tab names, and market-convention abbreviations (`EPS`,
`EBITDA`, tickers) stay in English regardless, since the pipeline matches on them literally.

```bash
/auto-coverage TSM lang=en
```

### Pipeline depth — stop early

A full 5-task run is slow. Cap it with `depth=N`, `through Task N`, or a target deliverable
name (`03_Valuation_Analysis`, `估值`) — Chinese phrasing like `到第三輪就好` works too.
Default is 5 (the full pipeline). Every task up to N still runs to full spec; only the
tasks after N are skipped, and the run reports a clean "stopped by request," not a failure.

```bash
/auto-coverage 2330.TW depth=3
```

```bash
/auto-coverage 2408.TW 到第三輪就好
```

Depth applies to every Mode 1 run in the session — a direct ticker, a no-prior-coverage
ticker inside a holdings batch, or a candidate deep dive inside exploration. It has no
effect on the earnings-reviewer or thesis-tracker paths, which aren't the 5-task pipeline.

### Holdings triage

Each ticker in the list is routed by its own situation, sequentially — never in parallel:

- **Reports within 30 days, or just reported** → `earnings-reviewer` (transcript + filings
  → model update → note draft). Faster and more relevant than a fresh initiation.
- **No near-term earnings, no prior report** → the full 5-task initiation pipeline.
- **No near-term earnings, prior report exists** → `thesis-tracker` to check whether the
  existing thesis is still intact, instead of regenerating everything.

---

## Output

Everything lands under a date-stamped folder.

Full initiation:

```
/Research/{TICKER}/{YYYY-MM-DD}/
  01_Company_Research.md
  02_Financial_Model.xlsx          <- valuation tabs are added to this same workbook
  03_Valuation_Analysis.md
  04_Charts/
    chart_01_*.png ... chart_NN_*.png
  05_Initiation_Report.docx
```

Near-term earnings:

```
/Research/{TICKER}/{YYYY-MM-DD}/
  Earnings_Update_Note.docx
  Model_Update.xlsx
```

Prior coverage exists:

```
/Research/{TICKER}/{YYYY-MM-DD}/
  Thesis_Check.md
```

Exploration:

```
/Research/_Exploration/{THEME}/{YYYY-MM-DD}/
  00_Sector_Overview.md
  00_Competitive_Landscape.md
  00_Peer_Comps.xlsx
  00_Idea_Shortlist.md
  {TICKER}/                        <- one per candidate, full initiation layout
```

---

## How it differs from calling the plugins directly

**No pausing between tasks.** `equity-research:initiating-coverage` ships in single-task
mode — a full initiation report normally takes five separate prompts, and the skill is
explicitly instructed to ask which task you want and never to chain them. The three agent
plugins each stop mid-workflow to request analyst approval. `auto-coverage` overrides
those pause points and runs the whole sequence in one go. Removing that manual
re-prompting is the entire reason this wrapper exists.

**What it does not override:** every data-integrity guardrail stays in force. Numbers are
cited or marked `[UNSOURCED]`, filings and transcripts are treated as untrusted data,
nothing is published or distributed, and no quality minimum is relaxed. Speed comes from
removing prompts, not from producing less. The deliverables list is also fixed — no
summary, roll-up, or completion documents, and no zip of the whole output folder (only
Task 4's own charts zip, staying inside `04_Charts/`).

**Rigor directives on the valuation.** The Task 3 and Task 5 invocations carry a fixed
set of general disclosure requirements: one dated current price used in every
calculation, a statement of how sensitive the rating is to the valuation-method weights
(and whether a plausible re-weighting flips it), explicit flagging of load-bearing
assumptions with their source quality (company-confirmed vs press-reported vs estimate),
an argued variant view wherever the model diverges materially from consensus, and a
closing list of observable signposts that would confirm or break the thesis. After
Task 3, shared figures (price, share count, market cap) are checked for consistency
across deliverables. These are disclosure requirements passed to the underlying skills —
`auto-coverage` still computes nothing itself.

**Verification instead of confirmation.** Before each task, `auto-coverage` checks that the
previous task's output exists, is non-empty, and is well-formed — the right tabs in the
workbook, the mandatory charts present, a price target in the valuation. If a check fails
it halts and names the failed task and the missing file rather than continuing with bad
inputs. In holdings mode a halt is scoped to that one ticker; the rest of the list
continues.

---

## Repo layout

```
.claude-plugin/
  marketplace.json
plugins/
  auto-coverage/
    .claude-plugin/plugin.json
    .mcp.json                          <- optional twse-mcp connector
    skills/auto-coverage/SKILL.md      <- orchestration logic
    commands/auto-coverage.md          <- /auto-coverage
```
