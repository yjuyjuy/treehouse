# AXI retrofit spec: treehouse

This document audits the installed `treehouse` binary against the ten AXI principles and records the smallest set of changes that would bring it to the bar.
It is a specification only.
No part of the retrofit is implemented here; a separate follow-on ticket performs the work.

Audited binary: `/root/.local/bin/treehouse`, version `v2.1.0`.
Every verdict below is backed by a transcript captured against that binary.
All mutating commands were run inside a throwaway repository at `/root/.jcode/scratch/dev54/repo`, created with `git init` and `treehouse init`, so no pool belonging to a live agent was touched.
`TREEHOUSE_NO_UPDATE_CHECK=1` was set for most transcripts to keep the update nag out of the captured output; where it was not set, the nag is shown because it is itself audit evidence.

## Scorecard

### 1. Token-efficient output - gap

Structured output is JSON, not TOON, and it is only available on two commands.

```
$ treehouse status --json
[{"name":"1","path":"/root/.treehouse/repo-1a4302/1/repo","status":"leased","lease_id":"cbe6e816e6a7cb02bdcbd868b20bd07b","lease_holder":"","leased_at":"2026-09-02T21:44:42.069425206Z","processes":[]},{"name":"2","path":"/root/.treehouse/repo-1a4302/2/repo","status":"leased","lease_id":"5d261d6198563e90a190f09705a70658","lease_holder":"","leased_at":"2026-09-02T21:44:42.099187449Z","processes":[]}]
```

The human default is a bare fixed-width table with an emoji prefix, and it carries no field names at all.

```
$ treehouse status
1     leased       ~/.treehouse/repo-1a4302/1/repo
2     leased       ~/.treehouse/repo-1a4302/2/repo  (held by agent-x)
3     leased       ~/.treehouse/repo-1a4302/3/repo
```

`prune`, `destroy`, `get` without `--lease`, `enter`, and `return` have no structured output at all.

### 2. Minimal default schemas - gap

The default human view is minimal in the wrong way: it drops field names entirely, so an agent must infer that column two is a status and column three is a path.
The `--json` view is the opposite problem, repeating `lease_id`, `lease_holder`, `leased_at`, and an empty `processes` array on every row whether or not the slot is leased.

```
$ treehouse status --json
[{"name":"1","path":"...","status":"leased","lease_id":"cbe6e816...","lease_holder":"","leased_at":"2026-09-02T21:44:42.069425206Z","processes":[]}, ...]
```

There is no `--fields` flag on any command to select a wider or narrower schema.

```
$ treehouse status --help
Flags:
  -h, --help   help for status
      --json   Print pool status as JSON
```

### 3. Content truncation - gap

There is no truncation contract and no `--full` escape hatch anywhere in the CLI.
Variable-length content is emitted whole: `status` prints one line per process inside each in-use worktree, and the pool list itself is unbounded up to `max_trees`.

```
$ treehouse status
1     in-use       ~/.treehouse/treehouse-46c1ed/1/treehouse
                   zsh (3829755)
```

`prune -v` is the designed escape hatch for detail, but it is a verbosity switch that changes diagnostics rather than a truncation preview with a size hint, and on this pool it produced output identical to the non-verbose run.

```
$ treehouse prune -v
🌳 Dry run: would prune 1 stale worktree and reclaim 98 B.
2     98 B  ~/.treehouse/repo-1a4302/2/repo
🌳 Re-run with --yes to delete these worktrees.
```

### 4. Pre-computed aggregates - gap

`status` prints rows and nothing else: no total, no per-status counts, no `max_trees` capacity figure.
An agent deciding whether an acquire will succeed has to count the rows itself and then read `treehouse.toml` for the cap.

```
$ treehouse status
1     leased       ~/.treehouse/repo-1a4302/1/repo
2     leased       ~/.treehouse/repo-1a4302/2/repo  (held by agent-x)
3     leased       ~/.treehouse/repo-1a4302/3/repo
```

The one place an aggregate does appear is the pool-exhausted error, which proves the numbers are available and simply are not surfaced on the success path.

```
$ treehouse get --lease
all 1 worktrees are in use or dirty (max_trees = 1). Run 'treehouse status' to see details, or increase max_trees in treehouse.toml
```

`prune` does better and pre-computes both a candidate count and reclaimable bytes, which is the shape the rest of the CLI should follow.

### 5. Definitive empty states - partial pass

The empty pool does produce an explicit statement rather than silence, which is the right instinct.

```
$ treehouse status
🌳 No worktrees in pool.
```

The verdict is not a full pass for two reasons that the channel test exposes.
The message goes to stderr, so an agent reading stdout sees nothing at all.

```
$ treehouse status 2>/dev/null
$ treehouse status 2>&1 1>/dev/null
🌳 No worktrees in pool.
```

And the JSON path answers the same question with a bare `[]`, which carries no statement of what was searched.

