---
description: List recent Grok sessions for this repo
argument-hint: '[-n <count>] [--limit <count>]'
disable-model-invocation: true
allowed-tools: Bash
---

List recent Grok sessions for the current repo via `grok sessions list`.

Raw slash-command arguments:
`$ARGUMENTS`

Core constraint:
- This command only LISTS sessions: it never resumes, deletes, or starts sessions.
- It forwards no flags other than `-n`/`--limit`.

The cwd rule:
- `grok sessions list` is cwd-scoped automatically — it shows sessions for the current working directory, including sibling worktrees of the same repo, grouped by worktree label — and it has NO `--dir` or `--all` flag.
- So the listing must cd to the repo root and run in the SAME Bash call: `cd <repo> && grok sessions list ...`. Never split the cd and the listing across two Bash calls — the shell's cwd resets between calls.

Flag passthrough:
- If `-n <count>` or `--limit <count>` appears in the arguments, pass it through verbatim. (Grok defaults to 20 when neither is given; do not inject `-n`.)
- Nothing else is forwarded.

Preflight:
- `command -v grok`. If the binary is missing, stop and point the user at `/grok:setup`.

Output rules:
- Relay the session table as-is (columns: SESSION ID | CREATED | UPDATED | STATUS | SUMMARY).
- Then append two notes:
  1. Delegations started with `--bg` run as Claude Code background tasks tracked by the harness itself (task notifications) — that replaces codex's job queue; there is no grok job queue to poll.
  2. Use `/grok:result [session-id]` to see a finished session's output, or `grok -r <session-id>` to resume one interactively.
