---
description: Check whether the local grok CLI is ready for delegation
argument-hint: ''
allowed-tools: Bash, Read
---

Check whether the local `grok` CLI is ready for delegation. Run the checks below via Bash and report each as a short checklist line.

Checks:

1. **Binary**: `command -v grok`. If missing, report it and print the install one-liner — `curl -fsSL https://x.ai/cli/install.sh | bash` — plus "run `grok` once interactively to authenticate". Still run checks 4–5 (they are file checks).
2. **Version**: `grok --version` (only if the binary exists).
3. **GROK_HOME**: `printenv GROK_HOME`. If set, report `GROK_HOME set — using <path>` (substituting the reported value) — every path check below (items 4/5/8) resolves via `${GROK_HOME:-$HOME/.grok}`, so this determines where they actually look. If unset, say nothing for this item (silent — do not report "not set").
4. **Auth state**: `[ -s "${GROK_HOME:-$HOME/.grok}/auth.json" ]` (non-empty; the `$HOME`-form default, never the tilde form, which silently fails to expand when double-quoted). If that test fails, report "not authenticated — run `grok` once interactively to log in". If it passes, run a structural JSON-parse validity probe whose ONLY observable output is `valid`/`invalid` via exit code — never file content, never a traceback:
   ```
   python3 -c "
   import json, sys
   try:
       json.load(open(sys.argv[1]))
       print('valid')
   except Exception:
       print('invalid')
       sys.exit(1)
   " "${GROK_HOME:-$HOME/.grok}/auth.json"
   ```
   Report "credentials file present and parseable" when the probe prints `valid`, or "credentials file present but not valid JSON (empty/truncated/corrupt) — run `grok` once interactively to re-authenticate" when it prints `invalid`. The probe parses the file for JSON validity only — its only observable output is valid/invalid, and file contents never enter context. Never print or read the contents of `~/.grok/auth.json` — it may contain tokens; only the non-empty test and the exit-code-only validity probe above touch it. Do not use the Read tool on auth.json.
5. **Defaults**: if `${GROK_HOME:-$HOME/.grok}/config.toml` exists, report the ACTIVE default model/effort from the real schema — the `[models]` section's `default` and `default_reasoning_effort` keys (NOT a bare `model`/`effort` line, and NOT a per-provider `[model.<id>]` override section, which is a different, more advanced surface). Use a TOML-section-aware reader so the match cannot land on the wrong section or on `model_api_token`:
   ```
   python3 -c "
   import sys
   path = sys.argv[1]
   in_models = False
   found = {}
   try:
       with open(path) as f:
           for line in f:
               s = line.strip()
               if s.startswith('[') and s.endswith(']'):
                   in_models = (s == '[models]')
                   continue
               if in_models and '=' in s:
                   key, _, val = s.partition('=')
                   key = key.strip()
                   if key in ('default', 'default_reasoning_effort'):
                       found[key] = val.strip()
       print('default=' + found.get('default', 'not set'))
       print('default_reasoning_effort=' + found.get('default_reasoning_effort', 'not set'))
   except FileNotFoundError:
       print('config.toml not found')
   " "${GROK_HOME:-$HOME/.grok}/config.toml"
   ```
   Do not dump the whole file. If the file is absent, report "config.toml not found — defaults apply".
6. **Rate-limit caveat**: state that the free tier has usage limits that agentic runs can exhaust quickly, and that a usage-limit error is NOT an authentication failure — no re-login needed.
7. **Review-gate**: state explicitly that there is no review-gate equivalent — the grok plugin has no stop hook, so codex's `--enable-review-gate` has no counterpart here (which is why this command takes no flags).
8. **Local privacy/telemetry overrides**: if `${GROK_HOME:-$HOME/.grok}/config.toml` exists, report each of the following by the REAL schema layout — `telemetry` and `feedback` (both live under `[features]`), and `trace_upload` (lives under a SEPARATE `[telemetry]` section, alongside `events_url`, a real credential-shaped `events_api_key`, and `mixpanel_enabled`) — report each value found or "not set (default)" per key. Single-line matches only for every one of these keys — never `-A`/`-B`/`-C` context flags anywhere near `trace_upload`, because `events_api_key` sits two lines away in `[telemetry]` and a context-flag match would also print it. Do not dump the whole file. If the file is absent, report "config.toml not found — defaults apply". This check reads `config.toml` exclusively; it never touches the credentials file named in item 4.
9. **Env-var overrides**: report the current shell's value of each of `GROK_TELEMETRY_ENABLED`, `GROK_TELEMETRY_TRACE_UPLOAD`, `GROK_FEEDBACK_ENABLED`, `DISABLE_ERROR_REPORTING` via `printenv`, each as "set to `<value>`" or "not set". Label items 8 and 9 explicitly as **local telemetry/trace-upload overrides** — these do NOT reflect the account-level coding-data-sharing status; never render them as a "coding data sharing: enabled/disabled" conclusion.

Account-level disclosure (always state this, installed grok CLI is 0.2.x):
- The authoritative account-level "coding data sharing" flag is not locally readable — it lives in `~/.grok/auth.json`, which this command never opens for content (item 4's non-empty test plus exit-code-only JSON-validity probe, never a content read). The default is opt-in unless you've run `grok /privacy opt-out` or your account has ZDR. The one authoritative way to see live status is the interactive `grok /privacy`.

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
