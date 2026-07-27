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
