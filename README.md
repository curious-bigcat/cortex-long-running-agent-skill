# Long-Running Agent Skill for Cortex Code

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that orchestrates complex, multi-session data and AI projects using the [Anthropic harness pattern](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). It enables an AI agent to make structured, incremental progress across many context windows without losing track of work.

## The Problem

When you ask Cortex Code to build something complex -- a multi-model dbt project, a Streamlit dashboard backed by Cortex AI, or a full data pipeline -- the agent tends to:

- Forget what it built in the previous session
- Try to create all tables, models, and UDFs in one shot
- Mark things as done before they actually work end-to-end
- Leave broken SQL or half-migrated schemas between sessions

This skill fixes that by imposing structure: a feature list, a progress log, smoke tests, and git discipline.

## Installation

```bash
git clone https://github.com/curious-bigcat/cortex-long-running-agent-skill.git
cp -r cortex-long-running-agent-skill ~/.snowflake/cortex/skills/long-running-agent
```

Or symlink for easy updates:

```bash
ln -s "$(pwd)/cortex-long-running-agent-skill" ~/.snowflake/cortex/skills/long-running-agent
```

Verify it loaded -- open Cortex Code and run `/skill`. You should see `long-running-agent` under Global skills.

## Quick Start

### 1. Initialize your project

Navigate to your project directory in Cortex Code and run:

```
$long-running init "Build a customer analytics pipeline: ingest raw CSV data into Snowflake staging tables, transform with dbt models into a dimensional model, add Cortex AI sentiment analysis on support tickets, and expose metrics via a Streamlit dashboard"
```

The skill will:
- Decompose your spec into **concrete, testable features** in `features.json`
- Create `init.sh` with smoke tests (e.g., `dbt debug`, `snow sql` health checks)
- Create `progress.md` as a session-by-session log
- Initialize git and register persistent context via `cortex ctx`

### 2. Work on features one at a time

```
$long-running
```

The agent follows four phases every session:

```
ORIENT  →  VERIFY  →  IMPLEMENT  →  CLEANUP
```

| Phase | What happens |
|-------|-------------|
| Orient | Reads `progress.md`, `features.json`, git log. Announces: "12/25 features passing. Next: staging table for support tickets." |
| Verify | Runs `init.sh` -- checks `dbt build` passes, Streamlit app starts, Snowflake connection works. Fixes regressions before new work. |
| Implement | Picks ONE feature. Builds it. Self-verifies against the feature's test steps. Marks `passes: true` only when verified. |
| Cleanup | Commits to git, updates `progress.md`, presents summary with completion percentage. |

### 3. Resume across sessions

```bash
# Next day, pick up where you left off
cortex -r last
$long-running
```

Or start a fresh session -- the skill reads `features.json` and `progress.md` to reconstruct state.

### 4. Check status anytime

```
$long-running status
```

## Real-World Examples

### Example 1: dbt + Snowflake Data Pipeline

```
$long-running init "Migrate legacy Oracle ETL to Snowflake with dbt:
- 3 source systems (ERP, CRM, support tickets)
- Staging layer with incremental loads
- Intermediate transformations for deduplication and SCD2
- Mart layer with customer_360, order_metrics, and support_analytics models
- dbt tests for uniqueness, not_null, accepted_values, and referential integrity
- Documentation with descriptions and column-level docs"
```

The skill generates features like:

```json
[
  {
    "id": 1,
    "category": "foundation",
    "description": "dbt project structure with profiles.yml pointing to Snowflake",
    "steps": [
      "Run dbt debug and verify connection succeeds",
      "Verify dbt_project.yml has correct config",
      "Verify profiles.yml uses the correct Snowflake warehouse and database"
    ],
    "passes": false,
    "priority": 1
  },
  {
    "id": 2,
    "category": "foundation",
    "description": "Snowflake staging schema DDL and source definitions",
    "steps": [
      "Verify RAW database and STAGING schema exist in Snowflake",
      "Run dbt source freshness and confirm sources are defined",
      "Verify schema.yml has source descriptions"
    ],
    "passes": false,
    "priority": 2
  },
  {
    "id": 5,
    "category": "functional",
    "description": "stg_erp__orders incremental model with deduplication",
    "steps": [
      "Run dbt run -s stg_erp__orders",
      "Verify row count matches source after dedup",
      "Run dbt test -s stg_erp__orders -- uniqueness and not_null pass"
    ],
    "passes": false,
    "priority": 5
  }
]
```

Each session, the agent picks one model, builds it, tests it, and commits -- so you never end up with 15 half-broken models.

### Example 2: Cortex AI Application

```
$long-running init "Build a Cortex AI-powered document processing pipeline:
- Stage for PDF uploads
- Cortex PARSE_DOCUMENT to extract text
- Cortex AI_EXTRACT for entity extraction (invoice number, amount, vendor)
- Cortex AI_CLASSIFY to categorize documents (invoice, receipt, contract)
- Results stored in structured Snowflake tables
- Streamlit UI for upload, review, and correction
- Cortex Search service for full-text search across processed documents"
```

