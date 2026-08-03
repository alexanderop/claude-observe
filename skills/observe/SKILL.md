---
description: Report on Claude Code session health — which sessions are running, which died without closing, what tools failed and why, what permissions were denied, and where context was compacted. Use when the user asks about session status, a hung or stuck session, why a session ended, what went wrong in a recent run, tool failures, permission denials, or subagent activity. Triggers include "is anything still running", "did that session die", "what failed", "why did it stop", "session health", "observe".
---

# Session observability

`claude-observe` records what a transcript cannot. Transcripts are a record of
what *happened*; this is a record of what was *prevented* and what *stopped
responding*.

## Reading the data

Run the reporter. The plugin's `bin/` is on the Bash tool's `PATH` while the
plugin is enabled, so this is a bare command — do not path-qualify it:

```bash
observe-report              # recent sessions, newest first
observe-report --live       # only sessions believed to be running
observe-report <session_id> # full event detail for one session (prefix match works)
observe-report --json       # machine-readable, for further analysis
```

Never parse the raw logs by hand when the reporter answers the question. Reach
for the raw JSONL under `$CLAUDE_PLUGIN_DATA/sessions/` only when the user
wants a field the reporter does not surface.

## Interpreting the three states

The state column is the whole point of this plugin. Report it precisely.

- **running** — a `SessionStart` with no `SessionEnd`, and the recorded pid is
  still a live Claude Code process. The pid is re-verified against `ps` on every
  report, because pids get recycled and a bare "is this pid alive" check will
  eventually lie.
- **ended** — a `SessionEnd` was recorded. The reason (`clear`, `logout`,
  `prompt_input_exit`, `resume`, `other`) says how it closed. This is the normal
  case and needs no comment.
- **ORPHANED** — a `SessionStart` with no `SessionEnd`, and the pid is gone.
  The session stopped without ever closing: killed, crashed, or the machine went
  down. **This is the state worth surfacing unprompted.** A transcript for an
  orphaned session looks identical to one that ended cleanly — it just stops —
  which is exactly the blind spot this plugin exists to close.

A session that is `running` but has produced no events for a long time is the
other case worth flagging: a hung agent emits nothing at all, so silence is
indistinguishable from deep thought unless you check the clock.

## What is and is not recorded

Recorded, because a transcript cannot contain it: liveness and pid, why a
session ended, typed tool failures (`tool_error_type`, e.g. `timeout`), denied
permissions, turns killed by API errors, subagent lifecycle, compaction
boundaries, and notifications.

Not recorded, deliberately: every successful tool call. Those are already in the
transcript with timestamps, and hooking them would put this plugin on the
critical path of every tool call in every project. If the user asks about tool
timing or what a session did, read the transcript — `observe-report` prints its
path for each session.

When something genuinely is not in the log, say so. Do not infer a cause from an
absence: a missing `SessionEnd` means "no close was recorded", which is a fact,
not a diagnosis.
