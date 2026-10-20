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
- Never include in the subject/summary line: "this commit", "I", "we", emoji (unless project convention), or AI self-reference.
- Append a `Co-Authored-By:` trailer as its own paragraph after the body (blank line separator), using whatever convention the current agent/session normally uses for commits (name + noreply email) — same as it would for any other commit.

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
      ship             — commit all groups, push, open PR, merge to default branch, delete remote branch
      commit 1 2       — commit specific groups (space or comma separated)
      skip 2           — exclude group from plan
      edit 2: message  — change a group's commit message
      stop             — stop, leave everything dirty
    ```

  - Re-display the current plan immediately after.

**`commit all`** (or `all` / `yes` / `y`):
  - For each group in number order:
    1. `git add -- <file1> <file2> ...` — quote each path
    2. Commit via heredoc so the body and trailer survive intact:
       ```bash
       git commit -m "$(cat <<'EOF'
       <type>(<scope>): <summary>

       <optional body>

       Co-Authored-By: <Agent Name> <noreply-email>
       EOF
       )"
       ```
    3. Report: `✓ [N] <commit message>`
  - End: "All N groups committed. Working tree is clean."
  - Then offer: "Ship it? (`ship` = push, PR, merge to <default>, delete remote branch)". Run Step 6 only if user answers `ship` / `yes` / `y`. Anything else: exit.

**`ship`**:
  - Commit all groups (same flow as `commit all`), then run Step 6.
  - Typing `ship` is the explicit go-ahead for commit, push, PR, merge and remote branch delete. Do not ask again, except for the guards in Step 6.

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
  - If the new message has no `Co-Authored-By:` line, append one automatically before committing. If it already has one, don't duplicate.
  - Re-display updated plan so user reviews before committing

**`stop`** (or `done` / `quit` / `enough`):
  - Respond: "Carving stopped. N files remain uncommitted."
  - Exit

## Step 6 — Ship (only after `ship` or a "yes" to the post-commit offer)

Requires `gh` authenticated. Run in order; stop at first failure and show the exact error. Never retry or force.

1. **Guards**
   - `base=$(gh repo view --json defaultBranchRef -q .defaultBranch)`; `branch=$(git branch --show-current)`.
   - `branch` empty (detached HEAD) or equal to `base`: stop. "On <base>, cannot open a PR from it. Create a feature branch first." Commits stay local.
   - Uncommitted files remain (skipped groups): warn they are left out of the PR, continue.
   - `git status --short` shows conflicts: stop.
2. **Remote name**: strip any `worktree-` prefix to get `remote_branch`. Else `remote_branch=$branch`.
3. **Push**: `git push -u origin "$branch:$remote_branch"`
4. **Create PR**: `gh pr create --base "$base" --head "$remote_branch" --title "<title>" --body "<body>"`
   - Title: the single commit's subject if one group, else a summary subject in the same `type(scope): summary` format.
   - Body: one bullet per commit subject, then the agent's PR attribution line if the session uses one.
   - PR already exists for the branch: reuse it (`gh pr view --json url`).
5. **Merge**: `gh pr merge "$remote_branch" --merge`
   - Merge-commit method, matching repo history. No `--admin`, no `--auto`. Blocked (checks, reviews, conflicts): stop, print PR URL, leave branch intact.
6. **Delete remote branch**: `git push origin --delete "$remote_branch"`
   - "remote ref does not exist" means repo auto-deleted it: fine, continue.
   - Local branch is left untouched.
7. **Report**: `✓ Shipped: <PR url> merged into <base>, origin/<remote_branch> deleted.`

## Edge Cases

- **git commit fails**: show exact error. Do not retry. Ask user to resolve.
- **Ship step fails after commits**: commits stay local. Report which sub-step failed and what already happened (pushed? PR open?) so the user can resume by hand.
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
