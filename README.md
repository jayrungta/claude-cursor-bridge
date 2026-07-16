# cursor-bridge

A guide to using the [Cursor CLI](https://cursor.com/cli) (`agent`) as Claude
Code's execution backend — so Claude spends its tokens on planning and
judgment, while Cursor does the writing-code, running-tests, and
reviewing-diffs work on a separate subscription.

## Who this is for

You already use **Claude Code** for planning and orchestration, and you have
(or want) a **Cursor** subscription whose agent capacity is mostly idle.
`cursor-bridge` turns that idle capacity into execution: Claude dispatches
work as short shell commands instead of burning its own context on every file
read and tool call.

It works standalone, and also slots into the
[superpowers](https://github.com/obra/superpowers) plugin's
`subagent-driven-development` / `dispatching-parallel-agents` flows with no
changes to those skill files.

## Prerequisites

| Requirement | Why |
|---|---|
| **[Claude Code](https://claude.ai/code)** with an active Claude subscription (Pro/Max or API) | Claude is the orchestrator — planning, briefs, judgment. Without it there's nothing to dispatch from. |
| **[Cursor](https://cursor.com)** subscription with CLI access (Pro / Business / Ultra) | Dispatched work runs against Cursor's models and usage limits, not Claude's. A free Cursor account won't get you useful agent runs. |
| **Cursor CLI** (`agent`) installed and logged in | The dispatcher shells out to `agent`. See [Install](#install). |
| **macOS or Linux** with bash, git, and a normal shell | Scripts assume a Unix-like environment. |
| **(Optional) [superpowers](https://github.com/obra/superpowers)** | Not required. If installed, `cursor-bridge` replaces Task-tool subagents at the same dispatch points. |

**Best results:** use both paid subscriptions. Claude for planning and
decision-making; Cursor for mechanical and multi-file execution. That's the
whole point — two subscriptions doing different jobs, neither sitting idle.

## How it works (mental model)

1. You (or Claude) write a **task brief** — a self-contained file a fresh
   agent can act on with no chat history.
2. Claude runs `scripts/cursor-dispatch` with a role (`implementer`,
   `reviewer`, or `fixer`), a model tier, and paths to the brief / report /
   diff.
3. Cursor's `agent` runs in an isolated git worktree (when the target is a
   git repo), does the work, and writes a short status back.
4. Claude reads the status (a few lines), not the agent's full tool trace,
   and decides what to do next.

Claude pays for composing the prompt and reading the status. Cursor pays for
everything in between.

| Mode | Role | Default tier | What it does |
|---|---|---|---|
| Implement | `implementer` | `standard` (or `fast` for mechanical tasks) | Writes code + tests, commits, self-reviews, reports back |
| Review | `reviewer` | `high` | Reads brief + diff, returns a structured verdict |
| Fix | `fixer` | same as the implementer it follows | Addresses reviewer findings, re-tests, commits |

## Install

### 1. Install and log into the Cursor CLI

```bash
curl https://cursor.com/install -fsS | bash
agent login
agent status   # should print "Logged in as <you>"
```

If `agent` isn't on your `PATH` after install, open a new shell or follow the
installer's PATH hint. `agent status` must show you as logged in before any
dispatch will succeed.

### 2. Clone this repo and register the skill

```bash
git clone https://github.com/<you>/claude-cursor-bridge.git ~/claude-cursor-bridge
ln -s ~/claude-cursor-bridge ~/.claude/skills/cursor-bridge
```

The repo is named `claude-cursor-bridge`; the skill Claude Code invokes is
`cursor-bridge`. The symlink name is what Claude Code reads.

### 3. (Optional) Customize model tiers

```bash
cp ~/claude-cursor-bridge/config/models.example.env ~/claude-cursor-bridge/config/models.env
$EDITOR ~/claude-cursor-bridge/config/models.env
```

See what's available on your account with `agent --list-models`. Model names
drift over time — `config/models.env` is the only place that needs to change.

### 4. Tell Claude Code to prefer the bridge

Add this to your global `~/.claude/CLAUDE.md` (all projects) or a
project-level `CLAUDE.md` (one repo):

```markdown
## cursor-bridge
For all code implementation (writing, editing, or refactoring source code;
fixing bugs; writing tests) and other mechanical execution work (batch file
operations, running one-off commands, media/data processing) -- whether or
not the target directory is a git repository -- delegate to the
`cursor-bridge` skill instead of doing it directly, whether the request comes
from direct chat or from within a superpowers flow
(`subagent-driven-development`, `dispatching-parallel-agents`,
`executing-plans`, TDD cycles). Never do this kind of work yourself when
`cursor-bridge` can do it.

Exceptions -- handle directly:
- Single-line/trivial fixes (typos, config/env tweaks, renames)
- Non-code work: planning, research, docs, memory/CLAUDE.md updates, git
  operations, running existing tests/builds, answering questions
- You're explicitly asked to write the code yourself

Always keep on Claude: planning, the progress ledger, task-brief authoring,
answering a subagent's questions, and BLOCKED/NEEDS_CONTEXT judgment calls.
Fall back to a real Claude subagent for one task if `cursor-dispatch` reports
BLOCKED twice or the task needs deep repo-wide architectural reasoning.
```

### 5. Smoke-check

From any git repo, ask Claude Code something like: *"Use cursor-bridge to add
a trivial comment to README and report back."* You should see a
`cursor-dispatch` shell call, then a short `Status: DONE` (or similar) block —
not Claude editing the file itself.

## Day-to-day usage

Once installed, you mostly talk to Claude normally. With the CLAUDE.md rule
above, Claude should route implementation work through the bridge on its own.

When you (or Claude) need to dispatch manually, the core command is:

```bash
scripts/cursor-dispatch --role implementer|reviewer|fixer --tier fast|standard|high \
  --brief <file> [--report <file>] [--diff <file>] [--context "<one-liner>"] \
  --workdir <repo> --worktree <slug>
```

- **`implementer` / `fixer`:** pass `--report <file>`. The command prints only
  a short status block (`Status: DONE | DONE_WITH_CONCERNS | BLOCKED |
  NEEDS_CONTEXT`, commits, one-line test summary, concerns, report path). The
  detailed report is written by the agent into that file.
- **`reviewer`:** pass `--diff <file>` instead. The command's full output *is*
  the review — no separate report file.
- **Git repos:** each dispatch runs in an isolated Cursor-managed worktree
  under `~/.cursor/worktrees/<repo>/<slug>`, never your working tree
  directly. Exception: if `--workdir` already points inside a linked
  worktree (e.g. a fixer continuing an implementer's task), it runs there
  instead of creating another.
- **Non-git directories:** `implementer` / `fixer` still run, directly in that
  directory (no isolation, no branch/commit safety net). `reviewer` requires
  git and will report `BLOCKED` otherwise.
- **Failures** (CLI missing, not logged in, bad JSON, non-zero exit) print
  `Status: BLOCKED` instead of crashing, so Claude can handle them like a
  stuck subagent.

### Standalone (no superpowers)

1. Write the task requirements to a brief file (a plan section, issue body,
   or a paragraph of context — anything a fresh agent needs).
2. Pick a report path (e.g. `.cursor-bridge/task-1-report.md`) for
   implementer/fixer.
3. For review, build a diff package first (see
   `docs/integration-superpowers.md` for a plain `git diff` one-liner).
4. Run `cursor-dispatch`. Read the short reply. Open the report file only if
   status isn't `DONE`.

### With superpowers

See [`docs/integration-superpowers.md`](docs/integration-superpowers.md) for
the exact dispatch-point → command → tier mapping. Short version: wherever
that skill says "dispatch a Task-tool subagent," run `cursor-dispatch`
instead, using the brief/report/diff paths that skill's own `task-brief` /
`review-package` scripts already produce.

### Parallel dispatch

Fire multiple independent tasks in one turn — each is a separate OS process
in a separate worktree:

```bash
cursor-dispatch --role implementer --tier fast --brief a.md --report ra.md \
  --workdir "$repo" --worktree task-a &
cursor-dispatch --role implementer --tier fast --brief b.md --report rb.md \
  --workdir "$repo" --worktree task-b &
wait
```

From Claude Code, use the Bash tool with `run_in_background: true` per call
in a single turn. Claude only pays for reading each short status line back.

## Model tiers

| Tier | Built-in default | Use for |
|---|---|---|
| `fast` | `composer-2.5` | Mechanical, well-specified, 1–2 file tasks |
| `standard` | `composer-2.5` | Multi-file coordination, integration work |
| `high` | `gpt-5.6-sol-high` | Review, and any judgment-heavy task |

Override in `config/models.env` (git-ignored; copy from
`config/models.example.env`).

## What stays on Claude vs Cursor

**Delegate to Cursor:** writing/editing/refactoring code, fixing bugs,
writing tests, batch file ops, mechanical execution.

**Keep on Claude:** planning, progress tracking, authoring task briefs,
answering a dispatched agent's questions, deciding what `BLOCKED` /
`NEEDS_CONTEXT` means next, trivial one-liners, docs/research/git ops,
answering your questions.

**Fall back to a Claude subagent** for one task if `cursor-dispatch` reports
`BLOCKED` twice, or the work needs deep repo-wide architectural reasoning.

## Things that surprise new users

- **Worktrees are isolation by convention, not a hard sandbox.** The agent
  has full shell access. Briefs that say `cd /path/to/repo && …` can land
  commits on the main checkout instead of the worktree. If status says
  `DONE` but the worktree branch has no new commit, check `git reflog` on
  the main repo.
- **Dependent tasks need merges.** A new worktree is always based on
  `--workdir`'s current HEAD. If Task 2 needs Task 1's files, fast-forward
  merge Task 1 into the main branch before dispatching Task 2.
- **Bootstrapping a brand-new repo can't be dispatched.** Creating the
  directory and `git init` must happen first (no `HEAD` to branch from).
  Do that one step directly, then dispatch the rest.
- **`agent` must stay logged in.** If dispatches start returning `BLOCKED`,
  run `agent status` / `agent login` before debugging anything else.
- **Usage is billed to Cursor.** Heavy parallel dispatches burn Cursor
  quota, not Claude tokens. Watch your Cursor usage dashboard if you fire
  many high-tier reviews at once.

## Repo layout

```
scripts/cursor-dispatch     core dispatcher: role + tier + brief/report/diff → agent CLI → status
scripts/resolve-model       tier name → concrete --model value
prompts/implementer.md      unattended implementer contract
prompts/task-reviewer.md    unattended reviewer contract
prompts/fixer.md            unattended fix-loop contract
config/models.example.env   tier → model defaults; copy to models.env to customize
docs/integration-superpowers.md   superpowers dispatch-point mapping
SKILL.md                    skill entry Claude Code loads
```

## License

MIT — see `LICENSE`.
