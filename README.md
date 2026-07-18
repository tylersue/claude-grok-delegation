# Grok Delegation for Claude Code

Delegate tasks from [Claude Code](https://claude.com/claude-code) to xAI's Grok via the [`grok` CLI (Grok Build)](https://github.com/xai-org/grok-build) — independent second opinions, plan/diff reviews, diagnosis passes, or parallel implementation work from a non-Claude model family.

## Why

Claude Code subagents can only run Claude models — the agent `model` field accepts no `grok-*` (or other vendor) IDs, and gateway rerouting to non-Claude models is unsupported. The sanctioned pattern is **CLI delegation**: a thin Claude courier subagent drives another vendor's agent CLI headlessly, and that vendor's model does the actual reasoning in its own agentic loop.

Why bother? Different model families catch different blind spots. A review from a second (or third) model family routinely surfaces issues the first one rationalized away — and delegation keeps the heavy externally-generated output out of your expensive main-loop context.

This plugin is the Grok half of that pattern. It pairs naturally with OpenAI's codex plugin for Claude Code, giving you a three-family panel: Claude (main loop) + GPT (Codex) + Grok.

## Prerequisites

- The `grok` CLI installed and authenticated (one-time, interactive):

  ```bash
  curl -fsSL https://x.ai/cli/install.sh | bash
  grok   # run once interactively to log in, then exit
  ```

- A Grok subscription/account that the CLI accepts.

## Install

**As a plugin** (recommended):

```
/plugin marketplace add tylersue/grok-delegation
/plugin install grok@grok-delegation
```

This registers the `grok:grok-worker` agent and the `/grok:delegate` skill.

**Manual copy** (bare `/grok`, no plugin system):

```bash
git clone https://github.com/tylersue/grok-delegation
cp grok-delegation/plugins/grok/agents/grok-worker.md ~/.claude/agents/
mkdir -p ~/.claude/skills/grok
cp grok-delegation/plugins/grok/skills/delegate/SKILL.md ~/.claude/skills/grok/SKILL.md
```

Restart your Claude Code session so the agent registers.

## Usage

```
/grok:delegate review this diff for concurrency bugs --read-only
/grok:delegate implement the retry logic described in TODO.md
/grok:delegate --model grok-4.5 --effort high audit the auth flow --read-only
/grok:delegate --resume apply the fix you proposed
/grok:delegate --bg refactor the parser to a visitor pattern
```

Claude will also delegate proactively (no slash command needed) when a cross-AI second opinion or a fresh diagnosis pass is the right move — see [docs/claude-md-rules.md](docs/claude-md-rules.md) for the CLAUDE.md rules that encourage this.

### Routing flags

| Flag | Effect |
|---|---|
| `--read-only` | Review/diagnosis only — Grok gets `read_file,grep,list_dir` and no shell |
| `--model <id>` | Pass a specific Grok model through verbatim |
| `--effort <level>` | Pass reasoning effort through |
| `--resume` / `--fresh` | Continue the most recent Grok session in this repo vs. start clean |
| `--bg` | Run in the background; report on completion |

Everything else is forwarded to Grok as the task.

## How it works

```
Claude Code (main loop)
  └─ grok-worker agent  (small, cheap Claude courier — forwards, never reasons)
       └─ grok CLI, headless: --prompt-file <task> --output-format plain --yolo
            └─ Grok's own agentic loop (reads, edits, shell) in your repo
```

Design principles:

- **The courier is thin.** The subagent forwards the task and relays the result verbatim, attributed. Grok does the reasoning.
- **Write-capable by default, read-only on request.** Reviews and diagnosis passes run with a read-only tool allowlist.
- **Never invent model IDs.** The courier only passes `-m` when you explicitly name a model.
- **Disagreements are surfaced, not resolved silently.** If Grok's take conflicts with Claude's, both positions are presented with attribution.

## Optional extras

- [docs/claude-md-rules.md](docs/claude-md-rules.md) — a copy-paste CLAUDE.md section that makes cross-AI delegation a standing habit (review before shipping, delegate diagnosis after repeated failed fixes, dual review for high-stakes designs).
- [docs/gsd-review-patch.md](docs/gsd-review-patch.md) — for users of [GSD (get-shit-done)](https://github.com/open-gsd/gsd-core): a patch guide that adds Grok as a first-class reviewer in `/gsd:review` alongside the built-in Codex/Gemini/etc. roster.

## License

[MIT](LICENSE)
