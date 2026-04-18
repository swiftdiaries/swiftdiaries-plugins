---
name: execute-dev-loop
description: Execute an implementation plan end-to-end using beads task tracking and superpowers development workflows. Creates an epic and tasks from plan sections, drives them to completion through TDD, parallel subagents, and code review. Use whenever the user references a plan file from docs/superpowers/plans/, says "execute the plan", "work the plan", "run the dev loop", "pick up plan tasks", "resume the plan", "run the pipeline", "dev loop", or wants to drive any implementation plan to completion. Also use when the user says "bd ready" and picks up tasks that reference a plan — this skill governs how those tasks get implemented.
license: Apache-2.0
---

# Execute Dev Loop

Drive an implementation plan to completion through a structured pipeline: **ingest -> execute -> complete**.

**Announce at start:** "Using execute-dev-loop to drive [plan name] to completion."

**This is a rigid skill.** Follow each phase exactly.

## Prerequisites

- `bd` CLI configured and working
- Plan file exists (typically in `docs/superpowers/plans/`)
- Git repo is clean (`git status` shows no uncommitted changes)

---

## Phase 1: Ingest Plan -> Create Epic + Tasks

### Step 1: Check for Resume

Read the plan file. Look for a `## Pipeline Tracking` section near the top.

**If tracking section found:** Tasks already exist. Skip to Phase 2.

**If no tracking section but tasks might exist:** Search beads for the epic whose description references this plan. `bd search` matches titles by default; use `--desc-contains` to match the plan path stored in the epic's description:
```bash
bd search --desc-contains="<plan-filename>" --type=epic
```
If an epic is found, run `bd children <epic-id>` to enumerate tasks, reconstruct the tracking section in the plan file, then skip to Phase 2.

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

### Step 4: Create Epic

```bash
bd create "<plan-title>" --type=epic \
  --description="Execution tracking for <plan-file-path>"
```

Save the epic ID (e.g., `ghostfin-abc`). The plan path in `--description` is what `bd search "<plan-filename>"` will match on resume (Phase 1, Step 1).

### Step 5: Create Tasks

For each task section, create a beads task. **Use parallel subagents when creating many tasks** for efficiency.

Each task must faithfully represent the plan section — capture:
- The task's purpose and scope
- Files to modify/create/delete
- Implementation steps
- TDD requirements (test commands, expected results, what to test)
- Verification commands (build, lint, type-check)

```bash
bd create "<task-title>" \
  --parent=<epic-id> \
  --skills="superpowers:subagent-driven-development" \
  --description="<faithful-description-from-plan>"
```

**Flag rationale (verified against `bd create --help`):**
- Positional title, `--type=task` (default) and `--priority=2` (default) omitted — drop defaults for brevity.
- `--parent=<epic-id>` establishes the epic→task hierarchical dep. This is the canonical way to tie a task to its epic. Do NOT stuff the epic ID into `--notes` — use `--parent`.
- `--skills="superpowers:subagent-driven-development"` enforces that whoever picks up the task invokes that skill. The flag is a single-string field; if the task needs multiple skills, confirm the delimiter convention in your beads config before stuffing more than one value in.
- Plan-path traceability lives on the **epic's** `--description` (Step 4), not on each task — tasks inherit traceability via `--parent`.

After all tasks are created, wire inter-task dependencies. `bd dep add <A> <B>` means **A depends on B** (A is blocked by B):
```bash
bd dep add <downstream-task> <upstream-task>                     # default type: blocks
bd dep add <downstream-task> <upstream-task> --type=<dep-type>   # semantic type
```

**Three ways to wire deps — pick by shape of the graph:**

| Situation | Mechanism |
|-----------|-----------|
| Linear or few, simple deps | `bd dep add <downstream> <upstream>` post-hoc, one per edge |
| Known at creation, per-task | `--deps "id1,id2"` or `--deps "blocks:id1,discovered-from:id2"` on `bd create` |
| Complex dep graph across many tasks (fan-in, fan-out, dep chains) | Write a JSON plan file and pass it with `bd create --graph=<plan.json>` — creates all issues and edges in one pass |

For plans with non-trivial task graphs (5+ tasks, interleaved deps), prefer `--graph`: one JSON, one command, no risk of partially-wired state between `bd create` and `bd dep add`.

**Dep types (from `bd dep add --help`):** `blocks` (default), `tracks`, `related`, `parent-child`, `discovered-from`, `until`, `caused-by`, `validates`, `relates-to`, `supersedes`. Pick the one that actually describes the relationship — `blocks` is the default but not always the right semantic (e.g., fast-follows are usually `discovered-from`, not `blocks`).

### Step 6: Update Plan File (CRITICAL)

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

