---
name: gate
description: Validate the working tree before commit — run the project's own test/lint checks, review the diff for bugs, auto-fix safe findings, ask the user about judgment calls, loop until clean. Use when user types /gate or asks to validate/review changes before committing.
trigger: /gate
---

# /gate

Validate a dirty working tree: run the project's own checks, review the diff, auto-fix what's safe, ask about the rest — looping until clean.

## Step 1 — Read all changes

Run both commands and read their full output:

```bash
git status --short
git diff HEAD
```

If both return empty output: respond "Nothing to gate. Working tree is clean." and stop.

**Merge conflicts detected** (`git status` shows `UU`/`AA`/`DD`): warn "Conflicts detected in <files> — excluding them. Resolve conflicts first." Exclude conflicted files from all checks and review.

## Step 2 — Detect and run project checks

Read `skills/gate/references/check-detection.md` for the detection
heuristics, then apply them to the current repo.

Run every check the heuristics detect. Only run checks relevant to files
that changed in the diff from Step 1 when the check tool supports scoping
(e.g. `eslint <changed files>`); otherwise run the project-wide command.

Collect results as a list of `{check_name, command, exit_code, output}`.
If a check command itself fails to execute (tool not installed, e.g. `npm`
missing) — note it once for the final summary footer, don't block the
remaining flow, and don't retry it.

## Step 3 — Review the diff

Read the full diff from Step 1 and review it for correctness and quality
bugs — same bar as the built-in `/code-review`: logic errors, edge cases,
security issues, resource leaks, off-by-ones, wrong operators, missing
null/bounds checks. Do this inline in the main conversation — do not
delegate to a subagent, since the fix loop in Step 5 needs to stay
conversational with the user.

Do not flag style/formatting nits already caught by a lint check from
Step 2.

For each issue found, record `{source: "review", file, line, problem,
fix_description}`.

## Step 4 — Classify findings

Combine every finding from Step 2 (failing checks, lint violations) and
Step 3 (review issues) into one list. Classify each as:

- **`auto-fix`** — mechanical fixes (formatting, lint-autofixable issues,
  import order, trivial typos) OR a bug with exactly one unambiguous
  correct fix (off-by-one, wrong comparison operator, missing null check
  where the correct check is obvious from surrounding code).
- **`ask-user`** — anything touching intent, any fix with more than one
  reasonable approach, or a failing test whose correct behavior isn't
  obvious from context alone.

When in doubt between the two, classify as `ask-user` — never guess on
something that changes behavior ambiguously.

## Step 5 — Display findings

Group findings by source (`Tests`, `Lint`, `Review`, etc — use the check's
own name for check-derived groups). Within each group, `auto-fix` findings
are already-applied by the time this displays (see Step 6), shown as
`✓ fixed: ...`; `ask-user` findings are numbered sequentially across the
whole list (not per-group) so `approve 1 3` can span groups.

Show this exact format:

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

If there are zero `ask-user` findings after auto-fixes are applied, skip
straight to Step 7 (clean) instead of displaying an empty action prompt.

## Step 6 — Fix loop

On each pass:

1. Apply every `auto-fix` finding immediately (edit the file directly).
2. Re-run only the checks whose files were just touched (from Step 2's
   check list) — not the full check suite, unless no check scoping is
   possible for that tool.
3. Display the result per Step 5.
4. If there are `ask-user` findings remaining, wait for user action:

**`?`** (or `help`): print the actions reference, then re-display findings.

```
Actions:
  approve 1 2      — apply finding as proposed
  fix 2: <note>    — apply with your instruction instead
  skip 2           — leave finding unresolved, exclude from re-checks
  stop             — stop, leave remaining findings as-is
```

**`approve 1 2`** (space or comma separated): apply each named finding's
proposed fix exactly as described, then start a new pass (back to Step 6
Step 1) — re-running affected checks and re-review only if new code was
touched.

**`fix 2: <note>`**: apply the finding's fix per the user's instruction
instead of the original proposal, then start a new pass.

**`skip 2`** (or `skip 1 3` for multiple): remove those findings from the
active list permanently for this `/gate` run (do not re-propose them this
session). Re-display remaining findings.

**`stop`** (or `done` / `quit`): respond "Gate stopped. N findings remain
unresolved." and exit — do not proceed to Step 7.

### Error handling

- **Auto-fix breaks something that passed before**: do not silently
  re-attempt the same fix. Surface it as a new `ask-user` finding on the
  next pass's display.
- **Max-pass guard**: after 5 passes through this loop, stop regardless of
  remaining findings: "Stopped after 5 passes. N findings remain — resolve
  manually or re-run /gate." Do not proceed to Step 7.
- **A `git` command fails** (e.g. `git diff` errors mid-loop): show the
  exact error, do not retry, ask the user to resolve it, then stop.

## Step 7 — Clean handoff

Reached when every finding is resolved (auto-fixed, approved-and-fixed, or
skipped) and no `ask-user` findings remain.

Respond: "Working tree clean — N findings resolved this run. Run /carve to
split into commits?"

- If the user confirms (`yes` / `y` / `/carve`): invoke the `carve` skill.
- If the user declines: respond "Done." and stop.

## Edge Cases

- **All findings skipped**: Step 7's "N findings resolved" should read "0
  findings resolved, N skipped" instead — still offer the `/carve` handoff
  since the tree itself isn't blocked, just unvalidated on those points.
- **No checks detected at all** (no test/lint tooling in the repo): Step 2
  produces zero check results; the run proceeds with review-only findings.
- **Single ask-user finding remaining after partial approval**: display it
  normally per Step 5, same actions available.

## Context

This is a Claude Code skill file at `~/.claude/skills/gate/SKILL.md`, with
detection heuristics in `~/.claude/skills/gate/references/check-detection.md`.
No code to compile, no tests to run for the skill itself.