```
$ treehouse status --json
[]
```

### 6. Structured errors and exit codes - gap

Errors are unstructured single lines on stderr, and the exit code does not distinguish usage errors from operational failures.
Every failure observed exits `1`; AXI reserves `2` for usage errors such as an unknown flag or a missing required argument.

```
$ treehouse status --stat
unknown flag: --stat                     # exit 1, on stderr, stdout empty
$ treehouse lease
unknown command "lease" for "treehouse"  # exit 1
$ treehouse return
worktree /root/.jcode/scratch/dev54/repo is not managed by treehouse   # exit 1
$ treehouse get --json
--json requires --lease                  # exit 1
```

Unknown flags and unknown commands are correctly rejected by name rather than silently ignored, which satisfies the fail-loud half of the principle.
What is missing is the self-correcting hint: none of the errors above lists the command's valid flags or suggests the command that fixes the problem.

Raw dependency output leaks through untranslated, naming `git` and quoting its message verbatim.

```
$ cd /tmp && treehouse status
not in a git repository: git rev-parse --show-toplevel: fatal: not a git repository (or any of the parent directories): .git
```

Idempotency is genuinely good: returning an already-returned worktree is a no-op that exits `0`.

```
$ treehouse return /root/.treehouse/repo-1a4302/2/repo   # exit 0
$ treehouse return /root/.treehouse/repo-1a4302/2/repo   # exit 0, again
🌳 Worktree returned to pool.
```

Non-interactivity is satisfied on the lease path but not on the default path.
`get --lease` completes with flags alone, while a bare `treehouse` opens an interactive subshell, so it is only safe for an agent to run with stdin closed.

### 7. Ambient context - gap

There is no session integration and no installable skill.
The CLI exposes no setup command that would install one.

```
$ treehouse --help
Available Commands:
  completion  Generate the autocompletion script for the specified shell
  destroy     Remove worktrees from the pool, safely by default
  enter       Open a subshell in an existing worktree by name, even if in use
  get         Acquire a worktree from the pool and open a subshell
  help        Help about any command
  init        Create a default treehouse.toml config file
  prune       Remove stale worktrees and opted-in orphans from the pool
  return      Terminate lingering processes and return a worktree
  status      Show the status of all worktrees in the pool
  update      Update treehouse to the latest version
```

The repository confirms the absence: it ships no `.agents/skills` directory, and no treehouse entry exists in the Claude Code or Codex hook configurations on this box.

### 8. Content first - gap

The bare invocation is a mutation, not a view.
`treehouse` with no arguments is an alias for `get`, so it acquires a worktree and opens an interactive subshell.

```
$ treehouse </dev/null
🌳 Setting up worktree...                                                   # stderr
🌳 Entered worktree at ~/.treehouse/repo-1a4302/2/repo. Type 'exit' to return.
🌳 Worktree returned to pool.
```

Stdout is empty and exit is `0`, so an agent that probes the tool the AXI way learns nothing and silently consumes a pool slot.
Outside a repository the same invocation prints a git error and exits `1`.

```
$ cd /tmp && treehouse
not in a git repository: git rev-parse --show-toplevel: fatal: not a git repository (or any of the parent directories): .git
```

This is the single largest gap, because the AXI-correct probe is the most destructive thing an agent can do by accident.

### 9. Contextual disclosure - partial pass

Two commands already suggest a concrete next command, and both suggestions are complete and actionable.

```
$ treehouse get --lease
🌳 Leased worktree at ~/.treehouse/repo-1a4302/3/repo. Run 'treehouse return ~/.treehouse/repo-1a4302/3/repo' to release it.   # stderr
$ treehouse prune
🌳 Re-run with --yes to delete these worktrees.
$ treehouse destroy /root/.treehouse/repo-1a4302/1/repo
🌳 Skipped 1 worktree:
  1     [leased]  ~/.treehouse/repo-1a4302/1/repo  re-run with --include-leased to include
```

The verdict falls short of a pass because the suggestions are on stderr rather than in the structured stdout output, there is no `help[n]` block, `status` offers no next step at all, and errors suggest nothing.

### 10. Consistent way to get help - partial pass

Per-subcommand `--help` is genuinely strong: `get`, `status`, `prune`, `destroy`, and `enter` each document their own flags with prose that explains the semantics, and `destroy --help` even carries a migration note for the removed `--force` flag.
The gaps are at the top and at the version fast path.

The home view does not identify the tool: there is no `bin:` line with the executable path and no one-sentence `description:`, because there is no home view at all (see principle 8).

The version flags are inconsistent.
`--version` and `-v` both work, but `-V` is rejected.

```
$ treehouse --version
v2.1.0
$ treehouse -v
v2.1.0
$ treehouse -V
unknown shorthand flag: 'V' in -V        # exit 1
```

