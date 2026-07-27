---
description: Run a Grok review that challenges the implementation approach and design choices
argument-hint: '[--base <ref>] [--scope auto|working-tree|branch] [--model <id>] [--effort <level>] [focus ...]'
disable-model-invocation: true
allowed-tools: Read, Glob, Grep, Bash(git:*), Agent, AskUserQuestion
---

Run an adversarial Grok review through the grok-worker courier in read-only mode.
Position it as a challenge review that questions the chosen implementation, design choices, tradeoffs, and assumptions.
It is not just a stricter pass over implementation defects.

Raw slash-command arguments:
`$ARGUMENTS`

Core constraint:
- This command is review-only.
- Do not fix issues, apply patches, or suggest that you are about to make changes.
- This command must pass `--read-only` to grok-worker and must never weaken the worker's read-only flag set.
- Never restate or override the worker's raw grok flags — pass the `--read-only` routing flag and let the worker enforce its own MCP-hardened read-only tool set.
- Keep the framing focused on whether the current approach is the right one, what assumptions it depends on, and where the design could fail under real-world conditions.

Scope resolution (same as `/grok:review`):
- `--scope working-tree`: review uncommitted work — staged + unstaged + untracked — gathered via `git status --short --untracked-files=all`, `git diff`, and `git diff --cached`.
- `--scope branch`: review commits vs the base — `git diff $(git merge-base <base> HEAD)...HEAD`.
- `--scope auto` (the default): working-tree if `git status --short --untracked-files=all` is non-empty, otherwise branch.
- `--base <ref>` sets the branch-scope base. Default base: the repo's default branch (`git symbolic-ref --short refs/remotes/origin/HEAD`, falling back to `main` or `master`).
- Treat untracked files as reviewable work. Only conclude there is nothing to review when the resolved scope is actually empty. When in doubt, run the review.
- Use `AskUserQuestion` at most once, and only when the review target is genuinely ambiguous — for example, the working tree is dirty AND an explicit `--base` was given (ask which target to review), OR the resolved scope is genuinely empty (see the empty-scope rule below). Whichever of these two triggers fires first spends the single question; never ask twice in one invocation.
- **Malformed input — error-and-stop, never guess (D-08):** an invalid `--scope` value (anything other than `auto`, `working-tree`, or `branch`) is a one-line error-and-stop naming the bad value and the accepted set (`auto`, `working-tree`, `branch`) — no preamble, no follow-up question, the worker is never spawned. Before using `--base <ref>` for anything, validate it with `git rev-parse --verify --quiet <ref>^{commit}`: exit 1 with empty stdout means the ref does not resolve — one-line error-and-stop naming the unresolvable ref and instructing the user to pass an explicit, valid `--base <ref>`. No lenient origin-prefix or `-`-prefix retry.
- **Repo-state edge cases (D-09):** not-a-repo (`git status` exits 128, stderr present) always error-and-stop relaying git's stderr — the worker is never spawned. Unborn HEAD (repo initialized, zero commits): `git status`/`git diff`/`git diff --cached` all exit 0 on unborn HEAD, so with `--scope auto` or `--scope working-tree`, degrade to reviewing the untracked working tree with a disclosed note — "unborn HEAD — reviewing untracked working tree; branch scope unavailable" — consistent with the "treat untracked files as reviewable work" rule above. Explicit `--scope branch` on unborn HEAD → error-and-stop (`git rev-parse HEAD` exits 128 — no HEAD to diff against).
- **Base-resolution dead ends (D-10), self-authored message — never guess a base:** if the default-branch fallback chain is exhausted (`git symbolic-ref --short refs/remotes/origin/HEAD` fails AND `git show-ref --verify --quiet refs/heads/main` AND `refs/heads/master` both fail), OR `git merge-base <base> HEAD` fails on disjoint histories, error-and-stop. Both of these cases produce EMPTY stderr from git (`show-ref --quiet` is silent by design; disjoint-history `merge-base` exits 1 with stdout AND stderr both empty) — so the message cannot relay git's own text; self-author a one-line report, e.g. "could not determine a default branch (origin/HEAD, main, master all absent)" or "no common history between `<base>` and HEAD", each instructing the user to pass an explicit `--base <ref>`.
- **Empty-diff vs git-error, never conflated (D-11):** a nonzero git exit is a git error — error-and-stop relaying git's stderr (except the two self-authored cases above). A genuinely empty scope (both relevant git calls exit 0 with empty output) is NOT an error — before concluding nothing-to-review, spend the one allowed `AskUserQuestion` (shared budget above) offering to widen scope (e.g. branch → working-tree). The worker is never spawned for an empty scope either way.

Building the review prompt:
- Read-only grok has no shell — it cannot run `git diff` itself. So this command gathers the git context via `Bash(git:*)` and embeds it in the task text handed to the worker: the changed-file list plus the scoped diff.
- If the diff is very large (roughly more than 4000 lines), fall back to `--stat` plus the file list and tell Grok to read the files directly — its read-only tools include `read_file`.
- Framing — the only difference from `/grok:review`: direct Grok to challenge the chosen implementation and design. Question the architecture, hunt for simpler alternatives, attack assumptions, and ask where the design fails under real-world conditions. This is explicitly NOT a line-level bug hunt — state in the prompt that it is not just a stricter defect pass.
- Focus text from the arguments steers the challenge; append it verbatim — do not rewrite or weaken it.
- The prompt names the scope and the base ref and asks for findings ordered by severity with file:line references.

Spawning the worker:
- Use the `Agent` tool with `subagent_type: "grok:grok-worker"`, in the foreground.
- The subagent must be spawned via the `Agent` tool from this command body, not via Skill indirection — the command runs inline so the `Agent` tool stays in scope.
- The spawn prompt is the built review task plus the working directory plus the routing flags spelled out: always `--read-only`; add `--model <id>` / `--effort <level>` only if explicitly present in the raw arguments (pass them through verbatim — never invent model IDs).

Output rules:
- Relay Grok's findings clearly attributed ("Grok's review: ..."), ordered by severity.
- Do not fix anything, and do not silently editorialize — if you disagree with a finding, say so explicitly with attribution.
- If the worker reports a missing binary, an authentication problem, or a rate limit, relay its guidance and point the user at `/grok:setup`.
- If the courier report begins `Grok run: FAILED` (or the `Grok run: TIMEOUT` class), present it as a failed run and relay the failure — never summarize partial output as a review.
