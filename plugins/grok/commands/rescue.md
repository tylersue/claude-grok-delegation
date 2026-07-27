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
- Forward the raw request to grok-worker; name the routing flags in the spawn prompt as hints only — `--read-only`, `--resume`, `--fresh`, `--model <id>`, `--effort <level>`, `--bg` — the worker's routing-flag grammar is solely responsible for parsing and stripping them out of the task text.
- Write-capable is the default; `--read-only` switches the worker to its hardened read-only review mode.
- Caution: default mode runs grok with `--yolo` always-approve — Grok can edit files and run shell commands without prompting; prefer `--read-only` when no edits are needed.
- `--bg` means run the Agent spawn as a Claude background task and report on completion. Do not forward `--bg` as task text.
- Leave `--model` and `--effort` unset unless explicitly given; pass explicit values through verbatim — never invent model IDs.
- `--resume` and `--fresh` are mutually exclusive — do not use both together (this command forwards raw text and does not parse flags itself, but the worker's routing-flag grammar treats `--resume` + `--fresh` as an error-and-stop condition, so never document or imply that combining them is valid). Task text that legitimately begins with a dash is passed through after a `--` terminator, consistent with the worker's grammar, so it is never misread as a flag.

Resume handling:
- If `--resume` or `--fresh` is present, do not ask — the user already chose.
- If neither is present AND the request reads as a follow-up to prior Grok work in this repo ("continue", "keep going", "resume", "apply the fix you found"), use `AskUserQuestion` exactly once with these two options, in this order:
  - `Continue the most recent Grok session (Recommended)`
  - `Start fresh`
- Then add the corresponding flag (`--resume` or `--fresh`) before routing.
- Otherwise route normally with no question. The worker may still infer continuation from the language — that is acceptable.

Preflight:
- Optionally run one inline Bash call — `command -v grok` — to fail fast. If the binary is missing, stop and tell the user to run `/grok:setup`; do not spawn the worker just to learn the binary is absent.
- If the worker later reports the grok CLI is missing (the courier's preflight abort — no `Grok run:` line is emitted for this one exempted case), relay its install guidance and mention `/grok:setup`. For a courier report of an attempted run, see the courier-failure handling under Output rules below.

Output rules:
- Return Grok's output essentially verbatim, attributed ("Grok's result: ...").
- Do not paraphrase, summarize, or do follow-up work of your own.
- If the user supplied no request, ask what Grok should investigate or fix.
- Overlap note: `/grok:delegate` remains the general-purpose delegation entry; `/grok:rescue` is the stuck/diagnosis-framed entry.

<!-- COURIER-FAILURE-START -->
**Courier-failure handling (D-12..D-15):** the courier's `Grok run:` status line is always preserved verbatim as the literal first line of what is presented here — never paraphrased. Branch on that literal line only; never re-derive the failure class from grok's own raw output prose.
- `Grok run: FAILED (exit N — auth)` — an authentication problem. Relay the courier's guidance verbatim and point the user at `/grok:setup`.
- `Grok run: FAILED (exit N — rate limit)` — state explicitly that this is NOT an authentication failure; do not point at `/grok:setup`. Suggest waiting and retrying later, or falling back to another reviewer or model.
- `Grok run: TIMEOUT (continuable with a follow-up -c run)` — the verbatim first line already carries the `-c` continuation hint; surface it as-is, never paraphrase it away.
- `Grok run: FAILED (exit N — generic)` — a transient, generic failure. Relay grok's output verbatim, then retry EXACTLY ONE time automatically — generic class ONLY; auth, rate-limit, and TIMEOUT never retry. Disclose the retry in the output (e.g. "generic failure — retried once automatically").
Never summarize or paraphrase partial output as a completed result: present a FAILED or TIMEOUT run as a failed run, not a finished one.
<!-- COURIER-FAILURE-END -->
- Rescue is write-capable by default: because a mid-write crash could leave a partial edit on disk, a generic-class retry after such a crash could double-apply file edits — the retry disclosure above names this risk for rescue runs.
