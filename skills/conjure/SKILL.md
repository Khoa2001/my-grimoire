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
- Worktree exists (just created, or already in one): warm caches (below) before that skill's project setup and baseline tests.
- Setup or baseline tests fail: report and ask, as that skill says. Do not continue to Step 2 until the user decides.

### Warm caches

Goal: new worktree reuses build output from the main checkout instead of reinstalling or rebuilding.

```bash
main_wt=$(git worktree list --porcelain | sed -n '1s/^worktree //p')
wt=$(git rev-parse --show-toplevel)
```

`main_wt` equals `wt` (not a linked worktree): skip. Detect ecosystems by lockfile or manifest at `$wt` root. Each row is independent:

| Detected | Action | Why |
|----------|--------|-----|
| `package-lock.json` or `yarn.lock` | `cp -c -R "$main_wt/node_modules" "$wt/node_modules"` | APFS clone: near-instant, no extra disk until files change. Own copy per branch. The install in setup reconciles any lockfile difference. |
| `Cargo.toml` | `cp -c -R "$main_wt/target" "$wt/target"` | Dependencies stay compiled, only the local crates rebuild. Own copy per branch: no lock contention, no cross-branch overwrites. |
| `pnpm-lock.yaml`, `bun.lock`, `bun.lockb`, `uv.lock`, `go.mod` | nothing | Their package store or cache is already global and shared. |

Rules:
- Skip a row when the destination already exists in the worktree, or the source is missing in the main checkout. No error.
- Never symlink a cache dir into the main checkout, never set a shared `CARGO_TARGET_DIR`: branches would overwrite each other's output.
- `cp -c` fails: remove the partial destination it created, report one line, continue. Cache is an optimization, not a gate.
- Report one line per warmed dir, e.g. `Warmed: node_modules (cloned from main checkout)`.

## Step 2 — Brainstorm

Once the workspace is ready, invoke Skill `superpowers:brainstorming`, passing along anything the user typed after `/conjure` as the feature description.

Run it exactly as written, including its approval gate and the hand-off to `superpowers:writing-plans`.

## Rules

- Worktree first, brainstorm second: any design doc lands on the feature branch, never on main.
- No commits or pushes. User says when.
