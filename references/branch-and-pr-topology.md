# Branch and PR Topology

The orchestrator owns branch bases and PR bases. Workers only implement on the branch they are assigned.

## Common branch naming

Follow repository conventions when they exist. Otherwise use a deterministic issue-prefixed name such as:

```text
issue-18-short-slug
```

Reuse an existing valid issue branch instead of creating a duplicate.

## Mode 1: Dependency stack (default)

Stack only where the dependency graph requires it.

Example:

```text
#18 independent
#19 depends on #18
#20 independent
```

Topology:

```text
main
├── issue-18       PR #18 -> main
│   └── issue-19   PR #19 -> issue-18
└── issue-20       PR #20 -> main
```

Scheduling:

- #18 and #20 may run in parallel.
- #19 does not launch until `issue-18` contains the verified prerequisite implementation.

## Mode 2: Linear stack / compound every PR

Use when the user explicitly asks to stack or compound all selected PRs into one another.

Example order #18 -> #19 -> #20 -> #21:

```text
main
└── issue-18       PR #18 -> main
    └── issue-19   PR #19 -> issue-18
        └── issue-20   PR #20 -> issue-19
            └── issue-21   PR #21 -> issue-20
```

The requested stack itself creates ordering dependencies. Do not launch #19 before the verified #18 branch exists; do not launch #20 before the verified #19 branch exists, and so on.

When issue dependencies force a different order than issue numbering, respect the dependency order. If several valid topological orders exist, preserve the user's explicit order or issue-number order as a stable tie-breaker.

Do not run downstream workers early and plan to "fix the stack later" unless the user explicitly asks for speculative parallelism/rebasing.

## Mode 3: Independent PRs

Each issue branches from the target branch and targets the target branch. Launch independent ready issues in parallel.

## Multiple prerequisites

Suppose #30 depends on independent #28 and #29.

Bad topology:

```text
main
├── issue-28
├── issue-29
└── issue-30 from issue-28   # missing #29
```

The #30 worker would be forced to guess or wait for #29.

Choose a base that contains all prerequisites. Usually this means one of:

1. Merge #28 and #29 to the target branch, then branch #30.
2. Use an established integration branch containing both prerequisites when the repository workflow explicitly supports that.
3. If the user requested a linear stack, order prerequisites into that stack so the downstream base actually contains both.

Do not fabricate a dependency branch relationship that does not contain required code.

## PR creation

After a worker returns DONE and verification passes:

1. Ensure the branch contains only intended work plus inherited prerequisite commits.
2. Push the branch if required.
3. Create/update one PR for the issue.
4. Set the PR base according to the selected topology.
5. Link the issue using repository conventions.
6. Record test evidence in the PR body when useful.

If the worker created a PR despite the contract, inspect it and correct/reuse it rather than blindly creating a duplicate.

## Merged bases in a stack

When an earlier stacked PR merges, GitHub may automatically retarget or show changed diffs for downstream PRs. Re-verify downstream PR bases/diffs after each merge and retarget/rebase only when needed to preserve the intended logical changes.
