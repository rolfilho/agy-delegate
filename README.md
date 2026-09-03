# agy-delegate

A wrapper that lets Claude Code delegate bounded, mechanical work to
[Antigravity](https://antigravity.google) (`agy`, Google's agentic CLI) and act as
orchestrator.

Raw `agy -p "..."` calls fail in repeatable ways: they exit 0 having written nothing,
write a generator script instead of the answer, accept a model name they don't have
and silently substitute another, burn a whole batch queue against an exhausted quota,
and edit files the plan never named. `agy-delegate` turns each of those into a
detected, named, non-zero exit.

| Guard | Failure it catches |
| --- | --- |
| Master switch (`agy-enabled`) | Delegation running when you meant it off |
| Model validation against `agy models` | A typo'd name silently replaced by a default |
| Quota block with recorded reset time | A batch queue burned against an open 429 window |
| Failure classification | Retrying quota, network or auth errors that cannot succeed |
| Pre/post directory snapshot | Coverage you'd otherwise take from agy's self-report |
| `--expect-created` / `--expect-modified` | The summary being a report instead of an assertion |
| `--protect <glob>` | agy tidying a state file the plan never named |
| Stray-script detection | agy writing a generator it cannot run, delivering nothing |
| Silent-partial retry | A batch exiting 0 having written nothing |
| `file:line` citation check | Fabricated line numbers presented as evidence |
| Template-collapse heuristic | The same generic skeleton returned across many files |

A non-zero exit means one of those is unclean. Every run appends a JSON line to
`agy-runs.jsonl`.

Each guard is there because a run failed without it. Some of what that looked like:
exit 0 after delivering 36 of 48 modules; whole batches exiting 1 because agy wrote a
Python generator that headless mode then refused to run, leaving the script on disk and
no output written; every run for weeks silently falling back to a different model,
visible only in agy's own logs as `Model ID <name> not in local config, defaulting to
<fallback>`; a cluster of runs fired into an open quota window, each spending a call to
learn what the previous call said. Two runs on real work modified files their plan
declared they wouldn't — which is what `--protect` and `--expect-modified` now catch.

One limit no guard fixes: agy does not emit uncertainty. An explicitly-invited
"not determined" answer came back zero times across dozens of items that included
genuinely ambiguous ones, twice, on two different task types. Never read its absence as
confidence.

## Install

Needs Python 3.8+ (standard library only) and the `agy` CLI on `PATH`, already
authenticated — this wrapper never handles login, so run `agy` once interactively
first.

```bash
install -m 755 agy-delegate ~/.claude/bin/agy-delegate   # or anywhere on PATH
cp AGY.md ~/.claude/
touch ~/.claude/agy-enabled                              # master switch, off by default
```

Reference `AGY.md` from your `CLAUDE.md`, or leave it in `~/.claude/` for the agent to
read on demand — it's the operating manual the agent follows. State files live beside
it, or in `$CLAUDE_CONFIG_DIR` when that variable is set.

Two defaults are mine, not yours. `DEFAULT_PROTECTED` in the script names one
project's state files (`03_State/*`, `MEMORY.md`, `AGENTS.md`, `CLAUDE.md`,
`.claude/*`) — change it to whatever in your project should never be silently edited.
`DEFAULT_MODEL` is a Gemini Flash tier; run `agy models` for what your account reaches.

## Use

```bash
agy-delegate PLAN.md --add-dir ./scratch --expect-created 8 --expect-modified 0
```

`--dry-run` prints the resolved `agy` command without running it. `AGY.md` has the
full flag reference and the rules for what's worth delegating.

The coverage summary is a starting point for verification, not a substitute. Read the
files a run touched against the plan and the source before anything delegated reaches
a deliverable.

## License

MIT — see `LICENSE`.
