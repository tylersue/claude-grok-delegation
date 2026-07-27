# Changelog

## 0.3.5 — 2026-07-27

Command robustness (phase 12; findings #4/#5/#6) plus a courier-failure handling extension: session-result retrieval, setup/transfer path handling, and the review commands' git edge cases are all hardened, and every courier consumer now classifies failures instead of conflating them.

- `/grok:result` now attempts native `grok export` as the PRIMARY retrieval path, with the existing on-disk transcript locate+parse surviving verbatim as a disclosed FALLBACK on any export failure; session ids are validated against the strict version-agnostic 8-4-4-4-12 lowercase-hex UUID grammar (replacing the old loose "no `/` or `..`" check) before any path or glob use, on both argument-supplied and list-derived ids; a canonicalize-and-prefix-verify containment check runs before ANY read (refuse-and-report, read nothing, on escape); and an always-on `Source:` line discloses which retrieval path fired (finding #4)
- `/grok:setup` and `/grok:transfer` honor `GROK_HOME` everywhere instead of the tilde form that silently fails to expand when double-quoted; `/grok:setup` validates auth structurally (non-empty + JSON-parse validity, zero content exposure) and anchors its config greps to the real `[models]`/`[telemetry]` schema so they can no longer match or print a credential-shaped key (finding #5)
- `/grok:review` and `/grok:adversarial-review` now define explicit behavior for not-a-repo, unborn HEAD, no default branch, bad/ambiguous `--base`, `merge-base` failure, invalid `--scope`, and empty-diff vs. git-error, closing the git-state edge cases from finding #6
- All four courier consumers (`/grok:review`, `/grok:adversarial-review`, `/grok:rescue`, and the `delegate` skill) gained per-class failure handling — auth / rate-limit / TIMEOUT / generic — with a single generic-class-only auto-retry, replacing the previous conflated auth-or-rate-limit handling
- `tests/validate_plugin.py` gained mutation-proven coverage for all of the above, including a new `check_courier_failure_sync()` check group enforcing N=4 byte-identical fenced-block sync across the four courier-failure consumers

## 0.3.4 — 2026-07-25

Read-path-confinement gap closure (follow-up to 0.3.3 / finding #2). Post-release review and phase verification found a critical fail-closed contradiction in the shipped 0.3.3 read-only path, plus validator-anchor and disclosure gaps; all are fixed here.

- Read-only mode now REPLACES the base command's `--sandbox workspace` with `--sandbox strict` — exactly one `--sandbox` flag per invocation — so the primary review path (`/grok:review`, `/grok:adversarial-review`, `--read-only`) can no longer be handed a doubled `--sandbox` flag the CLI hard-rejects (`cannot be used multiple times`); that doubled-flag fail-closed path from 0.3.3 is fixed (GAP-1).
- The `delegate` skill's trivial-question path now discloses the self-healing degraded case honestly instead of falsely reporting `Sandbox: strict` when the sandbox could not actually be applied (GAP-2).
- Sandbox disclosure uses only the two declared vocabulary forms (`reads unconfined` / `writes unconfined`) instead of a slashed `reads/writes unconfined` hybrid; the README network-control statement is corrected for write-capable runs (GAP-2).
- `tests/validate_plugin.py` gained region-scoped SKILL.md flag and retry-signature anchors, plus a docs `--sandbox workspace` sync anchor, closing three proven mutation escapes (GAP-3).
- Minor doc/id fixes: dangling README `web_search` reference, ID-reuse cleanup in `grok-worker.md`, and small wording corrections (GAP-4).

## 0.3.3 — 2026-07-24

Read-path confinement (finding #2): grok invocations now carry a kernel-enforced `--sandbox` profile in addition to the existing tool-level allowlist, so a review or implementation task can no longer be talked into reading or writing outside its intended scope just by asking nicely in the diff/task text. Live-verified on the installed CLI (grok 0.2.111): an outside-workspace read attempt under `strict` came back kernel-denied (`Permission denied`), and the resume-conflict preflight rejection this repo's own pre-phase sessions were guaranteed to hit was reproduced and gracefully degraded, never failed closed.

- Read-only mode runs under `--sandbox strict` — kernel-enforced (Landlock on Linux, Seatbelt on macOS) read confinement layered under the existing `--tools`/`--disallowed-tools`/`--deny` allowlist: the allowlist blocks tool *availability*, the sandbox blocks filesystem *access* even for allowed tools. A review can no longer be talked into reading `~/.ssh`, a parent repo, or a stray outside-workspace `.env` file.
- Write-capable `--yolo` runs carry `--sandbox workspace` — writes confined to CWD + `~/.grok/` + temp dirs while reads stay unrestricted; a delegated implementation task can no longer write to `~/.ssh`, other repos, or system paths.
- New preflight-rejection detection-and-retry mechanic: on a nonzero exit matching one of three known signatures (unknown/renamed sandbox profile, resume-conflict against a pre-existing session, or an old-CLI unrecognized `--sandbox` flag), the courier retries the identical command with `--sandbox` dropped and discloses the degraded state — never fails closed.
- Always-on `Sandbox:` disclosure line under every `Grok run:` status line (`strict` / `workspace` / `UNAVAILABLE — reads|writes unconfined (<reason>)`), on both entry points (`grok-worker` agent and the `delegate` skill's trivial-question path).
- README "Data egress & privacy" section gains the confinement boundary statement — what `--sandbox strict`/`workspace` confine and what they don't (in-workspace files stay readable by design; child-process network blocking is Linux-only, a no-op on macOS) — plus a pointer to grok's own custom `sandbox.toml` `deny`-glob opt-in for power users who want in-workspace secret denial.
- `docs/claude-md-rules.md` re-synced to name `--sandbox strict`/`workspace` alongside the existing flag set, closing a zero-validator-coverage drift risk the same class as finding #10.
- `tests/validate_plugin.py` gained mutation-proven, region-scoped anchors for the sandbox flags, the disclosure-line contract, the retry signatures, and the doc sync (18 → 20 check groups).

## 0.3.2 — 2026-07-21

Security hardening: honest data-egress disclosure, privacy-tooling surfaced in `/grok:setup`, and courier hardening against prompt-injection and failure-masking (findings #1/#3/#7). Consolidates the accumulated 0.2.2→0.3.2 work into a single release — no 0.3.0/0.3.1 were separately tagged.

- `cancel` row removed from the README command table — there is no grok job queue; background delegations are Claude Code tasks, stopped from Claude Code (the `grok sessions delete <id>` destructive-delete footnote stays)
- New "Data egress & privacy" README section disclosing the intended inference-API channel, the grok CLI's separate per-turn trace-upload pipeline, the July 2026 grok-build repo-upload incident, and how to minimize exposure
- `docs/claude-md-rules.md` read-only block now lists the full MCP-hardened flag set (`--tools` + `--disallowed-tools` + `--deny 'MCPTool(*)'`), matching `grok-worker.md` exactly
- Manual-install instructions rewrite the copied skill's frontmatter so its declared `name:` matches its directory (`name: grok`)
- `/grok:setup` surfaces the local data-sharing/telemetry posture (`config.toml` key-only grep for `telemetry`/`trace_upload`/`feedback` + the four hardening env vars) with an always-on opt-out block (`grok /privacy opt-out`, the env vars, ZDR, 30-day retention), and discloses it cannot read the account-level "coding data sharing" flag locally (that flag lives in `~/.grok/auth.json`, which setup never reads)
- `/grok:transfer` gains a forward-pointer to `/grok:setup`'s privacy check
- Non-evaluating Write-tool prompt-file delivery in `grok-worker.md` and the SKILL trivial-question path — task text never touches a shell command line, heredoc, or redirect (finding #1)
- `GROK_EXIT=$?` captured before any cleanup/reporting, a mandatory greppable `Grok run: SUCCESS/FAILED/TIMEOUT` status line, partial output on failure/timeout labeled "partial output — not a completed result", and the confirmed apostrophe-mismatch classification bug fixed (finding #3)
- Temp-file cleanup guaranteed three independent ways — `trap` on EXIT/TERM/INT, a follow-up cleanup rule for SIGKILL, and an age-gated preflight sweep (finding #3)
- Explicit, shell-safe routing-flag grammar (leading-only recognition, `--` terminator, duplicate/missing-value/`--resume`+`--fresh` error-and-stop, `--effort` list-checked, every value passed as its own separately quoted shell argument) written identically into `grok-worker.md` and `SKILL.md`, enforced byte-identical by a validator sync check (finding #7)
- `/grok:review`, `/grok:adversarial-review`, and `/grok:rescue` all branch on the `Grok run: FAILED`/`TIMEOUT` status line before ever presenting a report as a completed result
- `tests/validate_plugin.py` gained the D-01..D-15 assertions covering all of the above, each mutation-proven against a deliberately broken copy

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
