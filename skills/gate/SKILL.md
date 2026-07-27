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
