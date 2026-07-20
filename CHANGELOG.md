# Changelog

## 0.2.2 — 2026-07-19

Round-3 fixes from the first live `/grok:review` — the plugin reviewed itself. Grok reported 11 findings against v0.2.0..HEAD; adversarial verification confirmed 5 and partially confirmed 6 (nothing refuted), and every worthwhile fix landed.

- `/grok:result` Fallback 2 now also covers list-derived ids when the primary and `.cwd` lookups miss — `sessions list` can return a sibling or parent worktree's session whose transcript lives under that worktree's group dir (empirically reproduced false degradation)
- `/grok:result` validates the session id before any path or glob use (UUID shape; no `/` or `..`), refuses to fabricate an id when `grok sessions list -n 1` finds no sessions, and binds the URL-encoding one-liner to the repo cwd in the same Bash call
- `/grok:transfer` names the `GROK_CLAUDE_SESSIONS_ENABLED` env override (env var > config.toml > default-on) and spells out the three-way `sessions`-key detection: `false` → disabled, `true` → enabled (explicit), key/file absent → enabled (default)
- `/grok:status` wording no longer implies the command injects a default — grok itself defaults to 20 when neither `-n` nor `--limit` is given
- Validation suite hardened against the four mutation escapes the review exposed: literal no-`--import-claude-session` anchor, body-scoped `-n`/`--limit` passthrough check, occurrence-counted `--dir`/`--all` guard, and three new result.md boundary assertions matching the new wording

## 0.2.1 — 2026-07-19

Codex command parity, part 2: grok-native session-bridge commands — every codex command now has a grok equivalent or a documented N/A.

- New `/grok:status` — recent Grok sessions for the current repo via cwd-scoped `grok sessions list` (`-n`/`--limit` passthrough); backgrounded delegations are tracked by the Claude Code harness itself (codex's job queue has no grok counterpart)
- New `/grok:result` — prints a finished session's `summary.json` metadata and final assistant output from the on-disk transcript under `${GROK_HOME:-~/.grok}/sessions/`; strictly read-only with a hard file-access boundary (nothing outside the session directory, never `auth.json`), never resumes, and degrades to advising `grok -r <id>` if the on-disk layout differs
- New `/grok:transfer` — preflight + guided handoff into Grok's interactive `/resume` picker via the foreign-session scanner; requires `[compat.claude] sessions` (default on) and the `resume-claude` skill; disclosed as interactive-only and experimental ("staged" per grok's docs)
- `cancel` documented as N/A in the README command table — background delegations are Claude Code tasks, stopped from Claude Code; destructive `grok sessions delete <id>` footnote added
- `/grok:setup` ready line now includes `/grok:adversarial-review`
- README Commands table completed — the full codex command surface is now covered

## 0.2.0 — 2026-07-19

Codex command parity, part 1: a runtime-free command surface driving the existing grok-worker courier.

- New `/grok:review` — read-only Grok review of local git state, with scope resolution (`auto|working-tree|branch`, `--base <ref>`) and the git context embedded in the prompt
- New `/grok:adversarial-review` — the same read-only machinery with challenge-the-design framing: architecture, assumptions, simpler alternatives — not a line-level bug hunt
- New `/grok:rescue` — delegate investigation, an explicit fix, or follow-up work to grok-worker; write-capable by default, honors `--read-only`, `--resume|--fresh`, `--model`, `--effort`, `--bg`
- New `/grok:setup` — preflight: binary, version, auth state, config defaults, and the rate-limit caveat
- Review commands inherit the MCP-hardened read-only flag set and never weaken it
- Setup checks auth by file existence only and never prints credentials
- README gains a Commands table under Usage
- `/grok:delegate` unchanged as the general-purpose entry
- Remaining codex commands (status/result/transfer/cancel) tracked for 0.2.1 as grok-native session-bridge equivalents or documented N/A

## 0.1.3 — 2026-07-19

Community governance and a durable author contact channel.

- `author.email` added to both manifests (marketplace.json owner + plugin entry, plugin.json author) — the runtime-visible change driving this release
- CONTRIBUTING.md: bugs go direct to PR with reproduction steps or a grok-doc citation; enhancements/features are issue-first and wait for maintainer go-ahead
- CODEOWNERS (maintainer auto-review) plus bug-report/enhancement issue templates and a PR template
- Two-job CI workflow: `manifest-sanity` (dependency-free, required-candidate) and an informational `claude plugin validate` run

## 0.1.2 — 2026-07-18

Second cross-AI review round (Codex; findings adversarially verified before fixing).

- Read-only mode now blocks MCP meta-tools (`search_tool`, `use_tool`) and adds a `--deny 'MCPTool(*)'` backstop that survives `--yolo` — previously a nominally read-only run could invoke write-capable MCP tools
- GSD patch guide: both invocations carry the same MCP hardening; the "strictly read-only" claim replaced with an accurate per-flag explanation
- GSD patch guide: grok's exit status is captured and checked, so a crash with partial output is no longer accepted as a successful review
- Changed-files report gains a "No-longer-present entries" section (`comm -23`) and documents the same-status-modification caveat
- `--bg` added to the routing-flag strip list
- Rate-limit errors are now distinguished from authentication failures in the courier's guidance
- Documented that fixed `/tmp` paths in the GSD patch follow GSD's stock reviewer convention (hardening belongs upstream)

## 0.1.1 — 2026-07-18

First cross-AI review round (Grok + Codex; six findings, all verified before fixing) plus the version bump that makes updates reachable — Claude Code keys plugin updates on the version string.

- Disclosed the `--yolo` always-approve risk of write-capable delegation in README and agent
- Fixed the mktemp template (BSD/macOS only randomizes trailing `X`s; the suffixed form created one shared literal file) and added prompt-file cleanup
- Trivial-question shortcut now routes through a prompt file — inlining questions into shell quoting broke on embedded quotes and executed `$()`/backticks
- Changed-files report now diffs against a pre-run baseline instead of attributing pre-existing dirty files to Grok
- Documented `grok sessions list` for `-r <id>` discovery and `-c`'s per-working-directory scope
- GSD patch guide captures grok's stderr to a sidecar log instead of discarding it, and the failure stub includes install/auth hints
- Branding updated for the SpaceXAI rebrand

## 0.1.0 — 2026-07-17

Initial release: `grok-worker` courier agent, `delegate` skill, CLAUDE.md delegation rules doc, GSD `/gsd:review` patch guide.