**Follow-ups filed during execution (not blocking):**
- (none yet)
```

The **Follow-ups** list starts empty. During Phase 2, Step 5a appends entries here as review findings are filed as fast-follow beads tasks.

Commit the update:
```bash
git add <plan-file>
git commit -m "plan: add pipeline tracking to <plan-name>"
```

---

## Phase 2: Execute Loop

This is the core development loop. Repeat until no open tasks remain.

### Step 1: Check Ready Tasks

Scope queries to this plan's epic so unrelated tasks don't leak into the loop:

```bash
bd ready --parent=<epic-id>
```

- **Ready tasks exist:** Continue to Step 2.
- **No ready tasks, but open tasks exist:** Everything is blocked. Run `bd blocked --parent=<epic-id>` to show what's stuck. Report to the user and stop.
- **No open tasks remain:** Skip to Phase 3 (Complete).

### Step 2: Create Worktree (First Pass Only)

On the first execution pass, decide whether to use a worktree:

- **3+ tasks or plan touches multiple packages:** Create a worktree.
  Invoke **superpowers:using-git-worktrees** with branch name `feature/<plan-slug>`.
  (Derive slug from plan filename: `2026-04-11-post-onboarding-integration.md` -> `post-onboarding-integration`)
- **<3 tasks, single-package, narrow scope:** Work on current branch.

Update the Pipeline Tracking `Worktree` field with the path and branch.

### Step 3: Claim and Group Tasks (CRITICAL — Parallelism is the Default)

Claim all ready tasks:
```bash
bd update <id> --claim
```

**Build the parallel group — this is a required action, not an optional optimization.** For every pair of ready tasks, inspect the **files modified** list in their plan sections. Two tasks are **parallel-safe** iff their file sets are disjoint. Group all parallel-safe tasks together; tasks with file overlap form serial chains.

**Minimum output of this step:** a concrete grouping, e.g.
- Parallel group 1: `{task-A, task-B, task-C}` (no file overlap)
- Serial chain: `task-D -> task-E` (both touch `foo/bar.py`)

You MUST produce this grouping before Step 4. Do not proceed with "I'll figure it out as I go."

**Reconciling with `superpowers:subagent-driven-development`:** That skill's red flag *"Never dispatch multiple implementation subagents in parallel (conflicts)"* is about file conflicts between implementers. The file-overlap check you just performed **is** the conflict check — parallel-safe groups by construction have no conflicts. This skill explicitly overrides that red flag **for file-disjoint groups only**. File-overlapping tasks still run serially.

### Step 4: Implement (Dispatch the Whole Group in One Message)

Invoke **superpowers:subagent-driven-development** to implement. Do not read this as "consider invoking" — the invocation is mandatory. Do **NOT** implement tasks yourself in the main thread; every task goes through a subagent.

**Parallel dispatch rule:** For each parallel group from Step 3, dispatch all implementers **in a single message containing multiple Agent tool calls** (per the system prompt's "multiple tool calls in parallel" guidance). Serial chains are dispatched one task at a time, waiting for each to close before the next. Never dispatch a parallel group sequentially across multiple messages — that defeats the grouping work and wastes wall-clock time.

Each subagent receives:

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
- Collect every finding the review surfaces — do not silently drop any.
- Fix in-scope issues inline (see Step 5a for the triage rule).
- Re-run tests after fixes.

### Step 5a: File Fast-Follow Tasks from Review Findings (CRITICAL)

Every review finding is either **fixed inline** in this pass or **filed as a fast-follow beads task**. Nothing is left as a TODO comment, a chat note, or a "we'll remember it" item.

**Triage rule:**

| Finding looks like... | Action |
|-----------------------|--------|
| Mechanical fix in code just written (typo, missing import, obvious one-liner) within the task's scope | **Inline fix** — apply in this pass, no task |
| Out of scope of the current task (touches other files, other subsystems) | **Fast-follow** |
| Pre-existing issue surfaced by this task (e.g., a brittle test, a latent race) | **Fast-follow** |
| Reveals a spec ambiguity or design question | **Fast-follow** (do not guess an answer under pressure) |
| Non-trivial refactor, architectural hygiene, or perf concern | **Fast-follow** |

When in doubt, file a fast-follow. Traceability beats speed.

**Dep convention.** Task deps in this pipeline are always either **epic-to-task** (`--parent=<epic-id>` at creation) or **inter-task** (`bd dep add <downstream> <upstream>`, or `--deps` at creation, or a whole-graph JSON via `--graph`). `--notes` is free-text metadata and is NOT a dep mechanism — never use it to wire hierarchy.

**For each fast-follow finding:**

1. Create the task in one command — hierarchy, skill, provenance, and priority all on `bd create`:
   ```bash
   bd create "<concise finding summary>" \
     --parent=<epic-id> \
     --skills="superpowers:subagent-driven-development" \
     --deps="discovered-from:<original-task-id>" \
     --priority=3 \
     --description="<what was found, why it matters, where it was surfaced (task N / file / line)>"
   ```
   Save the new task ID. `discovered-from` captures provenance without blocking — the fast-follow lands on `bd ready` immediately.

2. If the fast-follow has a genuine ordering constraint — e.g., it must NOT start until the originating task closes — add a `blocks`-type dep:
   ```bash
   bd dep add <fast-follow-id> <upstream-task-id> --type=blocks
   ```
   Most fast-follows don't need this.

3. Append a row to the **Follow-ups filed during execution (not blocking)** list in the plan's Pipeline Tracking section:
   ```markdown
   - <fast-follow-id> — <concise description matching the bd title>
   ```

**Non-blocking by design.** Fast-follows do NOT gate epic closure. Phase 3 closes the epic when all originally-planned tasks close; open fast-follows remain on the `bd ready` queue for subsequent work. They are executed as fast-follow items in the next dev-loop iteration, or picked up by a later `bd ready` pass.

**Forbidden shortcuts:**
- "I'll just leave a `// TODO` and skip the task." No — file the task.
- "It's small, I can remember it." No — file the task.
- "I'll batch these findings at the end." No — file each one as it is identified.

