---
description: Run a Grok code review against local git state
argument-hint: '[--base <ref>] [--scope auto|working-tree|branch] [--model <id>] [--effort <level>] [focus ...]'
disable-model-invocation: true
allowed-tools: Read, Glob, Grep, Bash(git:*), Agent, AskUserQuestion
---

Run a Grok review through the grok-worker courier in read-only mode.

Raw slash-command arguments:
`$ARGUMENTS`

Core constraint:
- This command is review-only.
- Do not fix issues, apply patches, or suggest that you are about to make changes.
- This command must pass `--read-only` to grok-worker and must never weaken the worker's read-only flag set.
- Never restate or override the worker's raw grok flags — pass the `--read-only` routing flag and let the worker enforce its own MCP-hardened read-only tool set.

Scope resolution:
- `--scope working-tree`: review uncommitted work — staged + unstaged + untracked — gathered via `git status --short --untracked-files=all`, `git diff`, and `git diff --cached`.
- `--scope branch`: review commits vs the base — `git diff $(git merge-base <base> HEAD)...HEAD`.
- `--scope auto` (the default): working-tree if `git status --short --untracked-files=all` is non-empty, otherwise branch.
- `--base <ref>` sets the branch-scope base. Default base: the repo's default branch (`git symbolic-ref --short refs/remotes/origin/HEAD`, falling back to `main` or `master`).
- Treat untracked files as reviewable work. Only conclude there is nothing to review when the resolved scope is actually empty. When in doubt, run the review.
- Use `AskUserQuestion` at most once, and only when the review target is genuinely ambiguous — for example, the working tree is dirty AND an explicit `--base` was given (ask which target to review). Otherwise never ask.

Building the review prompt:
- Read-only grok has no shell — it cannot run `git diff` itself. So this command gathers the git context via `Bash(git:*)` and embeds it in the task text handed to the worker: the changed-file list plus the scoped diff.
- If the diff is very large (roughly more than 4000 lines), fall back to `--stat` plus the file list and tell Grok to read the files directly — its read-only tools include `read_file`.
- The prompt names the scope and the base ref, appends any focus text from the arguments verbatim, and asks for findings ordered by severity with file:line references.

Spawning the worker:
- Use the `Agent` tool with `subagent_type: "grok:grok-worker"`, in the foreground.
- The subagent must be spawned via the `Agent` tool from this command body, not via Skill indirection — the command runs inline so the `Agent` tool stays in scope.
- The spawn prompt is the built review task plus the working directory plus the routing flags spelled out: always `--read-only`; add `--model <id>` / `--effort <level>` only if explicitly present in the raw arguments (pass them through verbatim — never invent model IDs).

Output rules:
- Relay Grok's findings clearly attributed ("Grok's review: ..."), ordered by severity.
- Do not fix anything, and do not silently editorialize — if you disagree with a finding, say so explicitly with attribution.
- If the worker reports a missing binary, an authentication problem, or a rate limit, relay its guidance and point the user at `/grok:setup`.
