# Worktree cleanup + shared build caches

Date: 2026-11-05. Skills: `carve` (ship), `conjure`.

## Goals

1. After `ship` merges, remove the local worktree and local branch.
2. New worktrees reuse build caches (node_modules, target) from the main checkout.

Non-goals: worktrunk, sccache setup, pnpm/yarn install, any commit or push by the skills beyond existing `ship`.
Constraints: only git, native `EnterWorktree`/`ExitWorktree`, standard tooling. macOS/APFS.

## Research evidence

Environment: git 2.54.0 (Apple), macOS APFS. Scratch repos with a local bare origin.

### Cleanup

| Test | Result |
|------|--------|
| `ExitWorktree` contract | Acts only on worktrees `EnterWorktree` created in the same session, else no-op. `remove` refuses dirty/unmerged unless `discard_changes: true` (a force). `path`-entered worktrees are never removed. |
| `git worktree remove` + untracked file | `contains modified or untracked files, use --force to delete it` |
| `git worktree remove` run with cwd inside worktree | Succeeds (rc=0); next command: `Unable to read current working directory`. Must leave the dir first. |
| `git branch -d` while checked out in worktree | `cannot delete branch ... used by worktree` |
| `git branch -d`, stale local base, upstream ref pruned | `not fully merged` (compares against checkout HEAD) |
| `git branch -d`, main checkout on another branch | `not fully merged` |
| `git branch -d` after `merge --ff-only origin/<base>` | `Deleted branch` |
| `git fetch origin <base>:<base>` with base checked out | `refusing to fetch into branch ... checked out` |

Decision: `git worktree remove` + `git branch -d`, never force. `ExitWorktree` only as `action: "keep"` to move the session out when it entered the worktree. Base is fast-forwarded (`merge --ff-only origin/<base>`) in the main checkout first; main checkout must be on base and clean, else stop.

### Caches (macOS)

| Ecosystem | Measured | Decision |
|-----------|----------|----------|
| npm | `cp -c -R node_modules` 0.25s; follow-up `npm install` 1.4s vs 1.2s cold (warm npm cache, 8.7M tree). Changed dep in clone left main untouched. `npm install` replaced a symlinked `node_modules` with a real dir. | Clone. Symlink rejected. |
| Rust | Cold build 4.1s. After `cp -c -R target`: 0.12s, only local crate rebuilt. | Clone. |
| Rust shared `CARGO_TARGET_DIR` | Builds OK in test; lock contention not tested; output binary path shared. | Rejected. |
| bun | Warm global cache install 0.035s (cold 76s, cause not investigated). | Nothing. |
| uv | Fresh `uv sync` in worktree 0.06s. | Nothing. |
| Go | `GOMODCACHE`, `GOCACHE` global. | Nothing. |
| pnpm, sccache | Not installed, not tested. | pnpm: nothing (global store, per docs). sccache: not used. |

Not measured: disk delta (`du`/`df`) for clones. APFS clones are copy-on-write.

## Design

### conjure

New `### Warm caches` subsection in Step 1, after the worktree exists and before `using-git-worktrees` setup and baseline tests. Detect by lockfile/manifest. Never symlink, never set shared `CARGO_TARGET_DIR`. Skip when destination exists or source missing. `cp` failure: remove the partial copy, report, continue. Step 2 and Rules unchanged.

### carve

Step 6 sub-step 7 "Clean up local" (after remote branch delete), old Report is sub-step 8. Runs only if steps 1-6 succeeded.

Guards (any fail: stop, remove nothing): PR state `MERGED`; branch is the pushed feature branch and not base; `git status --porcelain` empty; `origin/<base>` fetched and contains branch; for a linked worktree, main checkout on base and clean.

Linked worktree: `ExitWorktree keep` if session-entered, `merge --ff-only`, `cd "$main_wt" && git worktree remove`, `git -C "$main_wt" branch -d`.
Main checkout: `git switch <base>`, `merge --ff-only`, `branch -d`.
No `--force`, no `git branch -D`, no `discard_changes`. `?` help and `ship` authorization line mention local worktree + branch removal.
