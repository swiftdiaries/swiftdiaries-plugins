---
name: execute-dev-loop
description: Execute an implementation plan end-to-end using beads task tracking and superpowers development workflows. Creates an epic and tasks from plan sections, drives them to completion through TDD, parallel subagents, and code review. Use whenever the user references a plan file from docs/superpowers/plans/, says "execute the plan", "work the plan", "run the dev loop", "pick up plan tasks", "resume the plan", "run the pipeline", "dev loop", or wants to drive any implementation plan to completion. Also use when the user says "bd ready" and picks up tasks that reference a plan — this skill governs how those tasks get implemented.
---

# Execute Dev Loop

Drive an implementation plan to completion through a structured pipeline: **ingest -> execute -> complete**.

**Announce at start:** "Using execute-dev-loop to drive [plan name] to completion."

**This is a rigid skill.** Follow each phase exactly. The discipline of the pipeline is the point.

## Prerequisites

- `bd` CLI configured and working
- Plan file exists (typically in `docs/superpowers/plans/`)
- Git repo is clean (`git status` shows no uncommitted changes)

---

## Phase 1: Ingest Plan -> Create Epic + Tasks

### Step 1: Check for Resume

Read the plan file. Look for a `## Pipeline Tracking` section near the top.

**If tracking section found:** Tasks already exist. Skip to Phase 2.

**If no tracking section but tasks might exist:** Search beads for tasks referencing this plan:
```bash
bd search "<plan-filename>"
```
If tasks are found, reconstruct the tracking section from beads state, update the plan file, then skip to Phase 2.

**If neither:** Continue with full ingestion.

### Step 2: Read and Parse the Plan

Read the full plan file. Extract:

- **Plan title** from the first `#` heading
- **Spec file** if referenced (usually in the header metadata)
- **Task sections** — look for `### Task N:` patterns, but adapt to whatever heading structure the plan uses
- **Per-task details:** title, description, files modified, steps, test commands, expected behavior
- **Dependencies** — infer from: explicit "depends on Task N" mentions, file overlap between tasks, or sequential ordering in the plan
- **TDD coverage** — which tasks specify test commands, expected results, or "write tests"?

Plan formats vary. Adapt parsing to the actual structure rather than assuming rigid formatting.

### Step 3: Ensure TDD Coverage

Check whether tasks in the plan include TDD steps (test commands, expected test output, "write tests first" instructions).

**If tasks lack TDD specifications:** The plan needs enrichment before task creation. Invoke **superpowers:writing-plans** with the focus on adding TDD to each task:

- For each task that modifies or creates code, add: what tests to write, what test commands to run, and what the expected passing/failing behavior looks like.
- Tasks that are purely config, deletion, or documentation may not need TDD — use judgment.
- The rewritten plan replaces the original. Re-read and re-parse after the rewrite.

**If all code tasks already have TDD:** Continue to Step 4.

This gate ensures every plan that enters the pipeline has testable tasks. Writing tests is not optional — if the plan doesn't specify them, fix the plan first.

### Step 4: Create Epic

```bash
bd create --title="<plan-title>" --type=feature --priority=2 \
  --description="Execution tracking for <plan-file-path>" \
  --notes="Plan: <plan-file-path>"
```

Save the epic ID (e.g., `ghostfin-abc`).

### Step 5: Create Tasks

For each task section, create a beads task. **Use parallel subagents when creating many tasks** for efficiency.

Each task must faithfully represent the plan section — capture:
- The task's purpose and scope
- Files to modify/create/delete
- Implementation steps
- TDD requirements (test commands, expected results, what to test)
- Verification commands (build, lint, type-check)

```bash
bd create \
  --title="<task-title>" \
  --description="<faithful-description-from-plan>" \
  --type=task \
  --priority=2 \
  --notes="Plan: <plan-file-path> | Task section: Task <N> | Epic: <epic-id>"
```

After all tasks are created, wire dependencies:
```bash
bd dep add <downstream-task> <upstream-task>
```

### Step 6: Update Plan File (CRITICAL)

This step is mandatory. The plan file is the persistent source of truth.

Add a `## Pipeline Tracking` section immediately after the plan's title/header block (before the first `---` or `## File Map`):

```markdown
## Pipeline Tracking

| Field | Value |
|-------|-------|
| Epic | `<epic-id>` |
| Status | in_progress |
| Worktree | (pending) |

| Task | Beads ID | Status |
|------|----------|--------|
| Task 1: <title> | <task-id> | open |
| Task 2: <title> | <task-id> | open |
| ... | ... | ... |
```

Commit the update:
```bash
git add <plan-file>
git commit -m "plan: add pipeline tracking to <plan-name>"
```

---

## Phase 2: Execute Loop

This is the core development loop. Repeat until no open tasks remain.

### Step 1: Check Ready Tasks

```bash
bd ready
```

- **Ready tasks exist:** Continue to Step 2.
- **No ready tasks, but open tasks exist:** Everything is blocked. Run `bd blocked` to show what's stuck. Report to the user and stop.
- **No open tasks remain:** Skip to Phase 3 (Complete).

### Step 2: Create Worktree (First Pass Only)

On the first execution pass, decide whether to use a worktree:

