# Changelog

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
