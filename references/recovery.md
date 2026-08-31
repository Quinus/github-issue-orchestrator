# Recovery and Restart

Use this protocol when the orchestration session was interrupted or when selected issues already have statuses, branches, worktrees, commits, or PRs.

## Reconstruct state from artifacts

For each selected issue, inspect:

- current issue/project status;
- issue comments relevant to orchestration;
- linked/open/merged PRs;
- remote and local branches named for the issue;
- commits/diffs on those branches;
- current PR base/target;
- CI/test state when available;
- explicit dependency relationships.

Do not assume a subagent from a prior Codex session still exists.

## Classify recovered work

### Verified PR exists

If a correct PR exists with expected implementation and base, treat the issue as review/complete according to merge state. Do not spawn another implementation worker.

### Branch has plausible unfinished work, no PR

Inspect the diff and commits. If work is safely resumable, launch a fresh worker on that same branch with explicit context about what already exists.

### Status says In Progress but no recoverable work exists

Treat the status as stale. Move it to the most accurate non-running state and record a concise recovery note if needed. Then reschedule normally.

### Worker previously blocked

Re-check the blocker. If the prerequisite now exists, clear the semantic blocked state and schedule a fresh worker. Do not assume the old worker resumes.

### Duplicate branches or PRs

Do not create more duplicates. Determine which artifact is authoritative from issue links, commit history, and repository conventions; reuse or close superseded artifacts only when safe and within the user's requested authority.

## Restartability test

A successful orchestration protocol satisfies this test:

1. Stop the current Codex session after any issue transition.
2. Start a fresh Codex session in the repository.
3. Invoke this skill for the same issue set.
4. The new session reconstructs the current DAG and work state from GitHub/git.
5. It neither duplicates completed work nor assumes nonexistent workers are active.
6. It continues with only currently ready work.
