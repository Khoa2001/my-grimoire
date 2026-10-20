# my-grimoire

Personal collection of Claude Code skills — version-controlled so they're auditable, portable, and shareable.

## What are skills?

Skills are markdown files that extend Claude Code's behavior. Drop a skill into `~/.claude/skills/<name>/SKILL.md` and invoke it with `/<name>` in any Claude Code session.

## Install a skill

```bash
# Copy a skill
cp -r skills/carve ~/.claude/skills/carve

# Or symlink (keeps in sync with this repo)
ln -s $(pwd)/skills/carve ~/.claude/skills/carve
```

## Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| [carve](skills/carve/SKILL.md) | `/carve` | Carve a dirty working tree into clean, atomic, feature-separated commits. Shows all groups at once for parallel review. `ship` also pushes, opens a PR, merges it and deletes the remote branch. |
| [conjure](skills/conjure/SKILL.md) | `/conjure` | Start a feature in one command: isolated worktree (`superpowers:using-git-worktrees`), then `superpowers:brainstorming`. |
