# CLAUDE.md rules for cross-AI delegation

Add this section to your global `~/.claude/CLAUDE.md` (or a project CLAUDE.md) to make Grok delegation a standing habit rather than something you have to remember to invoke. Adjust the trigger thresholds to taste.

```markdown
# Grok CLI Delegation

The SpaceXAI Grok CLI is a standing delegation target in every project, via the
`grok-worker` agent and the grok delegate skill (drives `grok` headless:
`--prompt-file`, `--yolo`, `--output-format plain`; read-only reviews add
`--sandbox strict` (kernel-enforced read confinement to the CWD plus
essential system paths) plus `--tools "read_file,grep,list_dir"` plus
`--disallowed-tools
"run_terminal_cmd,search_replace,web_search,search_tool,use_tool"` plus
`--deny 'MCPTool(*)'` — the `search_tool,use_tool` disallow and the
`--deny 'MCPTool(*)'` backstop, which survives `--yolo`, are what block
MCP meta-tools; write-capable `--yolo` runs add `--sandbox workspace`
(write confinement to the CWD, `~/.grok/`, and temp dirs) instead. Best
for independent second opinions from a non-Claude model family and
parallel delegated implementation.

Natural triggers (apply unless the user opts out):
- Before merging/shipping nontrivial work → get a cross-AI review
  (delegate with --read-only).
- Stuck after ~2 failed fix attempts → hand a diagnosis pass to
  `grok-worker` instead of a third solo attempt.
- High-stakes designs/plans → get an independent external review;
  reconcile disagreements explicitly, attributing whose opinion is whose.

The courier subagent is a thin (sonnet) forwarder; Grok does the reasoning.
The grok CLI requires one-time interactive auth (run `grok` once).
```

If you also use another vendor's delegation plugin (e.g. OpenAI's codex plugin), broaden the section to cover both and add a dual-review trigger for high-stakes work — a three-family panel (Claude + GPT + Grok) catches more than any pair.
