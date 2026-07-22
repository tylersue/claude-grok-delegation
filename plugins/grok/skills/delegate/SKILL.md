---
name: delegate
description: Delegate a task to SpaceXAI's Grok via the grok CLI (Grok Build) — independent second opinions, plan/diff reviews, diagnosis passes, or parallel implementation work from a non-Claude model family. Use when the user invokes the skill with a task, asks for "Grok's take", or wants a cross-AI second opinion.
---

# Grok CLI Delegation

Route the given task to SpaceXAI's Grok through the `grok-worker` agent (a thin courier that drives `grok` headless). Grok runs its own full agentic loop — file reads, edits, terminal commands — inside the target directory.

## Steps

1. **Preflight** (inline, one Bash call): `command -v grok`. If missing, tell the user to run `curl -fsSL https://x.ai/cli/install.sh | bash` and then `grok` once to authenticate; stop.
2. **Parse routing flags** from the arguments and strip them from the task text:
   - `--read-only` — review/diagnosis only, no edits (also infer this when the task is clearly a review/opinion request)
   - `--model <id>` — explicit Grok model passthrough
   - `--effort <level>` — reasoning effort passthrough
   - `--resume` / `--fresh` — continue the most recent Grok session in this repo vs. start clean
   - `--bg` — run the agent in the background and report when it completes
3. **Delegate**: spawn the `grok-worker` agent (Agent tool; registered as `grok:grok-worker` when installed via the plugin, or plain `grok-worker` if the agent file was copied to `~/.claude/agents/`) with the task text, the working directory, and the routing flags spelled out. Foreground by default; background only for `--bg` or clearly long-running work.
4. **Relay** Grok's output faithfully and attribute it ("Grok's review/result: ..."). If it disagrees with your own view, say so explicitly and give your reasoning — do not silently adopt or discard either position.

## Notes

- The courier agent never reasons about the task itself; Grok does the work. Don't pre-solve the task before delegating.
- Trivial pure-text questions with no repo interaction may skip the agent, but still go through a prompt file — never inline the question into shell quoting (embedded quotes break argument parsing, and `$()`/backticks execute before grok ever sees the text): `PROMPT_FILE=$(mktemp "${TMPDIR:-/tmp}/grok-task-XXXXXX")` (path only, no shell involvement), then use the Write tool (file_path `$PROMPT_FILE`, content the question verbatim) to populate it, then run `grok --prompt-file "$PROMPT_FILE" --output-format plain --max-turns 5` referencing only the quoted path, then `rm -f "$PROMPT_FILE"`.
- Pairs naturally with other cross-AI delegation plugins (e.g. OpenAI's codex plugin): for high-stakes changes, get more than one external model's read, then reconcile disagreements explicitly.
