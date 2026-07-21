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
7. **Local privacy/telemetry overrides**: if `~/.grok/config.toml` exists, grep it for the `telemetry`, `trace_upload`, and `feedback` keys only and report each value found, or "not set (default)" per key; do not dump the whole file. If the file is absent, report "config.toml not found — defaults apply". This check reads `config.toml` exclusively; it never touches the credentials file named in item 3.
8. **Env-var overrides**: report the current shell's value of each of `GROK_TELEMETRY_ENABLED`, `GROK_TELEMETRY_TRACE_UPLOAD`, `GROK_FEEDBACK_ENABLED`, `DISABLE_ERROR_REPORTING` via `printenv`, each as "set to `<value>`" or "not set". Label items 7 and 8 explicitly as **local telemetry/trace-upload overrides** — these do NOT reflect the account-level coding-data-sharing status; never render them as a "coding data sharing: enabled/disabled" conclusion.

Account-level disclosure (always state this, installed grok CLI is 0.2.x):
- The authoritative account-level "coding data sharing" flag is not locally readable — it lives in `~/.grok/auth.json`, which this command never opens/reads (existence-check only, per item 3). The default is opt-in unless you've run `grok /privacy opt-out` or your account has ZDR. The one authoritative way to see live status is the interactive `grok /privacy`.

Output rule:
- Present the checklist. If everything passes, say delegation is ready: `/grok:delegate`, `/grok:review`, `/grok:adversarial-review`, `/grok:rescue`.
- Always print this opt-out block, unconditionally, regardless of any detected state above:
  - `grok /privacy opt-out` — opt out of coding data sharing from inside the CLI
  - `GROK_TELEMETRY_ENABLED=0` — disable telemetry
  - `GROK_TELEMETRY_TRACE_UPLOAD=0` — disable the per-turn trace-upload pipeline specifically
  - `GROK_FEEDBACK_ENABLED=0` — disable feedback collection
  - `DISABLE_ERROR_REPORTING=1` — disable error reporting
  - Enterprise-plan accounts (accounts with an enterprise relationship) can request Zero Data Retention (ZDR)
  - Default API retention for the inference channel is 30 days
