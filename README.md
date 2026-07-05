# cursor-bridge

Use the [Cursor CLI](https://cursor.com/cli) (`agent`) as Claude Code's
execution backend, so Claude spends its own tokens on planning and judgment
while Cursor -- a separate subscription, running as ordinary OS processes --
does the writing-code, running-tests, and reviewing-diffs work.

## Why

A Claude Code Task-tool subagent bills its entire working context (every file
read, every tool call, every iteration) against the calling session's tokens.
A `cursor-dispatch` call is a shell command: Claude pays for composing a short
prompt and reading back a few lines of status, nothing else. If you have a
Cursor subscription that's mostly idle, this turns it into free execution
capacity for anything that doesn't require Claude's own judgment.

This is not Cursor-specific glue for one project -- it's a drop-in dispatch
backend that speaks the exact file-handoff contract used by the
[superpowers](https://github.com/obra/superpowers) plugin's
`subagent-driven-development` and `dispatching-parallel-agents` skills (task
briefs and diffs as files, short structured status back), so it slots in with
zero changes to those skill files. It also works standalone, without
superpowers installed at all.

## Install

1. Install the Cursor CLI and log in:
   ```bash
   curl https://cursor.com/install -fsS | bash
   agent login
   agent status   # should print "Logged in as <you>"
   ```
2. Clone this repo and symlink it into Claude Code's skills directory:
   ```bash
   git clone https://github.com/<you>/claude-cursor-bridge.git ~/claude-cursor-bridge
   ln -s ~/claude-cursor-bridge ~/.claude/skills/cursor-bridge
   ```
   (The repo is named `claude-cursor-bridge`; the skill itself is invoked as
   `cursor-bridge` -- the symlink name above is what Claude Code reads.)
3. (Optional) customize model tiers:
   ```bash
   cp ~/claude-cursor-bridge/config/models.example.env ~/claude-cursor-bridge/config/models.env
   $EDITOR ~/claude-cursor-bridge/config/models.env
   ```
   List what's available to you with `agent --list-models`.
4. Add the routing rule below to your `CLAUDE.md` (global `~/.claude/CLAUDE.md`
   for a blanket default across every project, or a project-level `CLAUDE.md`
   to scope it to one repo).

### CLAUDE.md snippet

```markdown
## cursor-bridge
For all code implementation (writing, editing, or refactoring source code;
fixing bugs; writing tests) in a git repository, delegate to the
`cursor-bridge` skill instead of writing code directly -- whether the request
comes from direct chat or from within a superpowers flow
(`subagent-driven-development`, `dispatching-parallel-agents`,
`executing-plans`, TDD cycles). Never write application code yourself when
`cursor-bridge` can do it.

Exceptions -- handle directly:
- Single-line/trivial fixes (typos, config/env tweaks, renames)
- Non-code work: planning, research, docs, memory/CLAUDE.md updates, git
  operations, running existing tests/builds, answering questions
- No git repo present (delegation requires a worktree)
- You're explicitly asked to write the code yourself

Always keep on Claude: planning, the progress ledger, task-brief authoring,
answering a subagent's questions, and BLOCKED/NEEDS_CONTEXT judgment calls.
Fall back to a real Claude subagent for one task if `cursor-dispatch` reports
BLOCKED twice or the task needs deep repo-wide architectural reasoning.
```

## Usage

The core primitive is `scripts/cursor-dispatch`:

```bash
scripts/cursor-dispatch --role implementer|reviewer|fixer --tier fast|standard|high \
  --brief <file> [--report <file>] [--diff <file>] [--context "<one-liner>"] \
  --workdir <repo> --worktree <slug>
```

- `implementer` / `fixer`: pass `--report <file>`. The command prints only a
  short status block (`Status: DONE | DONE_WITH_CONCERNS | BLOCKED |
  NEEDS_CONTEXT`, commits, one-line test summary, concerns, report path) --
  the detailed report lives in the file, written by the agent itself.
- `reviewer`: pass `--diff <file>` instead. The command's full output IS the
  review -- there's no separate report file for this role.
- Every dispatch runs in an isolated Cursor-managed git worktree
  (`~/.cursor/worktrees/<repo>/<slug>`) -- never your working tree directly.
- Any failure (CLI missing, not logged in, bad JSON, non-zero exit) prints a
  `Status: BLOCKED` line instead of erroring, so a controller can react to it
  the same way it reacts to a stuck Task-tool subagent.

See `SKILL.md` for the full mode table and `docs/integration-superpowers.md`
for the exact superpowers dispatch-point -> command -> tier mapping,
including how to source brief/diff files from that plugin's own
`task-brief` / `review-package` scripts.

## Model tiers

| Tier | Built-in default | Use for |
|---|---|---|
| `fast` | `composer-2.5` | Mechanical, well-specified, 1-2 file tasks |
| `standard` | `composer-2.5` | Multi-file coordination, integration work |
| `high` | `gpt-5.5-high` | Review, and any judgment-heavy task |

Override any of these in `config/models.env` (git-ignored; copy from
`config/models.example.env`). Model names drift over time -- this file is the
only place that needs to change.

## Parallel dispatch

No separate mechanism: fire multiple `cursor-dispatch` calls via the Bash
tool's `run_in_background: true`, one per independent task, in a single turn.
Each is a separate OS process in a separate worktree -- true concurrency, at
the cost of only a few lines of status readback per task instead of a full
subagent context per task.

```bash
cursor-dispatch --role implementer --tier fast --brief a.md --report ra.md \
  --workdir "$repo" --worktree task-a &
cursor-dispatch --role implementer --tier fast --brief b.md --report rb.md \
  --workdir "$repo" --worktree task-b &
wait
```

## Repo layout

```
scripts/cursor-dispatch     core dispatcher: role + tier + brief/report/diff -> agent CLI -> parsed status
scripts/resolve-model       tier name -> concrete --model value (config/models.env, or built-in defaults)
prompts/implementer.md      unattended implementer contract (brief in, report file + short status out)
prompts/task-reviewer.md    unattended reviewer contract (brief + diff in, structured verdict out)
prompts/fixer.md            unattended fix-loop contract (brief + findings in, report file + short status out)
config/models.example.env   tier -> model defaults; copy to models.env to customize
docs/integration-superpowers.md   exact mapping from superpowers dispatch points to cursor-dispatch calls
```

## License

MIT -- see `LICENSE`.
