# EV Charging Site Planner

Autonomous EV charging station site planning pipeline — scrapes real-world station data, analyzes geographic coverage gaps, evaluates candidate sites against multi-factor criteria, and produces investment-grade assessment reports with a full permitting tracker.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SITE ASSESSMENT PIPELINE                         │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │ scrape-      │───▶│ process-     │───▶│ analyze-coverage-    │  │
│  │ station-data │    │ station-data │    │ gaps                 │  │
│  │ (Playwright) │    │ (jq+python3) │    │ (sequential-think.)  │  │
│  └──────────────┘    └──────────────┘    └──────────┬───────────┘  │
│                                                     │              │
│                                          ┌──────────▼───────────┐  │
│                                          │ generate-candidate-  │  │
│                                          │ list (python3)       │  │
│                                          └──────────┬───────────┘  │
│                                                     │              │
│                          ┌──────────────────────────▼───────────┐  │
│                          │        evaluate-site (Opus)           │  │
│                          │  verdict: approve/conditional/reject  │  │
│                          │          /defer → rework              │  │
│                          └──┬──────────────────────┬────────────┘  │
│                    rework   │                      │ approve/cond  │
│                  ┌──────────▼──────────┐           │               │
│                  │ collect-missing-    │───────────┘               │
│                  │ data (Playwright)   │ (re-enters evaluate-site) │
│                  └─────────────────────┘                           │
│                                                     │              │
│                                         ┌───────────▼────────────┐ │
│                                         │ generate-assessment-   │ │
│                                         │ reports (report-writer)│ │
│                                         └───────────┬────────────┘ │
│                                                     │              │
│                                         ┌───────────▼────────────┐ │
│                                         │ stakeholder-review     │ │
│                                         │ [MANUAL GATE]          │ │
│                                         └──┬────────────┬────────┘ │
│                                    reject  │            │ approve  │
│                                  (back to  │            │          │
│                                 evaluate)  │  ┌─────────▼────────┐ │
│                                            │  │ initialize-      │ │
│                                            │  │ permit-tracking  │ │
│                                            │  └──────────────────┘ │
└────────────────────────────────────────────┴────────────────────────┘

SCHEDULED WORKFLOWS:
  Every Monday 6 AM   →  weekly-gap-refresh  (scrape + gap analysis)
  Every Wednesday 8AM →  permit-status-check (flag stalls + overdue)
  Quarterly (Jan/Apr/Jul/Oct) → quarterly-expansion (full Q report)
```

## Quick Start

```bash
cd examples/ev-charging-planner
ao daemon start

# Run a full site assessment for the configured region (DFW by default)
ao queue enqueue \
  --title "ev-charging-planner" \
  --description "Run full site assessment for DFW metro" \
  --workflow-ref site-assessment

# Check status
ao daemon stream --pretty
ao task list
```

To assess a different region, edit `data/config.json` before enqueueing.

## Agents

| Agent | Model | Role |
|---|---|---|
| `data-scraper` | claude-haiku-4-5 | Scrapes AFDC and PlugShare via Playwright; also collects missing site data |
| `gap-analyzer` | claude-sonnet-4-6 | Analyzes coverage gaps using sequential reasoning; tracks baselines in memory |
| `site-evaluator` | claude-opus-4-6 | Evaluates candidate sites with 5-factor rubric; issues structured verdicts |
| `report-writer` | claude-sonnet-4-6 | Produces site assessment reports, gap briefs, and quarterly expansion reports |
| `permit-tracker` | claude-haiku-4-5 | Monitors permit pipeline for stalls and overdue milestones |

## AO Features Demonstrated

| Feature | Where Used |
|---|---|
| **Playwright MCP** | `scrape-station-data` and `collect-missing-data` — real browser automation to scrape AFDC/PlugShare |
| **Sequential-thinking MCP** | `analyze-coverage-gaps` and `evaluate-site` — structured multi-dimensional reasoning |
| **Memory MCP** | All agents store/retrieve baselines, history, and cross-run context |
| **Decision contracts** | `evaluate-site` — structured verdict with `approved_sites`, `conditional_sites`, `deferred_sites` |
| **Rework loops** | `evaluate-site → collect-missing-data → evaluate-site` — auto-collects missing data and retries |
| **Manual gate** | `stakeholder-review` — human approval before permitting begins |
| **Command phases (jq)** | `process-station-data` — connector type summarization with jq |
| **Command phases (python3)** | `process-station-data` and `generate-candidate-list` — grid gap computation and site file generation |
| **Scheduled workflows** | Weekly gap refresh, weekly permit check, quarterly expansion report |
| **Multi-model** | Haiku for scraping/tracking, Sonnet for analysis/reporting, Opus for high-stakes site decisions |
| **Phase routing** | Verdict-based routing: rework→data-collection, approve/conditional/reject→reports, reject at gate→re-evaluate |

## Requirements

### No API Keys Required (defaults)
The AFDC API supports unauthenticated requests with `api_key=DEMO_KEY` at limited rate.
For production use, register free at https://developer.nrel.gov/ to get a key, then
set `NREL_API_KEY` in your environment and update the scraper directive.

### Tools
- Node.js 18+ (for npx MCP servers)
- Python 3.8+ (for data processing scripts)
- `jq` (for JSON processing in command phases)
- Chromium (auto-installed by Playwright MCP)

### MCP Servers (auto-installed via npx)
- `@playwright/mcp` — browser automation for scraping
- `@modelcontextprotocol/server-filesystem` — file read/write
- `@modelcontextprotocol/server-memory` — persistent knowledge graph
- `@modelcontextprotocol/server-sequential-thinking` — structured reasoning

## Output Structure

```
reports/
  coverage-gaps.json           # Structured gap analysis with severity ratings
  coverage-gap-summary.md      # Human-readable gap brief
  coverage-gap-brief.md        # Concise brief for stakeholders
  site-assessments/
    GAP-001-S01.json           # Raw assessment data per site
    GAP-001-S01-report.md      # Full formatted site report
  assessment-index.md          # Index of all assessed sites
  quarterly-expansion-report.md

data/
  stations-raw.json            # Raw scraped station data
  stations-clean.json          # Cleaned, deduplicated stations
  connector-summary.json       # Station counts by connector type
  coverage-grid.json           # 10-mile grid gap analysis
  candidate-list.json          # All candidate sites
  candidates/                  # Individual site data files
  permit-tracker.json          # Live permit pipeline status
  permit-status-update.md      # Latest permit check report
  config.json                  # Region configuration
```
