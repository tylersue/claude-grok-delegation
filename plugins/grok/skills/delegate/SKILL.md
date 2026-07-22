---
name: delegate
description: Delegate a task to SpaceXAI's Grok via the grok CLI (Grok Build) — independent second opinions, plan/diff reviews, diagnosis passes, or parallel implementation work from a non-Claude model family. Use when the user invokes the skill with a task, asks for "Grok's take", or wants a cross-AI second opinion.
---

# Grok CLI Delegation

Route the given task to SpaceXAI's Grok through the `grok-worker` agent (a thin courier that drives `grok` headless). Grok runs its own full agentic loop — file reads, edits, terminal commands — inside the target directory.

## Steps

1. **Preflight** (inline, one Bash call): `command -v grok`. If missing, tell the user to run `curl -fsSL https://x.ai/cli/install.sh | bash` and then `grok` once to authenticate; stop.
2. **Parse routing flags** from the arguments per the explicit grammar below (`--read-only` — review/diagnosis only, no edits, also inferred when the task is clearly a review/opinion request; `--model <id>` — explicit Grok model passthrough; `--effort <level>` — reasoning effort passthrough; `--resume`/`--fresh` — continue the most recent Grok session in this repo vs. start clean; `--bg` — run the agent in the background and report when it completes). This grammar is duplicated byte-for-byte in `plugins/grok/agents/grok-worker.md` — the two copies are kept in sync by `check_flag_grammar_sync()` in `tests/validate_plugin.py`; do not diverge.

<!-- FLAG-GRAMMAR-START -->
1. Routing flags are recognized ONLY at the start of the arguments (leading-only). The first non-flag token begins the task text, which is taken verbatim from there to the end.
2. `--` explicitly ends flag parsing; everything after `--` is verbatim task text (for tasks that legitimately begin with a dash).
3. Recognized routing flags: --read-only, --model <value>, --effort <value>, --resume, --fresh, --bg.
4. Missing value: --model or --effort with no following token → error-and-stop. Report the specific problem in one line; do not run grok.
5. Duplicate of any flag → error-and-stop (uniform, no special cases).
6. --resume together with --fresh → error-and-stop.
7. --effort value must be one of none, minimal, low, medium, high, xhigh, max (case-insensitive); anything else → error-and-stop.
8. --model value is shape-checked only: non-empty, a single token, no leading dash; passed through verbatim (never invent model IDs).
9. Every forwarded flag value is passed to grok as its OWN separately quoted shell argument (e.g. --model "$MODEL", --effort "$EFFORT") — never concatenated or interpolated into a command string.
10. Flag names appearing later inside the task text are never parsed or stripped.
<!-- FLAG-GRAMMAR-END -->
3. **Delegate**: spawn the `grok-worker` agent (Agent tool; registered as `grok:grok-worker` when installed via the plugin, or plain `grok-worker` if the agent file was copied to `~/.claude/agents/`) with the task text, the working directory, and the routing flags spelled out. Foreground by default; background only for `--bg` or clearly long-running work.
4. **Relay** Grok's output faithfully and attribute it ("Grok's review/result: ..."). If it disagrees with your own view, say so explicitly and give your reasoning — do not silently adopt or discard either position.

## Notes

- The courier agent never reasons about the task itself; Grok does the work. Don't pre-solve the task before delegating.
- Trivial pure-text questions with no repo interaction may skip the agent, but still go through a prompt file — never inline the question into shell quoting (embedded quotes break argument parsing, and `$()`/backticks execute before grok ever sees the text). Deliver it as three deterministic, observable steps, same as the courier agent — because shell variable state does NOT persist across separate Bash tool invocations, the resolved path must be observed and threaded forward as a literal:
  (1) Bash call — `PROMPT_FILE=$(mktemp "${TMPDIR:-/tmp}/grok-task-XXXXXX")` (path only, no shell involvement), then immediately `echo "$PROMPT_FILE"` so the resolved absolute path is observable in this call's tool result.
  (2) Write tool call — `file_path` MUST be the LITERAL absolute path printed by step (1) — never the string `$PROMPT_FILE`, which is neither absolute nor a value the Write tool can resolve. Content is the question verbatim.
  (3) run Bash call — reference that SAME literal path: hardcode it everywhere `$PROMPT_FILE` appears below (`--prompt-file` and the trap), or re-bind `PROMPT_FILE=<the literal path>` (a plain literal assignment, not a `$(…)` command substitution) as this call's first statement, since shell state does not persist across separate Bash tool invocations. In this same call set `trap 'rm -f "$PROMPT_FILE"' EXIT TERM INT` before running `grok --prompt-file "$PROMPT_FILE" --output-format plain --max-turns 5`, referencing only the same literal path — the trap removes the prompt file on normal exit and SIGTERM, so no separate `rm -f` is needed afterward.
- Pairs naturally with other cross-AI delegation plugins (e.g. OpenAI's codex plugin): for high-stakes changes, get more than one external model's read, then reconcile disagreements explicitly.
