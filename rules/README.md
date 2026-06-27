# Rules

Project-level and user-global rules can override or extend kamae-model-port
and kamae-model-bridge behavior.

## Rule format

Each rule is a Markdown file with YAML frontmatter:

```yaml
---
name: my-rule
applies-to: kamae-model-port  # or kamae-model-bridge, or *
kind: convention               # convention | library-preference | override
---

Rule body here.
```

## Priority

1. `.claude/rules/*.md` / `.codex/rules/*.md` in the project root (highest)
2. `~/.claude/rules/*.md` / `~/.codex/rules/*.md` (user-global)
3. `rules/defaults/*.md` in this repository (lowest)

Within a tier, the lexicographically last filename wins for the same `name`.

## Defaults

Place default rules in `rules/defaults/`. These ship with the skill and are
overridden by project or user rules with the same `name`.
