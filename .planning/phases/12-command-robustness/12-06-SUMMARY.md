---
phase: 12-command-robustness
plan: 06
subsystem: security
tags: [containment, symlink, path-canonicalization, markdown-instruction-plugin, validator]

# Dependency graph
requires:
  - phase: 12-command-robustness (plans 01-05)
    provides: "/grok:result directory-level containment check (D-02), UUID grammar (D-03), export-primary retrieval (D-01), Source: disclosure (D-04); check_result_boundaries() validator function; 12-REVIEW.md CR-01 finding; 12-VERIFICATION.md gap statement"
provides:
  - "Per-leaf (per-file) containment gate in /grok:result: summary.json, updates.jsonl, and the Fallback-1 .cwd read are each individually realpath-canonicalized and prefix-verified against the sessions root immediately before being opened"
  - "Region-scoped, mutation-proven validator anchor in check_result_boundaries() covering all three leaf-read sites independently"
  - "D-02 must-have truth now fully met (was partial per 12-VERIFICATION.md); 12-REVIEW.md CR-01 (the phase's only critical finding) closed"
affects: [phase-12-verification-reverify, future-phases-touching-grok-commands-result-md]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Per-leaf containment: reuse the exact directory-level realpath+prefix-compare python3 -c idiom, retargeted to a leaf path, immediately before that specific file is opened — not a new mechanism, the same one-liner applied at each read site"
    - "Region-scoped validator anchoring: text.find(marker) -> text.find(next_marker) slicing isolates each require() to its own section so an unrelated mention of the same words elsewhere in the file cannot satisfy the assertion"

key-files:
  created: []
  modified:
    - plugins/grok/commands/result.md
    - tests/validate_plugin.py

key-decisions:
  - "Used the identical `os.environ['HOME']` + os.sep-guarded realpath/prefix idiom already present in the directory-level check (result.md:36-43) for all three leaf gates, rather than introducing a second containment mechanism — keeps the file's containment story internally consistent and reuses code the validator already region-scopes correctly"
  - "Named the per-leaf failure distinctly (\"session file resolves outside the sessions tree\") from the directory-level failure (\"session directory resolves outside the sessions tree\") so a user-facing refusal message can indicate which layer caught the escape, while both refuse-and-report identically (read NOTHING, NOT a degrade case, point at `grok -r <session-id>`)"
  - "Placed the shared greppable marker phrase \"before it is opened\" at all three read sites (not just in the Containment check section prose) so the validator's per-region anchors are genuinely testing each site's own gate, not a single central mention"

requirements-completed: [REQ-05]

# Metrics
duration: ~20min
completed: 2026-07-28
---

# Phase 12 Plan 06: Close D-02 leaf-file containment gap in /grok:result Summary