`-v` is also overloaded: on `prune` it means `--verbose`, not `--version`.
Latency is not a concern for a Go binary; `--version` measured at 13 ms wall clock.

An update nag prints on stderr ahead of ordinary output unless suppressed, which is noise on every invocation.

```
$ treehouse --help
A new version of treehouse is available: v2.1.0 → v2.3.0
Run "treehouse update" to update
```

## Change list

1. Emit TOON on stdout for every command's structured output, converting at the output boundary and keeping the internal model as JSON (principle 1).
2. Replace `--json` with the TOON default and give `status`, `get --lease`, `prune`, `destroy`, and `return` a structured stdout representation each (principles 1, 6).
3. Reduce the default `status` row to name, status, and path, drop empty lease and process fields from the default schema, and add `--fields` to request the rest (principle 2).
4. Truncate variable-length content, notably per-worktree process lists and long pool listings, with a size hint and a `--full` escape hatch (principle 3).
5. Add pre-computed aggregates to `status`: total slots, capacity from `max_trees`, and per-status counts including how many are acquirable right now (principle 4).
6. Make the empty pool a definitive stdout statement in both the default and structured views, replacing the bare `[]` and moving the message off stderr (principle 5).
7. Move all agent-consumed output, including errors and next-step hints, to stdout, and leave stderr for banners, progress, and diagnostics (principles 5, 6, 9).
8. Return exit code 2 for usage errors, keeping 1 for operational failures and 0 for no-ops (principle 6).
9. Translate errors into an actionable structured form with the fixing command inlined, listing a command's valid flags on an unknown flag, and stop leaking raw `git` and `jj` output (principle 6).
10. Suppress the update-available nag on non-interactive invocations, or move it behind an explicit command (principle 6).
11. Add an explicit opt-in setup command that installs a session-start integration for Claude Code, Codex, and OpenCode, showing the current pool's compact state (principle 7).
12. Ship an installable skill generated from the same content as the home view, with a CI check that fails on drift (principle 7).
13. Make the bare invocation a read-only home view of the current pool, and require `treehouse get` explicitly to acquire (principle 8).
14. Add a `help[n]` next-step block to the home view, `status`, and every mutation result, and drop it from self-contained detail output (principle 9).
15. Add `bin:` and `description:` lines to the home view (principle 10).
16. Accept `-V` as a version flag alongside `-v` and `--version`, and resolve the `-v` collision with `prune --verbose` (principle 10).

## Non-goals

The following are consciously waived.

The version fast-path optimization described under principle 10 is not adopted.
That guidance targets ESM static-import cost in Node CLIs; treehouse is a single Go binary whose `--version` already returns in 13 ms, so there is no import graph to defer and no latency to reclaim.

Changing which commands mutate pool state, beyond making the bare invocation read-only, is out of scope.
The safety semantics of `get`, `return`, `prune`, and `destroy` are the subject of the project's own VISION.md and are deliberately untouched by an output-ergonomics retrofit.

The interactive subshell that `get` and `enter` open is retained.
AXI forbids interactive prompts for values an agent must supply, and `get --lease` already provides the complete flags-only path; the subshell is the product's human-facing feature, not a prompt standing between an agent and a result.

The quality of per-subcommand `--help` prose is left as it is.
It already meets principle 10's bar, so the retrofit only adds the home-view identification lines rather than rewriting help text.

## Evidence

Bare invocation, in the throwaway scratch repository, stdin closed so the interactive subshell exits immediately:

```
$ cd /root/.jcode/scratch/dev54/repo   # throwaway scratch repo, treehouse init already run
$ treehouse            # bare invocation, stdin closed
[stdout]
[exit] 0
[stderr]
🌳 Setting up worktree...
🌳 Entered worktree at ~/.treehouse/repo-1a4302/2/repo. Type 'exit' to return.
🌳 Worktree returned to pool.
```

Hot path, the non-interactive lease acquire a parallel agent actually runs, followed by the pool read that agent uses to see the result:

```
$ cd /root/.jcode/scratch/dev54/repo
$ treehouse get --lease --json      # non-interactive acquire, as a parallel agent runs it
[stdout]
{"path":"/root/.treehouse/repo-1a4302/3/repo","lease_id":"c7d8cf8fcd7a96ee6a92ab0bb6746450","lease_holder":"","leased_at":"2026-09-02T21:46:28.384814258Z"}
[exit] 0
[stderr]
🌳 Setting up worktree...
🌳 Leased worktree at ~/.treehouse/repo-1a4302/3/repo. Run 'treehouse return ~/.treehouse/repo-1a4302/3/repo' to release it.

$ treehouse status
[stdout]
1     leased       ~/.treehouse/repo-1a4302/1/repo
2     leased       ~/.treehouse/repo-1a4302/2/repo  (held by agent-x)
3     leased       ~/.treehouse/repo-1a4302/3/repo
[exit] 0
[stderr]
```
