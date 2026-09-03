# Antigravity Delegation

Antigravity (`agy`) is a separate agentic CLI (Google; Gemini/Claude/GPT-OSS backends).
Claude Code can delegate mechanical execution to it and act as orchestrator.

**Master switch.** Check `test -f ~/.claude/agy-enabled` before delegating. Absent means
delegation is OFF — do the work directly. Toggle with `touch` / `rm`. The wrapper
refuses to run when the switch is off, so this check just saves a wasted invocation.

## When to delegate

Only when a plan is already approved and the remaining work is mechanical execution:
applying a spec'd refactor, scaffolding boilerplate, bulk renames, generating tests
from an existing pattern, reading a batch of documents for stated facts. Not
architecture decisions, not ambiguous judgment calls, and not counting or verification
— a script gets those right and can be re-run, while agy's confidence signal on them
is not trustworthy.

The line can fall *inside* a single output file. Before delegating a templated
multi-file task, classify each mandated heading: **transformation** (restate or
reorganize facts stated in the source) or **judgment** (produce a payoff, rationale or
tradeoff nobody wrote down). Delegate transformation headings. For judgment headings,
either drop them from the plan and write them yourself after the run, or delegate them
separately with the specific facts to synthesize spelled out — a generic prompt
produces an interchangeable fill-in-the-blank skeleton in every file.

## What a good plan file looks like

The wrapper cannot fix a vague plan. The clean runs shared four habits:

- **An output → source mapping table**, one row per output file naming the exact source
  section and file. This is what lets agy split two outputs across one source page
  correctly, because the split is spelled out rather than inferred.
- **An anti-compression sentence for any expansion task**: "Expand, don't summarize.
  The whole output must be between 250 and 400 words. If your output is shorter than
  the source material, you did it wrong." Without it agy compresses by default.
- **A citation rule with the concrete anti-pattern named** — "never `file.md:44-47`"
  beats describing the problem abstractly. The wrapper injects this; if the plan
  repeats it, use the same example.
- **A group-specific facts section per run.** Sequential `agy-delegate` calls, one per
  content group, let each carry only its own facts and stay under the timeout.
  `--batch-inputs` splits one prompt across an input set; it does not express this.

## Running it

Never call `agy` directly — call the `agy-delegate` wrapper. It encodes fixes that are
easy to forget between sessions.

```
agy-delegate PLAN.md --add-dir <scoped-dir> [options]
```

`--add-dir` scopes agy's workspace to the smallest directory the task needs. For
extraction runs point it at a scratchpad copy, never the deliverable.

| Flag | Notes |
| --- | --- |
| `--model "<name>"` | Validated against `agy models`; an unknown name is rejected rather than silently replaced. |
| `--expect-created N` / `--expect-modified N` | The counts the plan declares. Mismatch fails the run. Pass both on every extraction run. |
| `--protect <glob>` | Repeatable, relative to `--add-dir`. A touched protected path fails the run and prints the `git checkout --` that reverts it. |
| `--uncertainty-token "<token>"` | The token the plan invites for "no defensible answer". A zero count is reported, because agy emits it zero times whether or not the inputs are ambiguous. |
| `--batch-inputs "<glob>"` | One `agy` run per batch (size 4–9, default 6). Use for any multi-document job — a silent partial is only visible in a small batch. |
| `--dangerously-skip-permissions` | Only for a run scoped by `--add-dir` that would otherwise stall on agy's permission prompts. Never where it can reach outside that directory or the network. |
| `--print-timeout` | Default `30m`; agy's own 5m default silently kills long runs. |
| `--retries` | Default 1. Ignored for quota, network and auth, which cannot succeed on a retry. |
| `--no-evidence-rules` | Drop the injected quote-not-line-number paragraph. |
| `--ignore-quota-block` | Start against an open quota window. Almost always wrong. |
| `--dry-run` | Print the resolved command without running it. |

Handled automatically, so you don't have to remember it: the master switch and quota
block; model validation; flag order with `-p` last; three injected prompt paragraphs
(anti-generator-script, write-only-the-named-files, cite-a-quote-not-a-line-number); a
git-repo warning on `--add-dir`; a before/after snapshot checked against
`--expect-*` and `--protect`; stray-script and `file:line` detection; a
template-collapse heuristic across 3+ prose outputs; failure classification before
retrying; treating a batch that exits 0 having written nothing as failed.

For a run likely to exceed 5 minutes, invoke with the Bash tool's
`run_in_background: true` — the wrapper is synchronous per invocation.

## Mandatory review step

agy's printed summary and the wrapper's coverage summary are self-reports, not proof.
Read the exit code first: non-zero means an assertion failed, a batch delivered
nothing, or a protected path was touched. Never report a delegated task done on a
non-zero exit, and never let a `CALIBRATION:` line pass unexamined — it names an input
nobody has judged yet. Then read the files the run touched against the plan and the
source before the work reaches anything client-facing.

If `--add-dir` isn't a git repo there's no diff/reset safety net for agy's edits. The
wrapper warns but does not block; `git init` first, or verify by reading changed files.

Every rule above came from a run that failed without one. If you keep an incident log
of your own runs, read it before debugging an unexpected agy failure.