**Per-file (leaf) containment gate added to `/grok:result` for summary.json, updates.jsonl, and the Fallback-1 `.cwd` read — closing the symlinked-leaf exfiltration vector (12-REVIEW.md CR-01, the phase's only critical finding) that the directory-only check missed, with a mutation-proven region-scoped validator anchor covering all three sites independently.**

## Performance

- **Duration:** ~20 min
- **Completed:** 2026-07-28T06:12:13Z
- **Tasks:** 1/1
- **Files modified:** 2

## Accomplishments

- Closed the sole remaining Phase 12 verification gap: `/grok:result`'s containment check previously canonicalized and prefix-verified only the session **directory**, then read `summary.json`, `updates.jsonl`, and the Fallback-1 `.cwd` marker via mechanisms (Read tool / `python3 open()` / `tail`) that all follow symlinks — meaning a `summary.json` symlinked to `${GROK_HOME}/auth.json` inside an otherwise-CONTAINED directory would have been read and printed as "metadata," directly violating the command's own stated boundary.
- Applied the identical realpath + prefix-verify one-liner idiom already used for the directory (result.md:36-43) to each of the three leaf reads individually, immediately before that file is opened, refusing-and-reporting (read NOTHING, NOT a degrade case) on any escaping leaf — exactly mirroring how the directory-level escape is handled.
- Retargeted the Fallback-1 `.cwd` bullet, which previously asserted reading `.cwd` markers "stays within the boundary" with no actual per-file check (the exact assumption 12-REVIEW.md CR-01 identified as the hole) — that `.cwd` read happens during locate, before the directory-level check even runs, so it now gets its own gate.
- Extended `check_result_boundaries()` with a new anchor, region-scoped independently to the Locating/Fallback-1 region, the Metadata region, and the Retrieving region, so a single central mention of the per-leaf language cannot satisfy the assertion for all three sites — each read site's own gate must be present.
- Mutation-proved the new anchor: reverted each of the three per-leaf gates in result.md in turn (restoring the old directory-only prose) and confirmed `python3 tests/validate_plugin.py` failed, naming the specific anchor for that site; restored and confirmed the suite returns to 22/22 green after each mutation.

## Task Commits

Each task was committed atomically:

1. **Task 1: Add per-leaf (per-file) containment before every result.md leaf read, with a lockstep mutation-proven validator anchor** - `110bc87` (fix)

_No plan-metadata commit in worktree mode — SUMMARY.md is committed separately per the parallel-executor protocol (STATE.md/ROADMAP.md updates are owned by the orchestrator after wave merge)._

## Files Created/Modified

- `plugins/grok/commands/result.md` - Added per-leaf containment gate (realpath canonicalize + prefix-verify against `${GROK_HOME:-$HOME/.grok}/sessions/`, immediately before opening) at all three leaf-read sites: Fallback-1 `.cwd` (Locating section), `summary.json` (Metadata section), `updates.jsonl` (Retrieving/FALLBACK section). Updated the Containment check section prose to state the gate runs PER FILE (leaf), not only per-directory, while preserving every existing directory-level anchor phrase verbatim.
- `tests/validate_plugin.py` - Extended `check_result_boundaries()` (same function, no new CHECKS registry entry) with a region-scoped anchor requiring the shared "before it is opened" marker in the Locating/Fallback-1, Metadata, and Retrieving regions independently, plus the distinct "session file resolves outside the sessions tree" leaf-failure wording in the Containment check region.

## Decisions Made

- Reused the exact existing `python3 -c` realpath/prefix-compare idiom (same `os.environ['HOME']` fallback, same `+ os.sep` trailing-separator guard) for all three leaf gates rather than inventing a new containment mechanism or reaching for `O_NOFOLLOW`/`--no-dereference` — consistent with the plan's constraint to reuse the identical one-liner already proven at the directory level, and keeps the markdown-instruction file's containment story in one idiom throughout.
- Gave the per-leaf failure its own wording ("session file resolves outside the sessions tree") distinct from the directory-level failure ("session directory resolves outside the sessions tree") so the refusal message can name which layer caught the escape, while explicitly stating both are refuse-and-report / read NOTHING / NOT a degrade case, identically.
- Put the "before it is opened" marker phrase directly at each of the three read sites (not only once in the central Containment check section) so the validator's region-scoped anchors are testing that each site actually carries its own gate — this is what makes the anchor mutation-proof per-site rather than satisfiable by a single stray sentence.

## Deviations from Plan

None - plan executed exactly as written. The plan's scope (WR-01..WR-04 explicitly out of scope) was respected; no other files were touched.

## Issues Encountered

None. The plan's `<read_first>` references (12-REVIEW.md CR-01, 12-VERIFICATION.md gaps block, 12-PATTERNS.md containment-check and region-scoped-anchor templates) contained the exact idiom and anchoring pattern needed, so implementation was a direct application rather than requiring new design decisions.

## Verification

- `python3 tests/validate_plugin.py` → `RESULT: PASS (22/22 groups green)` after the fix (group count unchanged, as planned — `check_result_boundaries()` was extended in place, no new CHECKS registry entry added).
- Mutation-proof (performed live during execution, not just documented): each of the three per-leaf gates was reverted independently (restoring old pre-fix prose) and the suite was re-run:
  - Reverting the Fallback-1 `.cwd` gate → `FAIL (21/22 groups green)`, naming `result.md: the Fallback-1 \`.cwd\` read happens before the directory-level containment check runs at all...`
  - Reverting the summary.json gate → `FAIL (21/22 groups green)`, naming `result.md: the summary.json metadata read must be per-file (leaf) canonicalized...`
  - Reverting the updates.jsonl gate → `FAIL (21/22 groups green)`, naming `result.md: the updates.jsonl FALLBACK read must be per-file (leaf) canonicalized...`
  - After each mutation, the file was restored (byte-identical diff confirmed against the pre-mutation good copy) and the suite re-confirmed `PASS (22/22 groups green)`.
- Manual/conceptual: a session dir that resolves inside `sessions/` but contains `summary.json` symlinked to `${GROK_HOME}/auth.json` now yields `ESCAPED` at the leaf gate (realpath of the symlink target does not equal or start-with the sessions root) and is refused-and-reported — read nothing — instead of being printed as "metadata."

## Threat Flags

No new threat surface introduced. This plan closes T-12-02 (leaf) as declared in the plan's own `<threat_model>` and does not touch any file, network, or auth surface beyond `plugins/grok/commands/result.md` and its validator. T-12-02-TOCTOU (the realpath-to-open non-atomicity residual) remains explicitly accepted-and-documented in the plan, not mitigated in-artifact — unchanged by this plan.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- D-02's containment-before-read guarantee for `/grok:result` is now fully met at both the directory AND per-file (leaf) level — the phase's last unmet must-have (was 26/27 per 12-VERIFICATION.md) is closed.
- 12-REVIEW.md CR-01 (the review's only critical finding) is resolved.
- `python3 tests/validate_plugin.py` is green 22/22 with no outstanding mutation-proof gaps for this control.
- Recommended next step: re-run phase 12 verification (`/gsd:verify-phase 12` or equivalent) to confirm 27/27 truths now verified, then the orchestrator can close out the phase. WR-01..WR-04 (non-blocking warnings from 12-REVIEW.md) remain deferred/out of scope for this plan, as declared.

---
*Phase: 12-command-robustness*
*Completed: 2026-07-28*
