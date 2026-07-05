# Integrating with superpowers

`cursor-bridge` does not modify any superpowers skill file. It slots into the
same file-handoff contract those skills already use: a task brief and a
report/diff are files, never inlined into the controller's context, and a
dispatched agent replies with a short structured status. The only thing that
changes is which engine executes the dispatch.

This table maps each superpowers dispatch point to the equivalent
`cursor-dispatch` call.

| Superpowers dispatch point | Brief/diff source | cursor-dispatch call | Default tier | Fall back to a real Claude subagent when |
|---|---|---|---|---|
| `subagent-driven-development` implementer dispatch | `scripts/task-brief PLAN N` (prints brief path) | `--role implementer --report <workspace>/task-N-report.md` | `fast` for mechanical/1-2-file tasks, `standard` for multi-file/integration tasks | Task needs an architectural decision the plan doesn't specify, or has failed twice |
| `subagent-driven-development` task reviewer dispatch | `scripts/task-brief` + `scripts/review-package BASE HEAD` (prints diff path) | `--role reviewer --diff <workspace>/review-*.diff` | `high` | Diff is large/subtle enough that you'd want the most capable model regardless of engine |
| `subagent-driven-development` fix dispatch (after reviewer findings) | Same brief, plus the reviewer's findings text passed via `--context` | `--role fixer --report <workspace>/task-N-fix-report.md` | Same tier as the implementer dispatch it follows | Second BLOCKED on the same task |
| `subagent-driven-development` final whole-branch review | `review-package` across the full range | `--role reviewer --tier high` | `high` (always -- this is the most capable tier regardless of diff size, per that skill's own rule) | Never -- this stays high tier either way; only the engine choice changes |
| `dispatching-parallel-agents` (N independent failures/tasks) | One brief per domain (write ad hoc, or one `task-brief` call per task if plan-backed) | N `cursor-dispatch` calls, one per domain, each via Bash `run_in_background: true` in the same turn | Per-task, using the same complexity signals as implementer dispatch | Any one domain that turns out non-independent after starting -- stop parallel dispatch for that domain and let Claude reason about it directly |
| `executing-plans` (single-session sequential execution, no subagents) | The plan file's task sections | `cursor-dispatch --role implementer` per task, sequentially | `fast`/`standard` per task | Whenever `executing-plans` itself says stop and ask for help |
| `test-driven-development` RED/GREEN/REFACTOR cycle | N/A -- this is baked into `prompts/implementer.md`'s instructions, not a separate dispatch point | N/A | N/A | N/A |

## Workspace paths

Reuse `subagent-driven-development`'s own workspace helper so briefs, reports,
and diffs live in one place and stay out of `git status`:

```bash
ws=$("$SUPERPOWERS_SDD_SCRIPTS/sdd-workspace")   # prints e.g. <repo>/.superpowers/sdd
brief=$("$SUPERPOWERS_SDD_SCRIPTS/task-brief" PLAN.md 3)   # prints the brief path it wrote
```

Then dispatch:

```bash
cursor-dispatch --role implementer --tier fast \
  --brief "$ws/task-3-brief.md" --report "$ws/task-3-report.md" \
  --context "Task 3 of PLAN.md; depends on Task 2's DB schema" \
  --workdir "$(git rev-parse --show-toplevel)" --worktree task-3
```

## Reviewer dispatch

```bash
diff=$("$SUPERPOWERS_SDD_SCRIPTS/review-package" "$BASE_SHA" "$HEAD_SHA")

cursor-dispatch --role reviewer --tier high \
  --brief "$ws/task-3-brief.md" --diff "$diff" \
  --context "Global constraint: all timestamps are UTC (see PLAN.md Global Constraints)" \
  --workdir "$(git rev-parse --show-toplevel)" --worktree task-3-review
```

The reviewer's full structured reply is `cursor-dispatch`'s only output --
print it to the controller as-is, exactly like a Task-tool reviewer
subagent's final message.

## Standalone, without superpowers installed

Generate an equivalent diff package with a plain `git diff`:

```bash
git log --oneline "$BASE..$HEAD" > /tmp/review.diff
echo >> /tmp/review.diff
git diff --stat "$BASE..$HEAD" >> /tmp/review.diff
echo >> /tmp/review.diff
git diff -U10 "$BASE..$HEAD" >> /tmp/review.diff
```

Any plain-text file with the requirements works as a brief -- there's nothing
superpowers-specific about the brief/report/diff file format itself, only the
convention of using files instead of inline context.

## Parallel dispatch, concretely

```bash
cursor-dispatch --role implementer --tier fast --brief brief-a.md --report report-a.md \
  --workdir "$repo" --worktree fix-a &
cursor-dispatch --role implementer --tier fast --brief brief-b.md --report report-b.md \
  --workdir "$repo" --worktree fix-b &
cursor-dispatch --role implementer --tier fast --brief brief-c.md --report report-c.md \
  --workdir "$repo" --worktree fix-c &
wait
```

Issued in the same turn (via the Bash tool's `run_in_background: true`, one
call per task), these run concurrently as separate OS processes in separate
worktrees -- mirroring "multiple dispatch calls in one response = parallel
execution" from `dispatching-parallel-agents`, at zero incremental Claude
context cost beyond reading back each short status line.
