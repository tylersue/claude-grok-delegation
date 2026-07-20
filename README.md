# Grok Delegation for Claude Code

Delegate tasks from [Claude Code](https://claude.com/claude-code) to SpaceXAI's Grok via the [`grok` CLI (Grok Build)](https://github.com/xai-org/grok-build) — independent second opinions, plan/diff reviews, diagnosis passes, or parallel implementation work from a non-Claude model family.

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
/plugin marketplace add tylersue/claude-grok-delegation
/plugin install grok@grok-delegation
```

(The marketplace registers under the name `grok-delegation`, so the install command uses that; the older `tylersue/grok-delegation` repo slug still works via GitHub's redirect.)

This registers the `grok:grok-worker` agent, the `/grok:delegate` skill, and the `/grok:review`, `/grok:adversarial-review`, `/grok:rescue`, `/grok:setup`, `/grok:status`, `/grok:result`, and `/grok:transfer` commands.

**Manual copy** (bare `/grok`, no plugin system):

```bash
git clone https://github.com/tylersue/claude-grok-delegation
cp claude-grok-delegation/plugins/grok/agents/grok-worker.md ~/.claude/agents/
mkdir -p ~/.claude/skills/grok
cp claude-grok-delegation/plugins/grok/skills/delegate/SKILL.md ~/.claude/skills/grok/SKILL.md
```

Restart your Claude Code session so the agent registers. Note that the `/grok:*` slash commands ship as plugin surface only — a manual copy gets the agent and skill but not the commands.

## Usage

```
/grok:delegate review this diff for concurrency bugs --read-only
/grok:delegate implement the retry logic described in TODO.md
/grok:delegate --model grok-4.5 --effort high audit the auth flow --read-only
/grok:delegate --resume apply the fix you proposed
/grok:delegate --bg refactor the parser to a visitor pattern
```

Claude will also delegate proactively (no slash command needed) when a cross-AI second opinion or a fresh diagnosis pass is the right move — see [docs/claude-md-rules.md](docs/claude-md-rules.md) for the CLAUDE.md rules that encourage this.

### Commands

0.2.1 completes codex command parity — every codex command now has a grok equivalent or a documented N/A; `/grok:delegate` remains the general-purpose entry.

| Command | What it does | Notes |
|---|---|---|
| `/grok:review` | Read-only Grok review of local git state | Supports `--base <ref>`, `--scope auto\|working-tree\|branch`, optional focus text; always read-only |
| `/grok:adversarial-review` | Same machinery, challenge-the-design framing | Questions architecture and assumptions, not just line-level defects; always read-only |
| `/grok:rescue` | Delegate investigation, a fix, or follow-up work to Grok | Write-capable by default (see the `--yolo` warning); honors `--read-only`, `--resume`\|`--fresh`, `--model`, `--effort`, `--bg` |
| `/grok:setup` | Preflight: binary, version, auth state, defaults, rate-limit caveat | Never prints credentials; no review-gate equivalent (no stop hook) |
| `/grok:status` | Recent Grok sessions for this repo (`grok sessions list`) | cwd-scoped, includes sibling worktrees; passes `-n`/`--limit` through; backgrounded delegations are tracked by the Claude Code harness itself, not a grok job queue |
| `/grok:result` | Print a finished session's summary and final output from the on-disk transcript | Defaults to the most recent session; read-only, never resumes, never touches credentials; if the on-disk layout differs, it advises `grok -r <id>` instead |
| `/grok:transfer` | Guided handoff of this Claude Code session into Grok's `/resume` picker | Preflights the compat flag, `resume-claude` skill, and a cwd-matching session; interactive-only and experimental ("staged" per grok's docs) |
| `cancel` | Not applicable — background delegations are Claude Code tasks; stop them from Claude Code | There is no grok job queue, so there is nothing for a command to cancel |

Sessions themselves can be deleted with `grok sessions delete <id>` — destructive: it removes the session locally AND remotely.

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

> **Warning:** write-capable delegation runs grok with `--yolo` — grok's always-approve mode. Grok can create, modify, and delete files and run arbitrary shell commands (including `rm` or `git push`) in the target directory without prompting. Only delegate write-capable tasks in trusted, committed (or backed-up) working trees; use `--read-only` for reviews and diagnosis.

Design principles:

- **The courier is thin.** The subagent forwards the task and relays the result verbatim, attributed. Grok does the reasoning.
- **Write-capable by default, read-only on request.** Reviews and diagnosis passes run with a read-only tool allowlist. Write-capable runs use `--yolo` (always-approve): treat them like handing Grok a shell in your repo.
- **Never invent model IDs.** The courier only passes `-m` when you explicitly name a model.
- **Disagreements are surfaced, not resolved silently.** If Grok's take conflicts with Claude's, both positions are presented with attribution.

## Optional extras

- [docs/claude-md-rules.md](docs/claude-md-rules.md) — a copy-paste CLAUDE.md section that makes cross-AI delegation a standing habit (review before shipping, delegate diagnosis after repeated failed fixes, dual review for high-stakes designs).
- [docs/gsd-review-patch.md](docs/gsd-review-patch.md) — for users of [GSD (get-shit-done)](https://github.com/open-gsd/gsd-core): a patch guide that adds Grok as a first-class reviewer in `/gsd:review` alongside the built-in Codex/Gemini/etc. roster.

## FAQ

### Can Claude Code use Grok (or other non-Claude models) as subagents?

Not directly. The subagent `model` field accepts only Claude models, and Anthropic doesn't support gateway rerouting to other vendors. The supported pattern is what this plugin implements: a thin Claude courier subagent drives the `grok` CLI headlessly, and Grok does the actual reasoning in its own agentic loop.

### How do I use Grok from Claude Code?

Install the plugin (see [Install](#install)), make sure the `grok` CLI is authenticated, then `/grok:delegate <task>` — or just ask Claude for "Grok's take" and it will delegate proactively if you've added the [CLAUDE.md rules](docs/claude-md-rules.md).

### Does Grok only review, or can it build too?

Both. Write-capable delegation is the default — Grok reads, edits, and runs commands in your repo (see the `--yolo` warning above). `--read-only` restricts it to reviewing and diagnosis with no write access, including MCP tools.

### Does this send my code to SpaceXAI?

Yes — delegation runs the `grok` CLI under your Grok account, so whatever the task requires Grok to read leaves your machine subject to SpaceXAI's terms, exactly as if you ran `grok` yourself. Don't delegate from repos whose policies forbid that.

### What does it cost?

The plugin is free (MIT). Grok usage is billed by SpaceXAI under your account — and note the free tier has usage limits that agentic runs can exhaust quickly.

### Does it work with GSD (get-shit-done)?

Yes — [docs/gsd-review-patch.md](docs/gsd-review-patch.md) adds Grok as a first-class `/gsd:review` reviewer alongside Codex, Gemini, and the rest of the stock roster.

## Contributing

Contributions are maintainer-reviewed: obvious bug fixes can go straight to a PR, while enhancements and features should start as an issue and wait for maintainer go-ahead before any code is written. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process and quality bar.

## License

[MIT](LICENSE)
