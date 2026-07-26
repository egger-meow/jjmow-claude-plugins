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

### 3. Connect the data sources

The agent plugins read market data over MCP — `capiq`, `factset`, and `daloopa`. Connect
whichever your firm provides before running. If a required source is missing,
`auto-coverage` stops and says so rather than estimating or fabricating figures.

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
removing prompts, not from producing less.

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
    skills/auto-coverage/SKILL.md      <- orchestration logic
    commands/auto-coverage.md          <- /auto-coverage
```
