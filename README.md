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

# Rewrite the copied skill's frontmatter name to match its directory (grok),
# not the packaged name (delegate):
sed -i '' 's/^name: delegate$/name: grok/' ~/.claude/skills/grok/SKILL.md   # BSD/macOS
sed -i 's/^name: delegate$/name: grok/' ~/.claude/skills/grok/SKILL.md      # GNU/Linux
perl -pi -e 's/^name: delegate$/name: grok/' ~/.claude/skills/grok/SKILL.md # portable alternative
```

Run exactly one of the three rename lines for your platform. Restart your Claude Code session so the agent registers. Note that the `/grok:*` slash commands ship as plugin surface only — a manual copy gets the agent and skill but not the commands; the frontmatter rename is what makes the manually-copied skill resolve as bare `/grok` instead of `/delegate`, since Claude Code's skill router keys off the frontmatter `name:` field, not the directory name.

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

## Data egress & privacy

Delegating a task hands it to the `grok` CLI running under your own Grok account. Two distinct channels carry data off your machine, and they're not the same thing:

**1. The task itself, via xAI's inference API.** Whatever context the delegated task requires Grok to read — files, diffs, your instructions — goes to xAI's inference API to run the CLI, exactly as if you'd typed it into `grok` yourself. This is the intended, necessary channel.

**2. The grok CLI's own per-turn trace-upload pipeline, independent of what the task needs.** Separately from inference, the `grok` CLI can ship session artifacts to xAI-controlled storage on a per-turn basis — including metadata (repo root path, git remote URL, cwd, your user id/email), prompt images, partial model output, logs, and local memory files. This is gated by a "coding data sharing" setting that has historically defaulted to **opt-in**, and whether uploads actually happen can depend on a **server-controlled flag** — so a local "off" setting is a statement about current server-side trust, not a guaranteed local no-op.

**Context:** in July 2026, a bug in the grok-build harness caused unintended repository content to reach this trace-upload pipeline (the "grok-build repo-upload incident"). xAI's response was to disable default trace retention, flip the relevant server-side upload flags, and open-source the harness for inspection. This plugin doesn't upload anything on its own — it drives the `grok` CLI you already run under your own account, and the exposure described here is a property of that CLI, not something this plugin adds.

**How to minimize what leaves your machine:**

- `grok /privacy opt-out` — opt out of coding data sharing from inside the CLI
- `GROK_TELEMETRY_ENABLED=0` — disable telemetry
- `GROK_TELEMETRY_TRACE_UPLOAD=0` — disable the per-turn trace-upload pipeline specifically
- `GROK_FEEDBACK_ENABLED=0` — disable feedback collection
- `DISABLE_ERROR_REPORTING=1` — disable error reporting
- Team-plan accounts can request **Zero Data Retention (ZDR)**
- Default API retention for the inference channel is **30 days**

A future `/grok:setup` update will surface these settings directly in the preflight check; until then, set the env vars in your shell profile or CI environment.

**Read/write confinement, separate from the two channels above.** Grok CLI delegation also runs under grok's own kernel-enforced `--sandbox` profiles — this controls what grok's `read_file`/`grep`/`list_dir` tools and any shell child processes can actually touch on disk, independent of what data leaves your machine via the two channels above.

**What's confined:** read-only reviews (`/grok:review`, `--read-only` runs) use `--sandbox strict` — kernel-enforced (Landlock on Linux, Seatbelt on macOS) confinement of reads to the current working directory plus essential system paths. A read-only run cannot be talked into reading `~/.ssh`, a parent repo, or a stray outside-workspace `.env` file even when the diff or your instructions say to — the confinement doesn't depend on the model obeying anything. Write-capable (`--yolo`) runs use `--sandbox workspace`: writes are confined to the CWD, `~/.grok/`, and temp dirs (reads stay unrestricted), so a `--yolo` run can no longer write to `~/.ssh`, other repos, or system paths.

**What's NOT confined — read this before assuming more than it delivers:**

- Files **inside** the workspace stay readable under `strict` — a repo-local `.env` is still readable, because the CWD is readable by design. The sandbox controls *where* grok can read/write, not *what* inside the workspace it can read.
- Child-process network blocking is **Linux-only** (seccomp) — a no-op on macOS, where Seatbelt does not enforce it. The tool-level `web_search` disallow described above remains the cross-platform network control regardless of platform.

**Degrade-and-disclose, never fail-closed:** if the sandbox is unavailable (an older CLI without `--sandbox`, an unsupported platform, or a renamed/unrecognized profile after a CLI upgrade), the run proceeds unconfined rather than aborting — and the report always carries a `Sandbox:` line disclosing the actual state (`Sandbox: strict` / `Sandbox: workspace` when active, `Sandbox: UNAVAILABLE — reads/writes unconfined (<reason>)` when degraded), so the confinement state is never silent.

**Want stronger in-workspace protection?** Define a custom grok `sandbox.toml` with `deny` globs (e.g. `**/.env`, `**/*.pem`) to kernel-deny reads of specific in-workspace secrets — kernel-enforced and airtight on macOS. This plugin never writes that config file for you; it's a manual opt-in you set up yourself, with zero config side effects from the plugin.

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

Yes — delegation runs the `grok` CLI under your Grok account, so whatever a task requires Grok to read leaves your machine subject to SpaceXAI's terms. See [Data egress & privacy](#data-egress--privacy) for what actually leaves, the grok CLI's own trace-upload pipeline, and how to minimize it. Don't delegate from repos whose policies forbid that.

### What does it cost?

The plugin is free (MIT). Grok usage is billed by SpaceXAI under your account — and note the free tier has usage limits that agentic runs can exhaust quickly.

### Does it work with GSD (get-shit-done)?

Yes — [docs/gsd-review-patch.md](docs/gsd-review-patch.md) adds Grok as a first-class `/gsd:review` reviewer alongside Codex, Gemini, and the rest of the stock roster.

## Contributing

Contributions are maintainer-reviewed: obvious bug fixes can go straight to a PR, while enhancements and features should start as an issue and wait for maintainer go-ahead before any code is written. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process and quality bar.

## License

[MIT](LICENSE)