Sessions might look like:

```
Session 3 — Feature #3: Cortex PARSE_DOCUMENT stored procedure
  - Created SP that reads PDFs from stage, calls PARSE_DOCUMENT
  - Verified with 3 sample PDFs: text extraction works
  - Committed: feat(3): parse_document SP with error handling
  - Progress: 3/10 features passing (30%)
  - Next: #4 — AI_EXTRACT entity extraction
```

### Example 3: Streamlit Dashboard with Semantic Views

```
$long-running init "Build an executive analytics dashboard in Streamlit:
- Semantic view for revenue metrics (ARR, MRR, churn, expansion)
- Semantic view for product usage (DAU, MAU, feature adoption)
- Streamlit multi-page app with Overview, Revenue, and Usage pages
- Cortex Analyst integration for natural language queries
- Row-level security via session context for regional filtering
- Caching with st.cache_data and 15-minute TTL"
```

### Example 4: ML Pipeline on Snowflake

```
$long-running init "Build a customer churn prediction pipeline:
- Feature engineering with dynamic tables (aggregated usage, billing, support metrics)
- Training dataset view joining features with churn labels
- Snowpark ML model training (XGBoost) with hyperparameter tuning
- Model registered in Snowflake Model Registry
- Inference UDF for real-time scoring
- Monitoring table tracking prediction drift over time
- Streamlit app for model performance review and manual overrides"
```

## Project Structure

After initialization, your project contains:

```
my-project/
├── features.json       # Feature list with pass/fail tracking
├── init.sh             # Smoke tests (dbt debug, snow sql, streamlit check, etc.)
├── progress.md         # Session-by-session log
├── .git/               # Git repo -- every feature is a commit
└── ... (your code: dbt models, Streamlit apps, SQL scripts, etc.)
```

### features.json rules

- Every feature starts as `"passes": false`
- Only the `passes` field can be changed -- descriptions and steps are immutable
- A feature is only marked passing after the agent verifies every test step
- The agent works on the lowest-priority incomplete feature first

### init.sh examples

For a dbt project:
```bash
#!/bin/bash
set -e
dbt debug                    # Verify Snowflake connection
dbt build                    # Build and test all models
echo "Smoke test passed"
```

For a Streamlit app:
```bash
#!/bin/bash
set -e
python -c "import streamlit; print('streamlit OK')"
python -c "from app import main; print('app imports OK')"
echo "Smoke test passed"
```

## How It Maps to Cortex Code Primitives

| Anthropic Pattern | Cortex Code Equivalent |
|---|---|
| Progress file | `progress.md` + `cortex ctx remember` |
| Feature list (JSON) | `features.json` + `cortex ctx task/step` |
| Cross-session memory | `cortex ctx` persistent memory and rules |
| Session resume | `cortex -r last` or `/resume` |
| Context management | `/compact` when context gets long |
| Git safety net | Built-in git tools (commit, revert, diff) |
| Smoke tests | `init.sh` run at the start of every session |

## Typical Workflow for a Data Engineer

```
Day 1 morning:
  $long-running init "<project spec>"
  $long-running                        # Feature #1: project structure
  $long-running                        # Feature #2: staging schema

Day 1 afternoon:
  cortex -r last
  $long-running                        # Feature #3: first staging model
  $long-running                        # Feature #4: second staging model

Day 2:
  cortex -r last
  $long-running status                 # "8/20 features passing (40%)"
  $long-running                        # Feature #9: intermediate model
  ...

Day 3-5:
  # Same pattern. Each session picks up exactly where the last left off.
  # No context lost. No repeated work. No broken state.
```

## Customization

### Adjust feature granularity

Edit `SKILL.md` to tune how many features the initializer generates:
- Small projects (scripts, single models): 5-15 features
- Medium projects (dbt packages, multi-page apps): 15-30 features
- Large projects (full platform builds): 30-50+ features

### Add specialized agents

Create custom agents in `.cortex/agents/` to extend the pattern:

- **dbt-test-agent** -- runs `dbt test` with verbose output after each model
- **sql-review-agent** -- checks for performance issues (missing clustering keys, full table scans)
- **security-agent** -- verifies masking policies and row access policies are applied

### Integrate with CI/CD

The git-commit-per-feature approach means you can wire up GitHub Actions or similar:

```yaml
on:
  push:
    branches: [main]
jobs:
  dbt-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dbt build --target ci
```

## Skill File Structure

```
long-running-agent/
├── SKILL.md                        # Main skill (3 workflows: init, resume, status)
├── references/
│   └── session-playbook.md         # Detailed per-session phase instructions
└── templates/
    └── features-template.json      # Feature JSON template
```

## License

MIT
