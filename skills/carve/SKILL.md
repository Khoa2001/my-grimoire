---
name: carve
description: Carve a dirty working tree into clean, feature-separated commits. Analyzes ALL remaining changes at once, proposes all groups together, lets user commit/skip/edit multiple groups simultaneously. Use when user types /carve or asks to split commits by feature.
trigger: /carve
---

# /carve

Carve a dirty working tree into clean, atomic, feature-separated commits — showing all groups at once for parallel review.

## Step 1 — Read all changes

Run both commands and read their full output:

```bash
git status --short
git diff HEAD
```

If both return empty output: respond "Nothing to carve. Working tree is clean." and stop.

**Merge conflicts detected** (`git status` shows `UU`/`AA`/`DD`): warn "Conflicts detected in <files> — excluding them. Resolve conflicts first." Exclude conflicted files from all groups.

## Step 2 — Identify ALL groups at once

Analyze the full diff. Partition **all** changed/untracked files into cohesive groups — files that belong together semantically (same concern, same module, same bug fix, same feature).

Rules:
- Each file belongs to exactly one group
- Prefer smaller, tighter groups over large vague ones
- Untracked files (`??`) are valid candidates — include them if they belong
- If everything is one concern, one group is fine

## Step 3 — Generate commit message for every group

For each group, write a commit message:
- Format: `<type>(<scope>): <imperative summary>`
- Types: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `chore`, `build`, `ci`, `style`, `revert`
- Subject ≤50 chars, hard cap 72. No trailing period.
- Imperative mood: "add", "fix", "remove" — not "added", "fixes"
- Body only when the *why* is non-obvious. Wrap at 72 chars.
- Never include: "this commit", "I", "we", emoji (unless project convention), AI attribution

## Step 4 — Display ALL groups together

Rules:
- No `Reason:` line — the commit message already conveys intent.
- Files: show the **last 2 path segments** of each file path, space-separated on one line. If a group has more than 3 files, wrap to a second indented line.
- Footer: `N groups · M files  (? for help)` — no `Total:` prefix.
- Do NOT show the actions menu here. It appears only when the user types `?`.

Show this exact format:

```
── Carve plan ──────────────────────────────────────────
[1] feat(auth): add login flow
    views/login_view.dart  bloc/auth_bloc.dart

[2] fix(ui): correct button padding
    widgets/app_button.dart

[3] chore: update dependencies
    pubspec.yaml  pubspec.lock
────────────────────────────────────────────────────────
3 groups · 6 files  (? for help)
```

## Step 5 — Process user action

**`?`** (or `help`):
  - Print the full actions reference:

    ```
    Actions:
      commit all       — commit all groups in order
      commit 1 2       — commit specific groups (space or comma separated)
      skip 2           — exclude group from plan
      edit 2: message  — change a group's commit message
      stop             — stop, leave everything dirty
    ```

  - Re-display the current plan immediately after.

**`commit all`** (or `all` / `yes` / `y`):
  - For each group in number order:
    1. `git add -- <file1> <file2> ...` — quote each path
    2. `git commit -m "<message>"` (heredoc for multi-line)
    3. Report: `✓ [N] <commit message>`
  - End: "All N groups committed. Working tree is clean."

**`commit 1 3`** (space or comma separated numbers):
  - Commit only the named groups in number order (same add → commit flow above)
  - If groups remain uncommitted: re-display remaining plan and wait for next action

**`skip 2`** (or `skip 1 3` for multiple):
  - Remove those groups from the plan
  - Add their files to a session-level exclusion list (never re-propose this session)
  - Re-display updated plan with renumbered groups

**`edit 2: new message`**:
  - Replace that group's commit message with the provided text
  - If message is empty: re-prompt "Message is empty — provide a message or say skip."
  - Re-display updated plan so user reviews before committing

**`stop`** (or `done` / `quit` / `enough`):
  - Respond: "Carving stopped. N files remain uncommitted."
  - Exit

## Edge Cases

- **git commit fails**: show exact error. Do not retry. Ask user to resolve.
- **All groups skipped**: "All groups skipped. N files remain uncommitted." Stop.
- **Single group remaining after partial commit**: display it normally, same actions available.

## Commit Message Examples

Good:
```
fix(auth): correct token expiry boundary check
```
```
feat(ui): add dark mode toggle to settings panel

User preference stored in localStorage, no backend changes needed.
```

Bad:
- "Updated some files" — no type, no scope, vague
- "feat: this commit adds the new user profile endpoint and also fixes the auth bug and updates the readme" — too many concerns, too long

## Context

This is a Claude Code skill file at `~/.claude/skills/carve/SKILL.md`. No code to compile, no tests to run.
