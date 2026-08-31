---
name: github-issue-orchestrator
description: Orchestrate implementation of one or more GitHub issues with Codex subagents. Use when the user asks Codex to work through GitHub issues/tickets, spawn a worker per issue, create branches or PRs, keep issue/project status current, respect dependencies, or build stacked/compounded PRs. Treat the current Codex session as the sole scheduler and GitHub as durable state. Analyze the dependency graph before spawning workers, launch only ready work, update status before launch, and prevent workers from waiting on or inventing missing prerequisites.
---

# GitHub Issue Orchestrator

Use the current Codex session as the control plane. Delegate implementation to disposable subagents, one issue per worker. Keep scheduling, GitHub state, branch topology, PR topology, verification, and recovery in the orchestrator.

Read these references when needed:

- `references/worker-contract.md` before spawning workers.
- `references/github-state.md` when discovering or changing issue/project status.
- `references/branch-and-pr-topology.md` before creating branches or stacked PRs.
- `references/recovery.md` when continuing an interrupted run or when existing branches/PRs are present.

## Non-negotiable invariants

1. Treat GitHub issues, PRs, comments, relationships, and status fields as durable workflow state.
2. Keep the main Codex session as the only orchestrator. Do not implement substantial ticket work in the orchestrator.
3. Give each worker exactly one issue.
4. Do not let workers choose additional tickets or spawn subagents.
5. Build a dependency graph before spawning any worker.
6. Spawn a worker only when all prerequisites required for that worker's starting branch are available.
7. Never spawn a worker whose next action would be to wait for another worker.
8. Set an issue to the repository/project's semantic `In Progress` state before launching its worker.
9. Workers report results or blockers; the orchestrator owns GitHub status transitions.
10. Verify worker output independently before advancing workflow state.
11. Re-read GitHub after every meaningful transition; do not rely on a stale in-memory plan.
12. Prefer correctness over concurrency. Parallelize only work that is actually independent at that moment.
13. Never invent missing dependency behavior. If a prerequisite is absent or ambiguous, keep the dependent issue unscheduled or mark it blocked with a concrete reason.
14. Make the workflow restartable from a fresh Codex session without relying on prior conversation state.

## Interpret the request

Extract these inputs from the user's request:

- **Issue selection**: explicit numbers, inclusive ranges, a parent issue and its children, or another clearly specified set.
- **Worker model**: if the user specifies a model or effort tier, use it exactly for implementation workers. Otherwise use the current Codex default.
- **Parallelism limit**: use the user's limit if given. Otherwise launch all currently ready independent issues that can safely run in parallel.
- **PR mode**:
  - `dependency-stack` by default: stack only when a dependency requires a branch as its base.
  - `linear-stack` when the user explicitly asks to stack/compound every selected PR into the next one. Treat the requested stack order itself as a dependency and serialize branch creation accordingly.
  - `independent` when the user explicitly requests independent PRs from the target branch.
- **Target branch**: infer the repository default branch unless the user specifies another base.

Do not infer issue dependencies from issue numbers alone.

## Phase 1: Preflight before spawning anything

Complete all preflight work before creating implementation workers.

1. Confirm the repository and GitHub remote.
2. Read every selected issue completely, including acceptance criteria, linked issues, sub-issues, blocking/blocked-by relationships, relevant comments, and existing PR links.
3. Inspect existing branches and open/merged PRs for the selected issues. Reuse valid existing work instead of duplicating it.
4. Read repository instructions such as `AGENTS.md` and determine build/test commands workers must follow.
5. Discover how the repository/project represents workflow status. Follow `references/github-state.md`.
6. Build a dependency DAG for the selected work.
7. Add implicit implementation dependencies only when the code/acceptance criteria clearly require them. Record the evidence; do not guess.
8. Apply PR-mode ordering constraints from `references/branch-and-pr-topology.md`.
9. Classify each issue:
   - `ready`: can start now from an available base.
   - `waiting`: has unfinished prerequisites that are already planned/running.
   - `blocked`: cannot proceed because a prerequisite or required decision is unavailable.
   - `already-in-progress`: valid branch/PR/work exists and should be recovered.
   - `complete`: already satisfies the workflow completion condition.
