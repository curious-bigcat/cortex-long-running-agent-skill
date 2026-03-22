---
name: long-running-agent
description: "Orchestrate long-running, multi-session projects using the Anthropic harness pattern. Use when: building complex apps across sessions, resuming long projects, needing structured incremental progress. Triggers: long running, multi-session, incremental build, long project, harness, init project, resume project, next feature."
tools: ["bash", "edit", "read", "write", "glob", "grep"]
---

# Long-Running Agent

Orchestrate complex, multi-session projects that span many context windows. Based on Anthropic's "Effective Harnesses for Long-Running Agents" pattern, adapted for Cortex Code's built-in tools.

## Core Principle

Work incrementally: one feature per session, always leave the project in a clean, resumable state.

## Setup

1. **Load** `references/session-playbook.md` -- required for all session workflows.

## Detecting Mode

Determine which workflow to run:

- If the user says `$long-running init <description>` or there is **no `features.json`** in the project root → run **Workflow A: Initialize**
- If `features.json` exists → run **Workflow B: Resume & Work**
- If the user says `$long-running status` → run **Workflow C: Status Report**

---

## Workflow A: Initialize Project

**Goal:** Set up a new project with all scaffolding for multi-session incremental development.

### Step 1: Understand the Spec

1. Read the user's description or requirements file.
2. Ask clarifying questions if the spec is too vague to decompose into features.

### Step 2: Generate Feature List

1. Expand the spec into **concrete, testable features** (aim for 10-50 depending on project size).
2. Write `features.json` in the project root using the structure from `templates/features-template.json`:
   ```json
   [
     {
       "id": 1,
       "category": "functional",
       "description": "Short, specific description of the feature",
       "steps": ["Step 1", "Step 2", "Step 3"],
       "passes": false,
       "priority": 1
     }
   ]
   ```
3. **CRITICAL RULES for features.json:**
   - Every feature starts with `"passes": false`
   - Features must be ordered by priority (dependencies first)
   - Each feature must be small enough to implement and verify in a single session
   - It is **unacceptable** to remove or edit feature descriptions. Only the `passes` field may be changed.

### Step 3: Create Project Scaffolding

1. **Create `init.sh`** in the project root:
   - Commands to install dependencies
   - Commands to start the dev server (if applicable)
   - A basic smoke test command that verifies the app runs
   - Make it executable: `chmod +x init.sh`

2. **Create `progress.md`** in the project root:
   ```markdown
   # Project Progress

   ## Project
   - **Name:** <project name>
   - **Created:** <date>
   - **Total Features:** <count>
   - **Completed:** 0

   ## Session Log

   ### Session 1 — <date>
   - **Type:** Initialization
   - **Actions:** Set up project scaffolding, feature list, init script
   - **Features completed:** None (setup only)
   - **State:** Clean, ready for first feature
   ```

3. **Initialize git** (if not already a repo):
   ```bash
   git init
   git add -A
   git commit -m "Initial project scaffolding: features.json, init.sh, progress.md"
   ```

### Step 4: Register Context

1. Save project info to persistent memory:
   ```bash
   cortex ctx remember "Long-running project: <name>. Features file: features.json. Progress: progress.md"
   cortex ctx rule add "Always read features.json and progress.md at the start of each session"
   cortex ctx rule add "Only work on ONE feature per session, commit before stopping"
   ```

2. Create `cortex ctx task` entries for the first few high-priority features:
   ```bash
   cortex ctx task add "<Feature 1 description>"
   cortex ctx step add "<step>" -t <task_id>
   ```

### Step 5: Confirm

Present a summary to the user:
- Total features generated
- Project structure created
- How to resume: `$long-running` or `$long-running next`
- How to check status: `$long-running status`

---

## Workflow B: Resume & Work

**Goal:** Orient, verify, pick one feature, implement it, leave clean state.

Follow the session playbook in `references/session-playbook.md` exactly. The phases are:

### Phase 1: Orient (mandatory, every session)

1. Read `progress.md` — understand what happened in previous sessions.
2. Read `features.json` — find the current state of all features.
3. Run `git log --oneline -20` — see recent commits.
4. Summarize: "X of Y features passing. Last session worked on: Z. Next up: W."

### Phase 2: Verify (mandatory, every session)

1. If `init.sh` exists, run it (or the relevant subset) to verify the environment works.
2. Run a basic smoke test to confirm no regressions.
3. **If the environment is broken:**
   - Fix it BEFORE doing anything else.
   - Commit the fix: `git commit -m "fix: <description of what was broken>"`
   - Log it in `progress.md`.
4. **If the environment is clean**, proceed to Phase 3.

### Phase 3: Implement (one feature only)

1. Select the **lowest-priority incomplete feature** from `features.json` (first `passes: false` entry).
2. Tell the user which feature you are working on.
3. Implement it incrementally, committing logical chunks to git.
4. After implementation, self-verify:
   - Run the feature's test steps from `features.json`
   - Manually verify if automated tests are not available
5. **Only if the feature truly works**, update `features.json`: set `"passes": true`.
6. Commit: `git add -A && git commit -m "feat: <feature description>"`

### Phase 4: Cleanup (mandatory, every session)

1. Update `progress.md` with a session entry:
   ```markdown
   ### Session N — <date>
   - **Feature worked on:** <description>
   - **Result:** Completed / Partial (explain)
   - **Commits:** <list commit hashes>
   - **Issues encountered:** <any problems>
   - **State:** Clean / Needs attention (explain)
   ```
2. Commit progress update: `git add progress.md && git commit -m "docs: update progress for session N"`
3. Mark completed `cortex ctx step done` entries.
4. Present summary to user:
   - What was done this session
   - Current feature completion count (X/Y passing)
   - What the next session should work on
5. If context is getting long, suggest: "Consider running `/compact` before the next feature."

---

## Workflow C: Status Report

**Goal:** Quick overview without doing any implementation work.

1. Read `features.json` and count passing/failing.
2. Read `progress.md` for the last 3 session entries.
3. Run `git log --oneline -10`.
4. Present:
   - Feature completion: X/Y (N%)
   - Features remaining (list next 5)
   - Last 3 session summaries
   - Estimated sessions remaining (1 feature per session)

---

## Failure Recovery

### Agent went off track
```bash
git log --oneline -10    # Find the last good commit
git revert HEAD          # Or: git reset --hard <good-commit>
```
Update `progress.md` to note the revert.

### Feature is too large
Split it into sub-features in `features.json`. Add new entries with higher IDs, adjust priorities. Never delete the original — mark it as `passes: true` once all sub-features pass.

### Context window is exhausted
Run `/compact` with instructions like: "Focus on the current feature being implemented and the project state from progress.md."

---

## Stopping Points

- **After Workflow A (init):** Stop and confirm with user before first feature.
- **After each feature in Workflow B:** Stop, present summary, let user decide to continue or end session.
- **Never stop mid-feature** without committing partial progress and noting it in `progress.md`.

## Error Handling

- **No features.json found and no init requested:** Ask user if they want to initialize a new project.
- **All features passing:** Congratulate and suggest cleanup, testing, or deployment.
- **init.sh fails:** Diagnose the failure, fix it, update the script, commit the fix.
