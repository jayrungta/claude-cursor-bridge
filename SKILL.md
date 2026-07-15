---
name: cursor-bridge
description: Delegate code implementation, review, and fix work to the Cursor CLI (agent) instead of a Claude Task-tool subagent, to keep Claude's own context/token usage for planning and judgment. Use for any implementation or fix dispatch (git repo or plain directory); review dispatch requires git, since it reviews a diff -- standalone, or as the execution backend for superpowers' subagent-driven-development, dispatching-parallel-agents, and executing-plans.
---

# cursor-bridge

Cursor CLI (`agent`) as Claude Code's execution backend. Claude stays the
orchestrator -- planning, tracking progress, judgment calls, talking to the
user. Execution (writing code, running tests, reviewing diffs) runs as a
`cursor-agent` OS process against a separate Cursor subscription, not as a
Claude Task-tool subagent that bills Claude's own context and tokens.

## Why this exists

A Task-tool subagent's entire working context -- every file it reads, every
tool call it makes, every iteration -- is billed against the calling Claude
session. A `cursor-dispatch` call is a backgroundable shell command: Claude
pays only for composing a short prompt and reading back a few lines of
status. If you're deciding whether a piece of execution work needs Claude's
own judgment or can run on a separate subscription instead, default to the
latter.

## Modes

| Mode | Role | Default tier | What it does |
|---|---|---|---|
| Implement | `implementer` | `standard` (or `fast` for mechanical tasks) | Writes code + tests for one task, commits, self-reviews, reports back |
| Review | `reviewer` | `high` | Reads a task brief + diff package, returns a structured spec/quality verdict |
| Fix | `fixer` | same as the implementer dispatch it follows | Addresses reviewer findings, re-tests, commits |
| Parallel | any of the above, N times | per-task | Multiple independent dispatches fired in one turn via backgrounded Bash calls |

## Core primitive: `scripts/cursor-dispatch`

```
scripts/cursor-dispatch --role implementer|reviewer|fixer --tier fast|standard|high \
  --brief <file> [--report <file>] [--diff <file>] [--context "<one-liner>"] \
  --workdir <repo> --worktree <slug>
```

- `implementer` / `fixer` require `--report` (the file they must write their
  detailed report to). The command prints only the short status block back.
- `reviewer` requires `--diff` instead of `--report` -- its reply IS the
  report (matches the convention that a reviewer subagent's final message is
  the report itself, no separate file).
- If `--workdir` is a git repository, every dispatch runs inside an isolated
  Cursor-managed git worktree (`~/.cursor/worktrees/<repo>/<slug>`), never the
  working tree directly. Exception: if `--workdir` already points inside a
  linked worktree (e.g. a `fixer` continuing the same task an `implementer`
  dispatch started), the script runs there directly instead of creating
  another one.
- If `--workdir` is NOT a git repository, `implementer`/`fixer` dispatches
  still run -- directly in that directory, with no worktree isolation and no
  branch/commit safety net. `reviewer` requires git (it reviews a diff) and
  will report BLOCKED in a non-git directory.
- On any failure (CLI missing, not logged in, bad JSON, non-zero exit), the
  script prints a `Status: BLOCKED` response instead of erroring out, so a
  controller's existing BLOCKED-handling logic applies unchanged.

Brief, report, and diff files are passed as **paths only** -- their contents
are never pasted into the dispatch command or its output. That's what keeps
this cheap.

## How to use this skill

**As the execution backend for superpowers** (recommended if
`subagent-driven-development` is installed): see
`docs/integration-superpowers.md` for the exact dispatch-point -> command ->
model-tier mapping. In short: everywhere that skill's docs say "dispatch a
Task-tool subagent," dispatch `cursor-dispatch` instead, using the brief/
report/diff file paths that skill's own scripts (`task-brief`,
`review-package`) already produce.

**Standalone, without superpowers:**
1. Write the task's full requirements to a brief file (a plan section, an
   issue body, a paragraph of context -- anything a fresh agent with no
   conversation history needs to do the work).
2. Pick a report file path (e.g. `.cursor-bridge/task-1-report.md`) if
   dispatching implementer/fixer.
3. For review, generate a diff package first (see
   `docs/integration-superpowers.md` for the one-liner using `git diff`, or
   reuse `review-package` if superpowers is installed).
4. Run `cursor-dispatch`. Read only the short reply. Open the report file
   yourself only if status is anything other than `DONE`.

**Parallel dispatch:** fire multiple `cursor-dispatch` invocations via the
Bash tool's `run_in_background: true`, one per independent task, in a single
turn. No separate mechanism needed -- see the README for the exact pattern.

## Model tiers

Tiers are resolved by `scripts/resolve-model` from `config/models.env` (copy
`config/models.example.env` to create it). Built-in defaults: `fast` and
`standard` -> `composer-2.5`, `high` -> `gpt-5.6-sol-high`. Use `fast`/`standard`
for implementation, `high` for review and for architecture-adjacent
judgment calls.

## When NOT to delegate here

- The task needs Claude's own judgment: planning, the progress ledger,
  answering a dispatched agent's questions, deciding what BLOCKED means next.
- Single-line/trivial fixes -- delegating costs more than doing it directly.
- `cursor-dispatch` reports BLOCKED twice on the same task, or the task needs
  deep repo-wide architectural reasoning -- fall back to a real Claude
  subagent for that one task.