### Step 6: Close Tasks and Update Plan (CRITICAL)

Close the originally-planned tasks completed in this pass (not the fast-follows — those remain open). `--suggest-next` prints newly-unblocked work, which feeds directly into Step 7:
```bash
bd close --suggest-next <id1> <id2> ...
```

Update the Pipeline Tracking section in the plan file:
- `open` -> `closed` for completed tasks
- Update any in-progress markers
- Confirm every fast-follow filed in Step 5a appears in the **Follow-ups** list

Commit everything:
```bash
git add <changed-files> <plan-file>
git commit -m "<descriptive-commit-message>"
```

### Step 7: Loop

Return to Step 1. The `--suggest-next` output from Step 6 shows what's newly unblocked. Fast-follows filed in Step 5a will surface in `bd ready --parent=<epic-id>` and should be picked up as fast-follow items in this same session before Phase 3, unless the user scopes them to a later run.

---

## Phase 3: Complete

### Step 1: Final Verification

Check the epic's completion state:
```bash
bd epic status <epic-id>
```

- All **originally-planned** beads tasks for this epic are closed
- Fast-follow tasks (from Step 5a) may still be open — that is expected and does not block completion
- All tests pass
- Build is clean
- No uncommitted changes

### Step 2: Update Plan File (CRITICAL)

Update the Pipeline Tracking section to reflect completion. Keep the Follow-ups list intact with any still-open fast-follows so the plan remains the canonical record of what spilled out of execution:

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

**Follow-ups filed during execution (not blocking):**
- <fast-follow-id> — <description>
- <fast-follow-id> — <description>
```

### Step 3: Close Epic

```bash
bd close <epic-id> --reason="All plan tasks complete; N fast-follows tracked separately"
```

Open fast-follow tasks remain on the `bd ready` queue and do not need to close for the epic to close.

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
| Phase 2, Step 5a | (inline — no skill) | File fast-follow beads tasks from review findings |
| Phase 3, Step 4 | `superpowers:finishing-a-development-branch` | Merge/PR/keep decision |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Creating tasks from a plan with no TDD | Invoke writing-plans to add TDD first |
| Creating tasks without updating plan file | Plan file update is **mandatory** after task creation |
| Summarizing task descriptions too aggressively | Copy implementation detail faithfully from the plan |
| Starting implementation before claiming | Always `bd update --claim` first |
| Skipping review for "small" changes | Always review against spec |
| Dropping review findings into TODO comments or memory | Every non-inline finding becomes a beads fast-follow in Step 5a |
| Using `--notes="Epic: ..."` for hierarchy | Hierarchy is `--parent=<epic-id>`; notes is free-text metadata only |
| Using `bd dep add` to wire a task against the epic | Epic relationship is `--parent=<epic-id>` at creation; `bd dep add` is for inter-task deps only |
| Filing a fast-follow without `--parent=<epic-id>` | Without it, the task is orphaned from the epic and will not show up under it |
| Tasks missing `--skills="superpowers:subagent-driven-development"` | Tag every code task with the enforcement skill at creation so the pickup agent invokes it |
| Hand-wiring a complex dep graph one `bd dep add` at a time | For 5+ tasks with non-trivial deps, write a JSON and use `bd create --graph=<plan.json>` — atomic, no partial-wire state |
| Fast-follows missing from the Follow-ups list in the plan | Append `- <bd-id> — <desc>` in the same commit as Step 6 |
| Blocking epic closure on open fast-follows | Fast-follows are non-blocking by design — close the epic when planned tasks close |
| Batching fast-follow filing at end of session | File each one as the review identifies it, not later |
| Forgetting plan file update on completion | Plan must show `complete` status with date |
| Working on blocked tasks | Only work `bd ready` tasks — check every loop iteration |
| Mixing plans in one worktree | One plan per worktree, always |
| Using non-opus model for subagents | Always `model: "opus"` |
| Implementing a task yourself in the main thread because "it's small" | Every task is dispatched to a subagent via `superpowers:subagent-driven-development`. No exceptions. |
| Dispatching file-disjoint tasks serially (one message per subagent) | File-disjoint tasks are a parallel group — dispatch them in a single message with multiple Agent tool calls. |
| Skipping the explicit file-overlap grouping step | Step 3 of Phase 2 is mandatory output. You must produce the parallel groups and serial chains before Step 4. |
