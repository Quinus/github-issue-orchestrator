# GitHub State Protocol

Use GitHub as the durable execution record. Discover the repository's actual status mechanism before mutating state.

## Discover the status mechanism

For the selected issues, inspect:

1. Native issue status/state fields when present.
2. GitHub Projects fields for projects containing the issues.
3. Existing repository labels only when labels are clearly being used as workflow state.
4. Existing issue/PR conventions and automation.

Prefer the status mechanism already used by the selected issues. Do not create a new project, field, or label vocabulary unless the user asks.

If several projects contain the issue, prefer the project whose status field is already populated consistently for the selected issue set. If ambiguity remains, avoid destructive remapping and use the most local established convention.

## Semantic status mapping

Map repository-specific values to these semantics:

- `Todo` / `Backlog` / `Ready`: not running.
- `In Progress` / `Doing`: a worker is actively implementing or a recoverable implementation run is active.
- `In Review` / `Review`: implementation is verified and a PR awaits review/merge.
- `Done` / `Complete`: completion condition is satisfied.
- `Blocked`: genuine external blocker if the project supports it.

Do not rename repository values. Use the closest established semantic value.

## Transition ownership

Only the orchestrator changes workflow state.

### Before worker launch

Perform these actions in order:

1. Confirm the issue is actually ready.
2. Set semantic status to `In Progress`.
3. Verify the mutation succeeded.
4. Create/verify branch and worktree.
5. Launch the worker.

If step 4 or 5 fails, revert to the most accurate non-running state and record the failure.

### Worker complete, PR ready

After independently verifying implementation and tests:

1. Create/update the PR.
2. Verify correct PR base and issue linkage.
3. Set semantic status to `In Review` if that state exists.

Do not mark `Done` merely because the worker says DONE or because a PR was opened.

### Merge/completion

After the requested completion condition is satisfied, set semantic status to `Done`.

### Blocker

Use semantic `Blocked` only for a real blocker that cannot be resolved by simply waiting for an already-running selected prerequisite.

For a selected prerequisite that is still executing, the dependent issue should normally remain in its non-running/ready state and should not have a worker yet.

## Comments

Use comments for durable explanations, not as a substitute for status fields.

Useful comments include:

- orchestration start with assigned branch/base;
- exact external blocker;
- recovery/retry explanation;
- important change to dependency interpretation.

Avoid noisy heartbeat comments.

## Accuracy invariant

At any point, a human looking only at GitHub should be able to distinguish:

- work not started;
- work actively being implemented;
- work ready for review;
- completed work;
- genuine blockers.

Before ending a run, correct stale `In Progress` states whose worker no longer exists and whose branch/PR does not represent recoverable active work.
