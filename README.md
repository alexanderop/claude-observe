# claude-observe

**Session observability for Claude Code — the half a transcript can't record.**

Claude Code already writes a transcript for every session in every project. That
transcript is an excellent record of what *happened*: every tool call, every
token, every subagent. It is a poor record of what was *prevented* and what
*stopped responding*.

A session that was killed looks exactly like one that finished cleanly — the
file just stops. A hung session emits nothing at all, so silence is
indistinguishable from deep thought. A denied permission and an API-killed turn
leave, at best, an error string you have to parse out of prose.

`claude-observe` records that half, in every project, with no per-project setup.

```
3 session(s) — ~/.claude/plugins/data/claude-observe/sessions

  ORPHANED  a1b2c3d4  2026-08-03 14:02:11  pid 41337
            ~/Projects/api
            stopped without closing — killed, crashed, or machine went down
            2 tool failure(s) · 1 API error(s)

  running   7deb9018  2026-08-03 15:44:02  pid 98441
            ~/Projects/liveclaudecode
            1 denial(s) · 3 subagent(s)

  ended     5209e2d4  2026-08-03 15:41:31  pid 98290
            ~/Projects/site
            ended: prompt_input_exit
```

## Install

```bash
/plugin marketplace add alexanderop/claude-observe
/plugin install claude-observe@claude-observe
```

Enabled once, it applies to every project. Nothing to add per repository.

## Use

```bash
observe-report              # recent sessions, newest first
observe-report --live       # only sessions believed to be running
observe-report <session_id> # full event detail (prefix match works)
observe-report --json       # machine-readable
```

Or just ask, and the bundled skill handles it: *"is anything still running?"*,
*"why did that session die?"*, *"what failed in the last run?"*

## The three states

The state column is the point of the plugin.

| State | Meaning |
|---|---|
| `running` | Started, never closed, and the recorded pid is still a live `claude` process. |
| `ended` | A `SessionEnd` was recorded, with the reason it closed. The normal case. |
| `ORPHANED` | Started, never closed, and the pid is gone. **Killed, crashed, or the machine went down.** |
| `unknown` | The pid could not be resolved, so liveness genuinely cannot be determined. |

`ORPHANED` is the one you can't get any other way. Liveness is re-verified
against `ps` on every report and compared by executable name, not by a substring
match — pids get recycled, and a recycled pid running any process that merely
references a path under `~/.claude` would otherwise pass.

## What it records, and what it deliberately doesn't

Hooked, because a transcript cannot contain it:

| Event | What it gives you |
|---|---|
| `SessionStart` | pid, cwd, and how the session began (`startup` / `resume` / `fork`) |
| `SessionEnd` | *why* it closed — `clear`, `logout`, `prompt_input_exit`, `other` |
| `StopFailure` | a turn killed by an API error |
| `PostToolUseFailure` | typed failures — `tool_error_type` is `timeout`, not prose to regex |
| `PermissionDenied` | what the classifier refused |
| `SubagentStart` / `SubagentStop` | subagent lifecycle with `agent_type` |
| `PreCompact` / `PostCompact` | compaction boundaries and trigger |
| `Notification` | what was surfaced to you |

**Not hooked: `PreToolUse` and `PostToolUse`.** They fire on every tool call in
every project and `PreToolUse` sits on the critical path, while their only real
payoff — wall-clock tool duration — is already derivable from the transcript's
own `tool_use → tool_result` timestamps.

Every event above is either rare or on a failure path, so on a healthy session
this plugin runs a handful of times total. **Instrument what you can't derive;
derive the rest.**

## Where the data lives

`${CLAUDE_PLUGIN_DATA}/sessions/<session_id>.jsonl` — which resolves to
`~/.claude/plugins/data/claude-observe/`, is created on first use, and survives
plugin updates.

One JSON object per line. The raw hook payload is embedded verbatim rather than
parsed and re-serialized, so nothing is lost and the recorder needs no JSON
parser:

```json
{"ts":"2026-08-03T19:41:01Z","event":"PostToolUseFailure","hook_pid":97709,
 "claude_pid":90455,"payload":{ ...the untouched hook payload... }}
```

Every payload carries `transcript_path`, so a log line joins to the transcript
Claude Code already wrote with no inference at all.

Nothing is sent anywhere. No network, no telemetry, no daemon.

## Design notes

**The recorder never fails a hook.** Every path exits 0. A broken observer that
breaks your session is worse than no observer.

**No hard dependencies in the hot path.** `bin/observe-record` is POSIX-ish bash
using only `ps`, `sed`, and `date`. The reporter is dependency-free Node, and it
is never on the critical path.

**Unresolved is not the same as dead.** If the pid walk fails, the recorder
writes `0` and the reporter says `unknown` rather than claiming the session
died. An absence of evidence is reported as an absence, never as a diagnosis.

**Finding the Claude Code process takes care.** The hook runs under a shell that
sources a snapshot from `~/.claude/...`, so matching the process tree on
arguments finds a shell and stops one hop early. Matching on the executable name
(`comm`) is correct. Verified against a real tree:

```
hook shell (/bin/zsh) → claude → login shell → terminal
```

## Development

```bash
claude --plugin-dir ./claude-observe    # load without installing
claude plugin validate ./claude-observe
```

Under `--plugin-dir` the data directory is `claude-observe-inline`, not
`claude-observe` — so a development run won't scribble on your installed logs.

## License

MIT