- **3+ tasks or plan touches multiple packages:** Create a worktree.
  Invoke **superpowers:using-git-worktrees** with branch name `feature/<plan-slug>`.
  (Derive slug from plan filename: `2026-04-11-post-onboarding-integration.md` -> `post-onboarding-integration`)
- **<3 tasks, single-package, narrow scope:** Work on current branch.

Update the Pipeline Tracking `Worktree` field with the path and branch.

### Step 3: Claim and Group Tasks

Claim all ready tasks:
```bash
bd update <id> --claim
```

**Group for parallelism:** Tasks that don't share files can run in parallel. Tasks that modify the same files must be sequential.

### Step 4: Implement

Invoke **superpowers:subagent-driven-development** to implement the task group. Each subagent receives:

1. **The task's plan section** — read from the plan file, not just the beads description. The plan has the full implementation detail (code snippets, exact file locations, step-by-step instructions).

2. **TDD workflow** (the default for code tasks):
   - Write failing tests first per the plan's test specifications.
   - Implement the code to make tests pass.
   - Verify all tests pass.

3. **Direct implementation** (only for non-code tasks like config, docs, deletion):
   - Implement per the plan steps.
   - Run any verification commands specified.

4. **Verification:** Run any build/test/lint commands specified in the plan.

**Subagent rule:** Always dispatch with `model: "opus"`.

### Step 5: Review

After implementation, invoke **superpowers:requesting-code-review** to review against the spec.

- **Review scope:** All changes from this execution pass.
- **Review against:** The plan's spec file (if one exists) and the task sections just implemented.
- Fix any issues the review identifies.
- Re-run tests after fixes.

### Step 6: Close Tasks and Update Plan (CRITICAL)

Close completed tasks:
```bash
bd close <id1> <id2> ...
```

Update the Pipeline Tracking table in the plan file:
- `open` -> `closed` for completed tasks
- Update any in-progress markers

Commit everything:
```bash
git add <changed-files> <plan-file>
git commit -m "<descriptive-commit-message>"
```

### Step 7: Loop

Return to Step 1. Check what's newly unblocked and continue.

---

## Phase 3: Complete

### Step 1: Final Verification

- All beads tasks for this epic are closed
- All tests pass
- Build is clean
- No uncommitted changes

### Step 2: Update Plan File (CRITICAL)

Update the Pipeline Tracking section to reflect completion:

```markdown
## Pipeline Tracking

| Field | Value |
|-------|-------|
| Epic | `<epic-id>` |
| Status | complete |
| Completed | <YYYY-MM-DD> |
| Worktree | <path> (branch: <branch-name>) |

| Task | Beads ID | Status |
|------|----------|--------|
| Task 1: <title> | <task-id> | closed |
| Task 2: <title> | <task-id> | closed |
| ... | ... | ... |
```

### Step 3: Close Epic

```bash
bd close <epic-id> --reason="All plan tasks complete"
```

### Step 4: Finish Branch

Invoke **superpowers:finishing-a-development-branch** to present options (merge, PR, keep, discard).

---

## Skill Invocation Chain

This skill orchestrates these superpowers skills in order:

| Phase | Skill | Purpose |
|-------|-------|---------|
| Phase 1, Step 3 | `superpowers:writing-plans` | Enrich plan with TDD if missing |
| Phase 2, Step 2 | `superpowers:using-git-worktrees` | Isolate plan work |
| Phase 2, Step 4 | `superpowers:subagent-driven-development` | Parallel task implementation |
| Phase 2, Step 4 (per task) | `superpowers:test-driven-development` | TDD when plan requires it |
| Phase 2, Step 5 | `superpowers:requesting-code-review` | Review against spec |
| Phase 3, Step 4 | `superpowers:finishing-a-development-branch` | Merge/PR/keep decision |

## Key Rules

1. **One plan = one epic = one worktree.** Never mix plans.
2. **Update the plan file at every stage transition.** Creation, task completion, and plan completion all require plan file updates. This is non-negotiable.
3. **Faithfully represent the plan.** Task descriptions capture the plan's full intent — files, TDD requirements, verification steps, code snippets. Don't summarize away detail.
4. **No plan enters the pipeline without TDD.** If the plan lacks test specifications, rewrite it with `superpowers:writing-plans` before creating tasks.
5. **Parallelize independent tasks.** Group tasks without shared files for subagent-driven-development.
6. **Always use opus model for subagents.**
7. **The plan file is the source of truth.** Anyone reading the plan file should know exactly where the pipeline stands.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Creating tasks from a plan with no TDD | Invoke writing-plans to add TDD first |
| Creating tasks without updating plan file | Plan file update is **mandatory** after task creation |
| Summarizing task descriptions too aggressively | Copy implementation detail faithfully from the plan |
| Starting implementation before claiming | Always `bd update --claim` first |
| Skipping review for "small" changes | Always review against spec |
| Forgetting plan file update on completion | Plan must show `complete` status with date |
| Working on blocked tasks | Only work `bd ready` tasks — check every loop iteration |
| Mixing plans in one worktree | One plan per worktree, always |
| Using non-opus model for subagents | Always `model: "opus"` |
