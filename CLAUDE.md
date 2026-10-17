# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Personal grimoire — version-controlled collection of Claude Code skills. Skills are markdown files at `skills/<name>/SKILL.md` that extend Claude Code behavior. Installed to `~/.claude/skills/<name>/SKILL.md` and invoked via `/<name>`.

No build system, no tests, no lint. All artifacts are markdown.

## Installing a skill

```bash
# Copy (one-time snapshot)
cp -r skills/carve ~/.claude/skills/carve

# Symlink (keeps in sync with repo — preferred)
ln -s $(pwd)/skills/carve ~/.claude/skills/carve
```

## Repo structure

- `skills/<name>/SKILL.md` — skill definitions (the deliverables)
- `docs/superpowers/specs/` — design specs for skill changes
- `docs/superpowers/plans/` — implementation plans for skill changes

## Skill file format

Every skill has YAML frontmatter:

```markdown
---
name: <slug>
description: <one-line description used by Claude for skill matching>
trigger: /<name>
---
```

The body is instruction prose Claude Code follows when the skill is invoked.

## Working on a skill

1. Edit `skills/<name>/SKILL.md` directly — this is the source of truth
2. If symlinked, changes are live immediately in any Claude Code session
3. Verify by reading the updated file and mentally tracing through a sample invocation
4. Commit with `feat(name):` or `fix(name):` scope

## Skills in this repo

| Skill | Trigger | Purpose |
|-------|---------|---------|
| carve | `/carve` | Carve dirty working tree into atomic, feature-separated commits. Analyzes all changes at once, proposes groups, lets user commit/skip/edit. |
