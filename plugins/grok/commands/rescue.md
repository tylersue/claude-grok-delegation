---
description: Delegate investigation, an explicit fix request, or follow-up rescue work to Grok
argument-hint: '[--read-only] [--resume|--fresh] [--model <id>] [--effort <level>] [--bg] [what Grok should investigate, solve, or continue]'
allowed-tools: Bash, Agent, AskUserQuestion
---

Delegate investigation, an explicit fix request, or follow-up rescue work to Grok through the grok-worker courier.

Invoke the `grok:grok-worker` subagent via the `Agent` tool (`subagent_type: "grok:grok-worker"`), forwarding the user's request as the task. `grok:grok-worker` is a subagent, not a skill — do not call `Skill(grok:rescue)` (that re-enters this command) and do not route through `Skill(grok:delegate)` (that adds a hop for nothing). The command runs inline so the `Agent` tool stays in scope.

Raw user request:
$ARGUMENTS

Routing rules:
- Forward the raw request to grok-worker with the routing flags spelled out and stripped from the task text: `--read-only`, `--resume`, `--fresh`, `--model <id>`, `--effort <level>`, `--bg` (the worker also strips them defensively).
- Write-capable is the default; `--read-only` switches the worker to its hardened read-only review mode.
- Caution: default mode runs grok with `--yolo` always-approve — Grok can edit files and run shell commands without prompting; prefer `--read-only` when no edits are needed.
- `--bg` means run the Agent spawn as a Claude background task and report on completion. Do not forward `--bg` as task text.
- Leave `--model` and `--effort` unset unless explicitly given; pass explicit values through verbatim — never invent model IDs.

Resume handling:
- If `--resume` or `--fresh` is present, do not ask — the user already chose.
- If neither is present AND the request reads as a follow-up to prior Grok work in this repo ("continue", "keep going", "resume", "apply the fix you found"), use `AskUserQuestion` exactly once with these two options, in this order:
  - `Continue the most recent Grok session (Recommended)`
  - `Start fresh`
- Then add the corresponding flag (`--resume` or `--fresh`) before routing.
- Otherwise route normally with no question. The worker may still infer continuation from the language — that is acceptable.

Preflight:
- Optionally run one inline Bash call — `command -v grok` — to fail fast. If the binary is missing, stop and tell the user to run `/grok:setup`; do not spawn the worker just to learn the binary is absent.
- If the worker later reports an authentication or rate-limit problem, relay its guidance verbatim and mention `/grok:setup`.

Output rules:
- Return Grok's output essentially verbatim, attributed ("Grok's result: ...").
- Do not paraphrase, summarize, or do follow-up work of your own.
- If the user supplied no request, ask what Grok should investigate or fix.
- Overlap note: `/grok:delegate` remains the general-purpose delegation entry; `/grok:rescue` is the stuck/diagnosis-framed entry.
