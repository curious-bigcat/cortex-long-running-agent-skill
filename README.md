# Long-Running Agent Skill for Cortex Code

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that orchestrates complex, multi-session projects using the [Anthropic harness pattern](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). It enables an AI agent to make structured, incremental progress across many context windows without losing track of work.

## The Problem

AI agents working on complex projects tend to:
- Forget what happened in previous sessions
- Try to build everything at once ("one-shotting")
- Declare victory prematurely
- Leave the environment in a broken state between sessions

## The Solution

This skill wraps Cortex Code in a disciplined harness with two workflows:

1. **Initializer** — expands a high-level spec into a structured feature list, creates a progress log, sets up smoke tests, and initializes git
2. **Coding sessions** — each session reads state, verifies the environment, implements exactly ONE feature, self-verifies, commits, and logs progress

The result is an agent that can stop, restart, and keep improving a project across many independent sessions.

## Installation

Copy the skill to your Cortex Code global skills directory:

```bash
# Clone the repo
git clone https://github.com/<YOUR_USERNAME>/cortex-long-running-agent-skill.git

# Copy to Cortex Code skills directory
cp -r cortex-long-running-agent-skill ~/.snowflake/cortex/skills/long-running-agent
```

Or symlink it:

```bash
ln -s "$(pwd)/cortex-long-running-agent-skill" ~/.snowflake/cortex/skills/long-running-agent
```

### Verify installation

Open Cortex Code and run `/skill` — you should see `long-running-agent` listed under Global skills.

## Usage

### Initialize a new project

```
$long-running init "Build a CLI todo app with add, list, complete, and delete commands"
```

This will:
- Generate a `features.json` file with testable features (all marked `passes: false`)
- Create an `init.sh` smoke test script
- Create a `progress.md` session log
- Initialize a git repo with an initial commit
- Register persistent context via `cortex ctx`

### Work on the next feature

```
$long-running
```

Each session follows four phases:

| Phase | What happens |
|-------|-------------|
| **Orient** | Reads `progress.md`, `features.json`, and git log to understand current state |
| **Verify** | Runs `init.sh` / smoke tests to catch regressions before new work |
| **Implement** | Picks ONE incomplete feature, builds it, self-verifies, marks it passing |
| **Cleanup** | Commits to git, updates `progress.md`, presents session summary |

### Check status

```
$long-running status
```

Shows feature completion percentage, recent session history, and what's next.

### Resume across sessions

```bash
# Resume the last Cortex Code session
cortex -r last

# Then continue working
$long-running
```

## Project Structure

When initialized, your project will contain:

```
my-project/
├── features.json     # Feature list with pass/fail tracking (JSON)
├── init.sh           # Smoke test / environment setup script
├── progress.md       # Session-by-session progress log
└── ... (your code)
```

### features.json format

```json
[
  {
    "id": 1,
    "category": "functional",
    "description": "User can add a new todo item",
    "steps": [
      "Run the add command with a description",
      "Verify the item appears in storage",
      "Verify confirmation output"
    ],
    "passes": false,
    "priority": 1
  }
]
```

**Rules enforced by the skill:**
- Features always start as `passes: false`
- Only the `passes` field may be changed — descriptions and steps are immutable
- Features must be verified before being marked as passing

## Skill File Structure

```
long-running-agent/
├── SKILL.md                        # Main skill definition (3 workflows)
├── references/
│   └── session-playbook.md         # Detailed per-session phase instructions
└── templates/
    └── features-template.json      # Feature JSON template
```

## How It Maps to Cortex Code Primitives

| Anthropic Pattern | Cortex Code Equivalent |
|---|---|
| Progress file | `progress.md` + `cortex ctx remember` |
| Feature list (JSON) | `features.json` + `cortex ctx task/step` |
| Cross-session memory | `cortex ctx` persistent memory & rules |
| Session resume | `cortex -r last` / `/resume` |
| Context management | `/compact` when context gets long |
| Git safety net | Built-in git tools |

## Customization

### Adjust feature granularity

Edit the skill prompt in `SKILL.md` to change the target feature count:
- Small projects: 5-15 features
- Medium projects: 15-30 features
- Large projects: 30-50+ features

### Add specialized agents

You can extend the pattern by defining additional custom agents in `.cortex/agents/` for:
- **Testing agent** — runs comprehensive tests after each feature
- **QA agent** — reviews code quality before marking features as passing
- **Cleanup agent** — refactors and polishes after all features pass

## License

MIT
