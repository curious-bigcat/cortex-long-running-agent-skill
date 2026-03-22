# Long-Running Agent Skill for Cortex Code

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that orchestrates complex, multi-session projects using the [Anthropic harness pattern](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). It enables an AI agent to make structured, incremental progress across many context windows without losing track of work.

## The Problem

When you ask Cortex Code to build something complex, the agent tends to:

- Forget what it built in the previous session
- Try to build everything in one shot
- Mark things as done before they actually work
- Leave the project in a broken state between sessions

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

### 1. Provide your project spec

You can provide your project requirements in two ways:

**Option A: Free text**

```
$long-running init "Build a REST API with user authentication, rate limiting, and a Postgres backend"
```

**Option B: Reference a markdown file**

Write your requirements in a `.md` file with as much detail as you want:

```markdown
# My Project Requirements

## Overview
Build a data processing system that ingests CSV files, validates them,
transforms the data, and loads it into a target database.

## Features
- File upload via API endpoint
- Schema validation with clear error messages
- Configurable transformation rules
- Incremental loading with deduplication
- Error logging and retry mechanism
- Admin dashboard to monitor pipeline status
```

Then point the skill at it:

```
$long-running init @requirements.md
```

The more detail you provide, the better the feature decomposition. But even a single sentence works -- the skill will ask clarifying questions if the spec is too vague.

### 2. What initialization creates

The skill decomposes your spec into concrete, testable features:

```
my-project/
├── features.json       # Feature list with pass/fail tracking
├── init.sh             # Smoke test script for your specific project
├── progress.md         # Session-by-session progress log
├── .git/               # Git repo -- every feature gets a commit
└── ... (your code)
```

**features.json** contains structured, verifiable features:

```json
[
  {
    "id": 1,
    "category": "foundation",
    "description": "Project structure with entry point that runs without errors",
    "steps": [
      "Run the main entry point or help command",
      "Verify it executes without errors",
      "Verify exit code is 0"
    ],
    "passes": false,
    "priority": 1
  },
  {
    "id": 2,
    "category": "functional",
    "description": "File upload endpoint accepts CSV and returns confirmation",
    "steps": [
      "POST a valid CSV to the upload endpoint",
      "Verify 200 response with file metadata",
      "Verify the file is persisted to the expected location"
    ],
    "passes": false,
    "priority": 2
  }
]
```

**init.sh** is tailored to your project -- it might run a test suite, start a dev server, check a database connection, or simply verify the code compiles. It runs at the start of every session to catch regressions.

### 3. Work on features one at a time

```
$long-running
```

Each session follows four phases:

```
ORIENT  →  VERIFY  →  IMPLEMENT  →  CLEANUP
```

| Phase | What happens |
|-------|-------------|
| **Orient** | Reads `progress.md`, `features.json`, and git log. Announces current state and next feature. |
| **Verify** | Runs `init.sh` to catch regressions. Fixes anything broken before starting new work. |
| **Implement** | Picks ONE incomplete feature. Builds it. Self-verifies against the feature's test steps. Marks `passes: true` only when verified. |
| **Cleanup** | Commits to git, updates `progress.md`, presents summary with completion percentage. |

### 4. Resume across sessions

```bash
# Pick up where you left off
cortex -r last
$long-running
```

Or start a fresh session -- the skill reads `features.json` and `progress.md` to reconstruct full project state.

### 5. Check status anytime

```
$long-running status
```

Shows completion percentage, recent session history, and what's next.

## How a Session Looks in Practice

```
$long-running

> ORIENT: Reading project state...
> Status: 5/14 features passing (36%)
> Last session: Completed feature #5 — "Schema validation with error messages"
> Next up: Feature #6 — "Configurable transformation rules"

> VERIFY: Running init.sh...
> All checks pass. Environment is clean.

> IMPLEMENT: Working on feature #6 — "Configurable transformation rules"
>   Steps to verify:
>     1. Define a transformation config format
>     2. Apply a sample transformation to test data
>     3. Verify output matches expected result
>
> (agent implements the feature, tests it, marks it passing)

> CLEANUP:
>   Committed: feat(6): configurable transformation rules with YAML config
>   Progress: 6/14 features passing (43%)
>   Next session: Feature #7 — "Incremental loading with deduplication"
>   Project state: Clean, all tests passing.
```

## Providing a Good Spec

The quality of the feature decomposition depends on the quality of your input. Here are guidelines:

### Minimal (works, but features will be broad)

```
$long-running init "Build a CLI tool that converts JSON to CSV"
```

### Better (clear scope, the skill can decompose well)

```
$long-running init "Build a CLI tool that converts JSON to CSV:
- Handles nested JSON by flattening with dot notation
- Supports arrays by expanding into multiple rows
- Accepts file path or stdin
- Outputs to file or stdout
- Handles encoding issues gracefully"
```

### Best (detailed spec as a markdown file)

```
$long-running init @spec.md
```

Where `spec.md` contains sections for overview, features, constraints, and acceptance criteria. The skill will turn each into a trackable feature with concrete verification steps.

### What makes a good spec

- **Be specific about outcomes**, not implementation. "Users can search by name and see results in under 2 seconds" is better than "Use Elasticsearch."
- **List the things a user can do.** Each action tends to map to one feature.
- **Mention constraints.** "Must work offline", "Must handle 10k rows", "Must support Python 3.10+" -- these become verification steps.
- **Don't worry about architecture.** The agent will figure out the how. Focus on the what.

## Typical Workflow

```
Day 1:
  $long-running init @requirements.md    # Decompose spec into features
  $long-running                          # Feature #1: project structure
  $long-running                          # Feature #2: core foundation

Day 2:
  cortex -r last
  $long-running status                   # "4/18 features passing (22%)"
  $long-running                          # Feature #5: next in line

Day 3+:
  # Same pattern. Each session picks up exactly where the last left off.
  # No context lost. No repeated work. No broken state.
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

## features.json Rules

- Every feature starts as `"passes": false`
- Only the `passes` field can be changed -- descriptions and steps are immutable
- A feature is only marked passing after the agent verifies every test step
- The agent works on the lowest-priority incomplete feature first

## Customization

### Adjust feature granularity

Edit `SKILL.md` to tune how many features the initializer generates:
- Small projects: 5-15 features
- Medium projects: 15-30 features
- Large projects: 30-50+ features

### Add specialized agents

Create custom agents in `.cortex/agents/` to extend the pattern:

- **test-agent** -- runs the full test suite after each feature
- **review-agent** -- reviews code quality before marking features as passing
- **cleanup-agent** -- refactors and polishes after all features pass

### Integrate with CI/CD

The git-commit-per-feature approach works naturally with CI pipelines. Each feature is an atomic, tested commit that can trigger automated checks on push.

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