10. Only after this classification may workers be launched.

When the user asked for several issues, briefly maintain an internal table equivalent to:

| issue | state | prerequisites | branch base | worker |
|---|---|---|---|---|
| #18 | ready | - | main | none |
| #19 | waiting | #18 | issue-18 | none |

Do not publish this table unless useful, but keep the model explicit.

## Phase 2: Schedule ready work

Repeat this scheduling loop until all selected issues are complete or genuinely blocked.

### A. Recompute the ready set

Re-read relevant GitHub state and recompute which issues are ready. A dependency is satisfied only when the code required by the dependent worker is present in that worker's chosen base.

A PR merely existing is not automatically enough. The dependent branch must actually contain the prerequisite implementation.

### B. Reserve each issue before launch

For every issue selected for launch, in this order:

1. Update the issue/project to semantic `In Progress` **before** spawning the worker.
2. Add a concise orchestration comment when useful, stating the branch and prerequisite/base being used.
3. Create or verify the issue branch from the correct base according to `references/branch-and-pr-topology.md`.
4. Create an isolated worktree/workspace when the environment supports it, so concurrent workers cannot mutate the same checkout.
5. Only after steps 1-4 succeed, spawn one implementation worker.

If branch/worktree creation or worker launch fails, restore the issue to the most accurate non-running state and record the failure. Never leave an issue marked `In Progress` for a worker that never started.

### C. Spawn exactly one worker per ready issue

Before spawning, read `references/worker-contract.md` and give the worker:

- exact issue number and full issue requirements;
- repository/worktree path and assigned branch;
- relevant prerequisite context that already exists in the base;
- required tests/build commands from repository instructions;
- explicit instruction to implement only that issue;
- explicit instruction to stop with `BLOCKED` rather than wait, guess, or implement another issue;
- the user's requested worker model/effort, if any.

Do not give a worker the entire unscheduled issue set as a todo list.

### D. Do not poll workers wastefully

After workers are launched, let them work. Do not repeatedly interrupt or interrogate a running worker without a concrete reason. Use native completion/wait mechanisms where available.

If other ready work exists within the concurrency limit, schedule it. Otherwise wait for a meaningful worker result.

## Phase 3: Handle worker results

A worker should return one of these semantic outcomes:

- `DONE`: implementation and required local checks completed on its branch.
- `BLOCKED`: a concrete prerequisite, repository fact, permission, or decision prevents correct implementation.
- `FAILED`: implementation attempt failed for a concrete reason.

Never treat a worker's `DONE` claim as proof of completion.

### On `DONE`

1. Inspect the actual diff and commits.
2. Confirm the work is scoped to the assigned issue.
3. Run or verify the required tests/checks; run additional focused checks when risk warrants it.
4. Verify the branch is based on the intended prerequisite/base.
5. Create or update the PR using the correct base branch.
6. Ensure the PR references/closes the GitHub issue according to repo conventions.
7. Move the issue to semantic `In Review` if that state exists and the PR is awaiting merge/review.
8. If the workflow includes autonomous merge and merge conditions are satisfied, merge it and move the issue to semantic `Done`.
9. If merge is not requested/allowed, do not mark the issue `Done` merely because a PR exists; keep it in the accurate review state.
10. Re-read GitHub and recompute the ready set. Newly satisfied dependents may now launch.

### On `BLOCKED`

1. Inspect whether the blocker is real.
2. If it is an unfinished selected prerequisite, do not let this worker wait. End this worker run and return the issue to a waiting/non-running state; schedule it again only after the prerequisite becomes available.
3. If it is a genuine external blocker, move the issue to semantic `Blocked` when supported and add a concise comment with the exact blocker and required resolution.
4. If the blocker reveals a missing dependency edge, add that edge to the orchestrator's DAG and recompute scheduling.
5. Never allow the worker to invent an API, stub another ticket's behavior, or silently absorb the prerequisite into its own scope.

