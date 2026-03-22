# Session Playbook

Detailed instructions for each phase of a coding session. Follow this exactly when running Workflow B (Resume & Work).

---

## Phase 1: Orient

**Time budget:** Keep this short. Read state, don't explore.

### Actions

1. **Read progress.md** — focus on the last 2-3 session entries:
   ```bash
   # Read the progress file
   ```
   Key info to extract:
   - What feature was last worked on
   - Was it completed or partial
   - Any known issues or broken state

2. **Read features.json** — parse the feature list:
   ```bash
   # Read features.json
   ```
   Key info to extract:
   - Total features count
   - How many pass (`passes: true`)
   - First feature where `passes: false` (this is the next target)

3. **Check git history** for recent work:
   ```bash
   git log --oneline -20
   git status
   ```
   Key info to extract:
   - Are there uncommitted changes? (If yes, assess and commit or stash)
   - What were the last few commits about?

4. **Announce orientation results:**
   ```
   Project status: X/Y features passing (N%).
   Last session: <summary>.
   Next feature: #<id> — "<description>"
   ```

### Guardrails

- Do NOT start reading source code files during orientation. Stay focused on state files.
- If there are uncommitted changes, handle them BEFORE proceeding:
  - If they look like partial work: commit with `wip:` prefix
  - If they look like leftover garbage: `git stash` or `git checkout -- .`

---

## Phase 2: Verify

**Purpose:** Ensure the project is in a working state before making changes.

### Actions

1. **Check for init.sh:**
   ```bash
   ls init.sh
   ```
   If it exists, run it (or the test portion):
   ```bash
   bash init.sh
   ```

2. **Run smoke test:**
   - If the project has a test command: run it
   - If it's a web app: start the server, verify it loads
   - If it's a library: run the existing test suite
   - If none of the above: at minimum, check that the code compiles/parses

3. **Assess results:**

   | Outcome | Action |
   |---------|--------|
   | All tests pass, app runs | Proceed to Phase 3 |
   | Minor test failure | Fix it now, commit as `fix:`, then proceed |
   | Major breakage | Fix it first. This IS the session's work. Update progress.md |
   | Cannot determine | Ask user for guidance |

### Guardrails

- NEVER skip verification. Even if you "know" it works, run the check.
- If fixing a broken state takes the entire session, that is acceptable. Log it and move on.
- Always commit fixes separately from feature work.

---

## Phase 3: Implement

**Purpose:** Complete exactly ONE feature.

### Feature Selection

1. Read `features.json`, find the first entry where `passes: false`.
2. This is your target. Announce it to the user:
   ```
   Working on feature #<id>: "<description>"
   Steps: <list the steps>
   ```

### Implementation Strategy

1. **Plan before coding:** Think through the approach. If the feature is complex, outline the changes needed.

2. **Work incrementally:** Make small, logical changes. Commit frequently:
   ```bash
   git add -A && git commit -m "feat(<feature-id>): <what this commit does>"
   ```

3. **Test as you go:** After each significant change, run relevant tests.

4. **Stay focused:** Do NOT work on other features, refactor unrelated code, or add enhancements beyond the feature spec.

### Self-Verification

Before marking a feature as passing, verify EVERY step listed in the feature's `steps` array:

1. Go through each step in order.
2. Verify it works (run test, check output, exercise the UI).
3. **Only if ALL steps pass**, update `features.json`:
   ```json
   "passes": true
   ```
4. If any step fails, keep working or document what's blocking.

### Guardrails

- **ONE feature per session.** If you finish early and want to start another, ask the user first.
- **Never mark a feature as passing without verification.** Premature completion is the #1 failure mode.
- **Never modify feature descriptions or steps.** Only change the `passes` field.
- **If the feature is too large** to complete in one session:
  - Implement as much as you can
  - Commit partial progress
  - Do NOT mark it as passing
  - Log the partial state in progress.md
  - The next session will continue where you left off

---

## Phase 4: Cleanup

**Purpose:** Leave the project in a perfect state for the next session.

### Actions

1. **Ensure all changes are committed:**
   ```bash
   git status  # Should show clean working tree
   ```
   If not clean:
   ```bash
   git add -A && git commit -m "feat(<feature-id>): complete <description>"
   ```

2. **Update progress.md** — append a new session entry:
   ```markdown
   ### Session N — YYYY-MM-DD
   - **Feature worked on:** #<id> — <description>
   - **Result:** Completed / Partial
   - **Commits:** <hash1>, <hash2>, ...
   - **Issues encountered:** <any problems, or "None">
   - **State:** Clean
   - **Next up:** #<next-id> — <next description>
   ```

3. **Commit progress update:**
   ```bash
   git add progress.md && git commit -m "docs: session N progress update"
   ```

4. **Update cortex ctx:**
   ```bash
   cortex ctx step done <step_id>
   ```

5. **Present session summary to user:**
   ```
   Session complete.
   - Completed: #<id> — "<description>"
   - Progress: X/Y features passing (N%)
   - Next session: #<next-id> — "<next description>"
   - Project state: Clean, all tests passing
   ```

6. **Context management:**
   - If the session has been long, suggest: "Run `/compact` to free context for the next feature."
   - If resuming later, remind: "Use `cortex -r last` to resume, or start a new session and run `$long-running`."

### Guardrails

- NEVER end a session with uncommitted changes.
- NEVER end a session without updating progress.md.
- NEVER skip the session summary.

---

## Quick Reference: Session Flow

```
┌─────────────────────┐
│  1. ORIENT           │  Read progress.md, features.json, git log
│     (~2 min)         │  Announce status
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  2. VERIFY           │  Run init.sh / smoke test
│     (~3 min)         │  Fix broken state if needed
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  3. IMPLEMENT        │  Pick ONE feature
│     (bulk of time)   │  Build, test, verify
│                      │  Mark passes: true
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  4. CLEANUP          │  Git commit, update progress.md
│     (~2 min)         │  ctx step done, summary
└─────────────────────┘
```
