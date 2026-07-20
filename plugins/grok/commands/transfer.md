---
description: Bridge this Claude Code session into Grok's /resume picker (guided handoff)
argument-hint: ''
disable-model-invocation: true
allowed-tools: Bash, Read
---

Preflight and guide a handoff of this Claude Code session into Grok — grok imports Claude conversations only through its interactive `/resume` picker; there is no headless import.

Core constraint:
- Interactive-only: no `--import-claude-session` CLI flag exists (confirmed against grok source) — the handoff ends with instructions, not an automated import. Repeat this disclosure in the user-facing output.
- Experimental: grok's own docs label the harness-compat session cells "staged; no scanner consumer yet" while the source implements the scanner — behavior may change between grok versions. Repeat this disclosure in the user-facing output.
- Do not confuse with grok's TUI `/import-claude` command — that imports SETTINGS (permissions/env/MCP/hooks), not conversations.
- This command is report-only: it never edits `~/.grok/config.toml`, never installs skills, and never writes anywhere.

Preflight checks — run via Bash, report each as a short checklist line:

1. **Binary**: `command -v grok`. If missing, report it and point at `/grok:setup` — but still run the remaining file checks.
2. **Compat flag**: `[compat.claude] sessions` DEFAULTS TO TRUE — it is only off when explicitly disabled. If `~/.grok/config.toml` exists, inspect only the `[compat.claude]` block for a `sessions = false` line (grep the block/key — never dump the whole file). Report "enabled (default)" / "enabled (explicit)" / "DISABLED — re-enable `sessions` under `[compat.claude]` in `~/.grok/config.toml`". Note that an environment variable can also override (resolution order: env var > config.toml > default-on).
3. **`resume-claude` skill**: check grok's standard skill discovery locations for a `resume-claude` skill directory or SKILL.md — `./.grok/skills/resume-claude/`, the `.agents/skills/` tiers between the cwd and the repo root, and `~/.grok/skills/resume-claude/`. Without this skill grok performs ZERO foreign-session filesystem I/O. If absent from the standard dirs, report it as likely missing — `[skills].paths` in config.toml can add directories, and grok's own `/skills` command is the authoritative list.
4. **Claude session JSONL**: grok's scanner reads `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`, cwd-matched EXACTLY (sidechains excluded). Claude Code encodes the project path in the directory name with `/` replaced by `-` (e.g. `-Users-alice-code-myrepo`); locate that directory for the current repo path and check that at least one `*.jsonl` exists — the current session's own transcript qualifies. Report only the count and the most recent mtime; NEVER print JSONL contents.

Handoff instructions (the user-facing payload, printed after the checklist):
- Run `grok` interactively in this EXACT directory — the scanner matches the cwd exactly; a subdirectory or a different worktree will not match.
- Open the `/resume` picker and select the Claude Code session from the list — it appears alongside grok's own sessions; picking it continues the conversation in Grok.
- Repeat both disclosures here: the handoff is interactive-only (no headless import exists) and experimental ("staged" per grok's own docs).
- If any check failed, list what to fix first instead of the instructions.
