---
name: conjure
description: Start a new feature in one command. Sets up an isolated git worktree, then runs the brainstorming flow. Use when user types /conjure or wants to begin a new feature with isolation plus design discussion.
trigger: /conjure
---

# /conjure

Begin a new feature: isolated workspace first, then brainstorm. Replaces typing `/superpowers:using-git-worktrees` and `/superpowers:brainstorming` back to back.

Both steps are invoked by name, so updates to the superpowers plugin flow through. Do not copy their content here.

## Step 1 — Isolate

Invoke Skill `superpowers:using-git-worktrees`.

- User always wants isolation: do not ask its Step 0 consent question.
- Use the native `EnterWorktree` tool, not manual `git worktree add`.
- Already in a linked worktree: skip creation, report the path and branch.
- Branch name: clean, no `worktree-` prefix, so the remote name stays clean.
- Setup or baseline tests fail: report and ask, as that skill says. Do not continue to Step 2 until the user decides.

## Step 2 — Brainstorm

Once the workspace is ready, invoke Skill `superpowers:brainstorming`, passing along anything the user typed after `/conjure` as the feature description.

Run it exactly as written, including its approval gate and the hand-off to `superpowers:writing-plans`.

## Rules

- Worktree first, brainstorm second: any design doc lands on the feature branch, never on main.
- No commits or pushes. User says when.
