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
| [carve](skills/carve/SKILL.md) | `/carve` | Carve a dirty working tree into clean, atomic, feature-separated commits. Shows all groups at once for parallel review. `ship` also pushes, opens a PR, merges it, deletes the remote branch, then removes the local worktree and branch (only if the PR is merged and the worktree is clean; never forced). |
| [conjure](skills/conjure/SKILL.md) | `/conjure` | Start a feature in one command: isolated worktree (`superpowers:using-git-worktrees`), then `superpowers:brainstorming`. New worktrees clone `node_modules` / `target` from the main checkout (`cp -c`, APFS) so installs and builds are not repeated. |
| [plane-quest](skills/plane-quest/SKILL.md) | `/plane-quest` | Take a Plane ticket from check to In Review: read-only Plane check, set In Progress when gaps are found (only from Backlog, Grooming, Ready or Todo), conjure for gaps, build, emulator test vs prototype, carve ship, then set state and post a short Vietnamese comment with one screenshot per done UI item (added through Chrome). Looks up Drive, Plane, prototype before asking; asks only when all are silent. Never sets Done or Released unless told. |
| [bug-quest](skills/bug-quest/SKILL.md) | `/bug-quest` | Hunt bugs in a feature, an app or all apps (user flow on the simulator, rules from PRD/SRS/Plane, prototype pixels and motion, code and test gaps), show a list with pictures, file the picked ones as GitHub issues with `status:*` labels (pictures added through Chrome). `fix` takes an issue number: reproduce, root cause, failing test, fix, comment, then with `ship` a PR with `Fixes #N` (never merged). Repo facts come from the repo guide (`docs/guides/hunt-bugs.md`). |
