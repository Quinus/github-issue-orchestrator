# Worker Contract

Read this before spawning every implementation worker.

The worker is an executor, not a scheduler.

## Required worker scope

Give the worker exactly one GitHub issue and one assigned branch/worktree.

The worker must:

1. Read the assigned issue and relevant repository instructions.
2. Inspect the prerequisite code already present in its base.
3. Implement only the assigned issue.
4. Add or update tests needed for the assigned issue.
5. Run the required focused checks.
6. Commit the implementation on the assigned branch when the environment/workflow expects commits.
7. Return a structured result to the orchestrator.

The worker must not:

- select or start another GitHub issue;
- spawn another agent;
- change GitHub project/issue workflow status;
- create a different branch topology;
- wait for another worker to finish;
- repeatedly poll another branch, issue, PR, or worker;
- implement an unfinished prerequisite issue;
- invent a missing API, schema, behavior, or contract that belongs to another issue;
- broaden scope merely to make its own issue possible;
- merge PRs.

## Block instead of waiting or guessing

If a prerequisite is missing or incompatible, stop work and return `BLOCKED` immediately.

A good blocker is concrete:

```text
BLOCKED
Issue: #20
Reason: #18 is expected to expose FooRepository.createFoo(), but the assigned base does not contain that API.
Needed: prerequisite implementation providing that API, or an updated issue contract.
Work preserved: branch issue-20-foo contains no speculative implementation.
```

A bad response is:

```text
I am waiting for #18.
```

A worse response is silently inventing the expected dependency.

## Result format

Ask the worker to finish with one of these forms.

### DONE

```text
DONE
Issue: #<n>
Summary: <what changed>
Tests: <commands and result>
Commits: <commit ids or concise description>
Notes: <important follow-up, or none>
```

### BLOCKED

```text
BLOCKED
Issue: #<n>
Reason: <specific blocker>
Needed: <what must become true>
Work preserved: <branch/worktree state>
```

### FAILED

```text
FAILED
Issue: #<n>
Reason: <specific failure>
Attempts: <what was tried>
Work preserved: <branch/worktree state>
```

## Suggested worker instruction

Adapt this to the repository and requested model:

```text
You are the implementation worker for GitHub issue #<n> only.
Work only in the assigned worktree/branch: <branch>.
Implement the issue completely and follow AGENTS.md/repository instructions.
Run the required tests/checks.
Do not work on any other issue. Do not spawn agents. Do not change GitHub workflow status or PR topology.
If required prerequisite behavior is missing from your assigned base, do not wait and do not invent it. Stop and return BLOCKED with the exact missing dependency.
When finished, return DONE/BLOCKED/FAILED using the worker contract.
```
