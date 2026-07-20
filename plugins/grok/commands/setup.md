---
description: Check whether the local grok CLI is ready for delegation
argument-hint: ''
allowed-tools: Bash, Read
---

Check whether the local `grok` CLI is ready for delegation. Run the checks below via Bash and report each as a short checklist line.

Checks:

1. **Binary**: `command -v grok`. If missing, report it and print the install one-liner — `curl -fsSL https://x.ai/cli/install.sh | bash` — plus "run `grok` once interactively to authenticate". Still run checks 3–4 (they are file checks).
2. **Version**: `grok --version` (only if the binary exists).
3. **Auth state**: `[ -f "$HOME/.grok/auth.json" ]` — report "credentials file present" or "not authenticated — run `grok` once interactively to log in". Never print or read the contents of `~/.grok/auth.json` — it may contain tokens; existence check only. Do not use the Read tool on auth.json.
4. **Defaults**: if `~/.grok/config.toml` exists, grep it for the `model` and `effort` keys only and report their values; do not dump the whole file.
5. **Rate-limit caveat**: state that the free tier has usage limits that agentic runs can exhaust quickly, and that a usage-limit error is NOT an authentication failure — no re-login needed.
6. **Review-gate**: state explicitly that there is no review-gate equivalent — the grok plugin has no stop hook, so codex's `--enable-review-gate` has no counterpart here (which is why this command takes no flags).

Output rule:
- Present the checklist. If everything passes, say delegation is ready: `/grok:delegate`, `/grok:review`, `/grok:adversarial-review`, `/grok:rescue`.