### On `FAILED`

1. Inspect the failure and branch state.
2. Decide whether a focused retry is appropriate.
3. Preserve useful commits if safe.
4. Keep GitHub status accurate; do not mark failed work as review/done.
5. For retry, launch a fresh worker with the failure evidence and the same single-issue scope.

## Dependency rules

Treat dependency management as an orchestrator responsibility.

An issue is not ready when any of these are true:

- an explicit `blocked by` relationship is unresolved;
- its acceptance criteria require code from another selected issue that is not in its branch base;
- its branch base does not yet contain a required prerequisite implementation;
- the requested linear PR stack requires an earlier stack branch that does not yet exist;
- the worker would need to guess an API/data model/schema that another issue is supposed to establish.

If two issues touch the same fragile area but have no true dependency, prefer serialization when concurrent implementation would create high merge-conflict or semantic-risk overhead.

## GitHub status discipline

Follow `references/github-state.md` for discovery and mapping. At minimum enforce these transition semantics:

```text
not running -> In Progress  BEFORE worker launch
In Progress -> In Review    AFTER verified PR is ready for review
In Review   -> Done         AFTER merge/completion condition
any active  -> Blocked      ONLY for a genuine external blocker
```

Never use a comment such as "starting" as a substitute for changing the actual status field when a status field exists.

Never let a worker own status updates.

## Branch and PR discipline

Follow `references/branch-and-pr-topology.md`.

Key rules:

- Create the branch before worker launch.
- Use one branch per issue.
- Workers implement/commit; the orchestrator owns PR topology and final PR creation/update.
- Stack a dependent PR on a prerequisite branch only when that base contains everything the dependent issue needs.
- Never branch from an unfinished dependency merely because the issue numbers are adjacent.
- For multiple independent prerequisites, wait until a base exists that contains all of them; do not pretend one prerequisite branch contains the others.
- If the user asks to compound every PR linearly, treat the stack order as a dependency and do not launch a downstream worker before its base branch exists.

## Recovery and restartability

When starting with existing `In Progress` issues, branches, worktrees, or PRs, read `references/recovery.md`.

Never assume an old subagent is still alive in a fresh Codex session. Reconstruct state from GitHub and git artifacts.

The skill is behaving correctly when a fresh Codex session can be started in the repo, invoke this skill against the same issue set, and safely continue without duplicating branches, PRs, or work.

## Completion condition

Finish only when every selected issue is in one of these categories:

- completed according to the user's requested lifecycle (for example merged/done);
- has a verified PR awaiting human review when autonomous merge was not requested;
- genuinely blocked with the blocker recorded in GitHub.

Before finishing:

1. Re-read all selected issues and PRs.
2. Correct any stale workflow statuses.
3. Ensure no issue remains `In Progress` without an active or recoverable implementation path.
4. Ensure PR bases match the intended topology.
5. Report completed, review-pending, and blocked issues succinctly.

## Examples

User:

```text
Take GitHub issues 18 to 21. Launch a 5.6 Luna Max subagent for each. For each issue create a new branch and PR, compound the PRs into each other. Keep GitHub issue status up to date.
```

Interpretation:

- Read #18-#21 before spawning anything.
- Because "compound the PRs" explicitly requests a linear stack, determine a valid order from dependencies; when equally valid, preserve issue order.
- Treat each earlier stack branch as a prerequisite for the next worker.
- Set #18 to `In Progress`, create its branch, then spawn its 5.6 Luna Max worker.
- Do **not** spawn #19 merely to wait for #18's branch content.
- After #18 returns and is verified, create/verify PR #18, then create #19's branch from #18's branch, set #19 `In Progress`, and launch #19.
- Continue through the stack, keeping status accurate at each transition.

User:

```text
Work through #31, #32, #33 and #34 with subagents. Parallelize whatever is independent and stack only real dependencies.
```

Interpretation:

- Use `dependency-stack` mode.
- Build the DAG first.
- Launch every currently ready independent issue, subject to safe concurrency.
- Keep dependent issues unlaunched until their base contains all prerequisites.
