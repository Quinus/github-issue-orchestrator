# GitHub Issue Orchestrator

A [Codex](https://openai.com/codex/) skill that turns your Codex session into an orchestrator for implementing GitHub issues with disposable subagent workers — one worker per issue, GitHub as the durable source of state.

## The concept

Instead of having one agent grind through a list of issues sequentially (or worse, in parallel with no coordination), this skill makes your current Codex session the **control plane**:

- **You** give it a set of issues (e.g. `#18`–`#21`).
- **The orchestrator** (your Codex session) reads every issue, inspects existing branches/PRs, builds a dependency graph, and classifies each issue as `ready`, `waiting`, `blocked`, `already-in-progress`, or `complete`.
- **Workers** (Codex subagents) each get _exactly one_ issue, an isolated branch, and the context they need. They never pick up extra tickets or spawn their own subagents.
- **GitHub is the state store.** Issue/project statuses (`In Progress` → `In Review` → `Done`), comments, branches, and PRs are the workflow's memory — so any interrupted run can be resumed from a brand-new Codex session.

Key guarantees enforced by the skill:

- Dependencies are derived from actual issue content and branch topology, never from issue numbers.
- No worker is ever spawned to "wait" on another worker.
- Status is set to `In Progress` _before_ a worker launches; a worker's `DONE` claim is independently verified before a PR is created.
- Three PR modes: `dependency-stack` (default), `linear-stack` (compound every PR into the next), and `independent`.
- Correctness over concurrency — work is parallelized only when genuinely independent.

## Usage

Once installed, invoke it in a Codex session in your repository with a natural-language request:

```text
Take GitHub issues 18 to 21. Launch a subagent for each. For each issue create
a new branch and PR, compound the PRs into each other. Keep GitHub issue status
up to date.
```

```text
Work through #31, #32, #33 and #34 with subagents. Parallelize whatever is
independent and stack only real dependencies.
```

You can control:

- **Issue selection** — explicit numbers, ranges, or a parent issue with children.
- **Worker model/effort** — e.g. "use 5.6 Luna Max workers".
- **Parallelism** — a concurrency limit, or let all ready issues run.
- **PR mode** — `dependency-stack` (default), `linear-stack`, or `independent`.
- **Target branch** — defaults to the repository's default branch.

The orchestrator runs a scheduling loop: recompute ready work → reserve (status, branch, worktree) → spawn one worker per ready issue → verify results → update GitHub → repeat until every issue is merged, awaiting review, or genuinely blocked (with the blocker recorded on the issue).

## Install with skills.sh

[skills.sh](https://skills.sh) is a skill registry/installer for agent skills. Install this skill globally or per-project with:

```sh
npx skills add https://github.com/Quinus/github-issue-orchestrator
```

or from the registry shorthand:

```sh
npx skills add github-issue-orchestrator
```

This copies the skill (its `SKILL.md`, reference docs, and agent config) into your skills directory so Codex (and other compatible agent CLIs) can discover and invoke it.

## Repository layout

```
skills/github-issue-orchestrator/
  SKILL.md                        # skill entrypoint: orchestration protocol & invariants
  agents/openai.yaml              # subagent configuration
  references/
    worker-contract.md            # what a worker receives and must obey
    github-state.md               # discovering/mapping issue & project status
    branch-and-pr-topology.md     # branching, stacking, and PR rules
    recovery.md                   # resuming interrupted or pre-existing runs
```

## License

MIT
