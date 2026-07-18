# Adding Grok as a `/gsd:review` reviewer (GSD patch guide)

[GSD (get-shit-done)](https://github.com/open-gsd/gsd-core) ships a cross-AI plan review command, `/gsd:review`, with a hardcoded reviewer roster (Gemini, Claude, Codex, CodeRabbit, OpenCode, Qwen, Cursor, plus local model servers). Grok is not in the stock roster. This guide patches it in as a first-class peer: `/gsd:review --grok` reviews the phase plan with Grok and writes a `## Grok Review` section into the phase's REVIEWS.md.

> **Caveats before you start**
>
> - This patches files that GSD owns. **`/gsd:update` may overwrite them — reapply the patch after every GSD update.**
> - Written against the GSD 1.x layout (`~/.claude/get-shit-done/workflows/`, npm package `get-shit-done-cc`, which resolves reviewer model config through `gsd-sdk`). GSD 2 (`@opengsd/gsd-core`) restructured these files (workflows live under `gsd-core/workflows/` and config resolves through a `gsd_run` shim) — the same seven edits apply conceptually, but the anchor lines differ.
> - An upstream contribution would be the durable fix; GSD's contribution process is issue-first (see their CONTRIBUTING.md).

Both files live in your GSD install directory — for a global GSD 1.x install: `~/.claude/get-shit-done/workflows/`.

## Patch 1–6: `workflows/review.md`

**1. Reviewer detection** — in the "Check each CLI" block, after the `codex` line, add:

```bash
command -v grok >/dev/null 2>&1 && echo "grok:available" || echo "grok:missing"
```

**2. Flag documentation** — in the reviewer-flag list, add:

```markdown
- `--grok` → include Grok (SpaceXAI Grok Build)
```

**3. Install hints** — in the install-instructions list, add:

```markdown
- grok: https://x.ai/cli (`curl -fsSL https://x.ai/cli/install.sh | bash`)
```

**4. Model config** — alongside the other `*_MODEL` config reads, add:

```bash
GROK_MODEL=$(gsd-sdk query config-get review.models.grok 2>/dev/null | jq -r '.' 2>/dev/null || true)
```

This makes the model configurable via `review.models.grok` in GSD config; unset means the grok CLI's own default model.

**5. Invocation block** — with the other reviewer invocation blocks (e.g. before CodeRabbit's), add:

````markdown
**Grok (SpaceXAI Grok Build):**
```bash
if [ -n "$GROK_MODEL" ] && [ "$GROK_MODEL" != "null" ]; then
  grok --prompt-file /tmp/gsd-review-prompt-{phase}.md -m "$GROK_MODEL" --yolo --tools "read_file,grep,list_dir" --output-format plain --no-auto-update 2>/tmp/gsd-review-grok-{phase}.err > /tmp/gsd-review-grok-{phase}.md
else
  grok --prompt-file /tmp/gsd-review-prompt-{phase}.md --yolo --tools "read_file,grep,list_dir" --output-format plain --no-auto-update 2>/tmp/gsd-review-grok-{phase}.err > /tmp/gsd-review-grok-{phase}.md
fi
if [ ! -s /tmp/gsd-review-grok-{phase}.md ]; then
  {
    echo "Grok review failed or returned empty output."
    echo ""
    echo "stderr (last 20 lines):"
    tail -20 /tmp/gsd-review-grok-{phase}.err 2>/dev/null
    echo "If grok is not installed: curl -fsSL https://x.ai/cli/install.sh | bash"
    echo "If not authenticated: run \`grok\` once interactively to log in."
  } > /tmp/gsd-review-grok-{phase}.md
fi
```
````

The `--tools "read_file,grep,list_dir"` allowlist keeps the review strictly read-only — no shell, no edits.

Stderr goes to a sidecar `.err` log because grok logs diagnostics (including install/auth failures) to stderr — with `2>/dev/null` they vanish silently.

**6. REVIEWS.md template** — add `grok` to the reviewers list in the output frontmatter:

```yaml
reviewers: [gemini, claude, codex, grok, coderabbit, opencode, qwen, cursor, ollama, lm_studio, llama_cpp]  # populate at runtime with only the reviewers actually invoked
```

and add a section to the template body (after the Codex section):

```markdown
## Grok Review

{grok review content}
```

## Patch 7: `workflows/plan-review-convergence.md`

`/gsd:plan-review-convergence` has its own flag parser. Two edits:

**Flag documentation** — add `--grok` to the documented reviewer flags:

```markdown
Extract from $ARGUMENTS: phase number, reviewer flags (`--codex`, `--grok`, `--gemini`, `--claude`, `--opencode`, `--ollama`, `--lm-studio`, `--llama-cpp`, `--all`), `--max-cycles N`, `--text`, `--ws`.
```

**Flag parsing** — after the `--codex` grep line, add:

```bash
echo "$ARGUMENTS" | grep -q '\-\-grok' && REVIEWER_FLAGS="$REVIEWER_FLAGS --grok"
```

## Verify

```
/gsd:review --grok <phase>
```

should detect `grok:available`, run the read-only review, and produce a `## Grok Review` section in the phase's REVIEWS.md. To pin a model: set `review.models.grok` in your GSD config.
