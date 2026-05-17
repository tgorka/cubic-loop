---
"cubic-loop": minor
---

Ship cubic-loop as a Claude Code plugin. Adds `.claude-plugin/plugin.json`
(plugin name `cubic`, version `0.1.0`, skills path `./skills/`) and
`.claude-plugin/marketplace.json` (single-plugin marketplace named
`cubic-loop`) modelled on `tgorka/bmad-stepper`. Users can now install
via:

```
/plugin marketplace add tgorka/cubic-loop
/plugin install cubic@cubic-loop
```

The skill becomes available as `cubic:cubic-loop`. Hand-copying
`skills/cubic-loop/` to `~/.claude/skills/` still works as a fallback.
`cubic.yaml` excludes `.claude-plugin/**` from auto-stamp so plugin
metadata changes always get human review. README documents the install
flow and updates the invocation examples to the namespaced form.
