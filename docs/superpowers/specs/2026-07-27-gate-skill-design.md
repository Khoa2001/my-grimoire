# `/gate` Skill — Design Spec

## Problem

No repeatable way to validate a working tree before committing: run whatever
tests/lint/typecheck the project defines, review the diff for bugs, apply
safe fixes automatically, and surface judgment calls to the user — all in one
pass, on demand. Inspired by `kunchenguid/no-mistakes` (a standalone git-proxy
CLI with a similar pipeline), scoped down to a plain markdown Claude Code
skill: no daemon, no git remote, no worktree isolation, no PR automation.

## Scope

New skill: `skills/gate/SKILL.md` (trigger `/gate`) +
`skills/gate/references/check-detection.md`.

Diff scope is **working tree vs last commit** (uncommitted changes only —
staged + unstaged), matching `carve`'s read step. Not branch-vs-main; not a
pre-push hook.

## Architecture

Single skill, no daemon/proxy. On `/gate`:

```
read diff → detect+run project checks → review diff → classify findings
    → apply auto-fixes → ask user on ask-user findings → re-run checks
    → repeat until clean or max passes → offer /carve handoff
```

Check-detection heuristics (how to find and run a project's test/lint/
typecheck commands across ecosystems) live in `references/check-detection.md`,
loaded only when Step 2 runs — keeps the main SKILL.md flow scannable, same
split `brainstorming` uses for `visual-companion.md`.

## Components

1. **Diff reader** — `git status --short` + `git diff HEAD`. Both empty →
   "Nothing to gate. Working tree is clean." Stop.
   Merge conflicts (`UU`/`AA`/`DD`) → warn and exclude those files, same as
   `carve`.

2. **Check detector** (`references/check-detection.md`) — looks for
   `package.json` scripts (`test`, `lint`, `typecheck`), `Makefile` targets,
   `pytest.ini`/`pyproject.toml`, `go test ./...`, `cargo test`/`clippy`, etc.
   Runs whatever exists. A check with no detectable command is skipped
   silently (not reported as a failure).

3. **Code reviewer** — inline analysis of the diff for correctness/quality
   bugs, same bar as the built-in `/code-review`. Runs in the main thread
   (not delegated to a subagent) so the fix loop below stays conversational.

4. **Finding classifier** — every finding (from checks or review) is bucketed:
   - **auto-fix**: mechanical (formatting, lint-autofixable, import order,
     trivial typos) OR an obvious one-fix bug (off-by-one, wrong operator,
     missing null check) — anything with one unambiguous correct fix.
   - **ask-user**: anything touching intent, ambiguous fixes, failing test
     whose correct behavior isn't obvious from context.

5. **Fix loop**:
   - Apply all auto-fix findings immediately.
   - Re-run only the checks affected by files just touched.
   - Display remaining ask-user findings, numbered, and wait for action.
   - Repeat until: no findings remain, or all remaining ask-user findings are
     resolved (approved/fixed/skipped), or the max-pass guard trips.

6. **Handoff** — once clean: "Working tree clean. Run /carve to split into
   commits?" User confirms or declines; declining just ends the skill.

## Finding Display Format

Grouped by source, most recent pass only (not cumulative history):

```
── Gate: 2 checks run · 5 findings ─────────────────────
Tests (npm test)
  ✓ fixed: src/utils/parse.ts:42 — off-by-one in slice bound

Lint (eslint)
  ✓ fixed: src/App.tsx:10 — unused import

Review
  [1] src/auth/session.ts:88 — token expiry uses `<` should be `<=`
  [2] src/api/client.ts:15 — retry loop has no backoff, could hot-loop
──────────────────────────────────────────────────────────
2 auto-fixed · 2 need your call  (? for help)
```

`?` shows:

```
Actions:
  approve 1 2      — apply finding as proposed
  fix 2: <note>    — apply with your instruction instead
  skip 2           — leave finding unresolved, exclude from re-checks
  stop             — stop, leave remaining findings as-is
```

## Error Handling

- **Check command fails to run** (tool missing, e.g. no `npm` installed) —
  skip that check, note it once in the summary footer, don't block the loop.
- **Auto-fix breaks something that passed before** — do not silently
  re-attempt; surface as a new ask-user finding on the next pass.
- **Max-pass guard** — 5 fix-loop passes. If findings remain after 5, stop
  and dump the remaining list for manual handling: "Stopped after 5 passes.
  N findings remain — resolve manually or re-run /gate."
- **`git` command fails** (e.g. `git diff` errors) — show exact error, do not
  retry, ask user to resolve, same as `carve`'s edge case handling.

## Testing

No automated tests — this is a markdown skill file with no build system
(same as `carve`, `impeccable`). Verified by reading the finished SKILL.md
and mentally tracing a sample run: dirty tree with one lint issue, one
obvious bug, and one ambiguous review finding.
