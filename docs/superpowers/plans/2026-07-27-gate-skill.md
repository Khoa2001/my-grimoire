# `/gate` Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a new `/gate` skill that validates the working tree on demand — runs the project's own test/lint checks, reviews the diff for bugs, auto-fixes safe findings, asks the user about judgment calls, loops until clean, then offers to hand off to `/carve`.

**Architecture:** New skill directory `skills/gate/` with `SKILL.md` (the flow) and `references/check-detection.md` (project test/lint command detection heuristics, loaded only at Step 2). No code, no automated tests — this is a markdown skill file interpreted by Claude Code. Verification is done by reading the finished file and mentally tracing sample runs, same as `carve`.

**Tech Stack:** Markdown skill file + one reference doc, following the `skills/carve/SKILL.md` and `skills/brainstorming` (main skill + reference file) patterns already in this repo/plugin cache.

## Global Constraints

- Diff scope: working tree vs last commit (uncommitted changes only — staged + unstaged). Never branch-vs-main.
- Trigger: `/gate`.
- Merge conflicts (`UU`/`AA`/`DD`): exclude conflicted files, warn, same as `carve`.
- No git-proxy, no worktree isolation, no PR automation — those are explicitly out of scope (that's what distinguishes this from `no-mistakes`).
- Max fix-loop passes: 5.
- Spec of record: `docs/superpowers/specs/2026-07-27-gate-skill-design.md`.

---

### Task 1: Scaffold SKILL.md + diff reader (Step 1)

**Files:**
- Create: `skills/gate/SKILL.md`

**Interfaces:**
- Produces: `## Step 1 — Read all changes` section other tasks append after.

- [ ] **Step 1: Write frontmatter and header**

  Create `skills/gate/SKILL.md` with:

  ```markdown
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
  ```

- [ ] **Step 2: Verify the file**

  Read `skills/gate/SKILL.md` back. Confirm frontmatter has `name: gate`, `trigger: /gate`, and Step 1 matches `carve`'s Step 1 pattern (same two commands, same empty-tree and conflict handling).

- [ ] **Step 3: Commit**

  ```bash
  git add skills/gate/SKILL.md
  git commit -m "feat(gate): scaffold skill with diff reader step"
  ```

---

### Task 2: Check detector (Step 2) + reference file

**Files:**
- Create: `skills/gate/references/check-detection.md`
- Modify: `skills/gate/SKILL.md` (append after Step 1)

**Interfaces:**
- Produces: `## Step 2 — Detect and run project checks` section; a set of "check results" (per-check name, pass/fail, raw output) that Task 3's review step and Task 4's classifier consume.

- [ ] **Step 1: Write the reference file**

  Create `skills/gate/references/check-detection.md`:

  ```markdown
  # Check Detection Heuristics

  Look for these signals in the repo root (and nearest ancestor if run from a
  subdirectory) and run whatever is found. Skip anything not found — do not
  report a missing tool as a failure.

  ## Node / JavaScript / TypeScript

  If `package.json` exists, read its `scripts` block:
  - `test` script present → run `npm test` (or `yarn test` / `pnpm test` if
    a `yarn.lock` / `pnpm-lock.yaml` is present instead of `package-lock.json`)
  - `lint` script present → run the equivalent lint command
  - `typecheck` or `tsc` script present → run it; otherwise if `tsconfig.json`
    exists, run `npx tsc --noEmit`

  ## Python

  - `pyproject.toml` or `pytest.ini` or a `tests/` directory with `test_*.py`
    files → run `pytest`
  - `pyproject.toml` with a `[tool.ruff]` section, or a `.ruff.toml` /
    `ruff.toml` file → run `ruff check .`
  - `mypy.ini` or `pyproject.toml` with `[tool.mypy]` → run `mypy .`

  ## Go

  - `go.mod` present → run `go test ./...`
  - Run `go vet ./...` unconditionally when `go.mod` is present

  ## Rust

  - `Cargo.toml` present → run `cargo test`
  - Run `cargo clippy` if available

  ## Makefile fallback

  If a `Makefile` exists, prefer its `test` / `lint` / `typecheck` targets
  over the ecosystem-specific commands above when both exist (the Makefile
  target is assumed to wrap the correct invocation for the project).

  ## Reporting

  For each detected and run check, capture: command run, exit code, and
  relevant output (failing test names, lint violations with file:line).
  A check with no detectable command for its category is omitted from the
  report entirely — never listed as skipped or failed.
  ```

- [ ] **Step 2: Add Step 2 to SKILL.md**

  Append to `skills/gate/SKILL.md` after Step 1:

  ```markdown
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
  ```

- [ ] **Step 3: Verify**

  Read both files back. Confirm `check-detection.md` covers Node/Python/Go/
  Rust/Makefile without placeholders, and SKILL.md's Step 2 references it by
  path and defines the `{check_name, command, exit_code, output}` shape that
  later steps will consume.

- [ ] **Step 4: Commit**

  ```bash
  git add skills/gate/SKILL.md skills/gate/references/check-detection.md
  git commit -m "feat(gate): add project check detection step"
  ```

---

### Task 3: Code review + finding classifier (Steps 3-4)

**Files:**
- Modify: `skills/gate/SKILL.md` (append after Step 2)

**Interfaces:**
- Consumes: check results list `{check_name, command, exit_code, output}` from Task 2.
- Produces: a unified findings list, each `{source, file, line, problem, fix_description, bucket}` where `bucket` is `auto-fix` or `ask-user`. This shape is consumed by Task 4 (display) and Task 5 (fix loop).

- [ ] **Step 1: Add Step 3 (code review) to SKILL.md**

  Append:

  ```markdown
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
  ```

- [ ] **Step 2: Add Step 4 (classifier) to SKILL.md**

  Append:

  ```markdown
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
  ```

- [ ] **Step 3: Verify**

  Read the appended sections. Confirm Step 3 explicitly says "inline, not a
  subagent" and Step 4's bucket definitions match the spec's auto-fix/
  ask-user line (mechanical + obvious one-fix bugs vs ambiguous/intent).

- [ ] **Step 4: Commit**

  ```bash
  git add skills/gate/SKILL.md
  git commit -m "feat(gate): add diff review and finding classification steps"
  ```

---

### Task 4: Finding display format (Step 5)

**Files:**
- Modify: `skills/gate/SKILL.md` (append after Step 4)

**Interfaces:**
- Consumes: classified findings list `{source, file, line, problem, fix_description, bucket}` from Task 3.

- [ ] **Step 1: Add Step 5 (display) to SKILL.md**

  Append:

  ```markdown
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
  ```

- [ ] **Step 2: Verify**

  Read the appended section. Confirm the example block matches the spec's
  finding-display format exactly (header line, per-source grouping, numbered
  ask-user findings, footer line).

- [ ] **Step 3: Commit**

  ```bash
  git add skills/gate/SKILL.md
  git commit -m "feat(gate): add findings display format"
  ```

---

### Task 5: Fix loop, action handlers, error handling (Step 6)

**Files:**
- Modify: `skills/gate/SKILL.md` (append after Step 5)

**Interfaces:**
- Consumes: findings list + bucket from Task 3, display format from Task 4.
- Produces: loop that terminates in either "clean" (all findings resolved) or "max-pass stop" — both consumed by Task 6's Step 7.

- [ ] **Step 1: Add Step 6 (fix loop + actions) to SKILL.md**

  Append:

  ```markdown
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
  ```

- [ ] **Step 2: Verify**

  Read the appended section. Confirm all four actions (`?`, `approve`,
  `fix`, `skip`, `stop`) are present, the max-pass guard is exactly 5, and
  error handling matches the spec's three cases (auto-fix regression,
  max-pass, git command failure).

- [ ] **Step 3: Commit**

  ```bash
  git add skills/gate/SKILL.md
  git commit -m "feat(gate): add fix loop with approve/fix/skip actions"
  ```

---

### Task 6: Clean handoff to /carve + final assembly

**Files:**
- Modify: `skills/gate/SKILL.md` (append after Step 6)

**Interfaces:**
- Consumes: "clean" termination state from Task 5's fix loop.

- [ ] **Step 1: Add Step 7 (handoff) to SKILL.md**

  Append:

  ```markdown
  ## Step 7 — Clean handoff

  Reached when every finding is resolved (auto-fixed, approved-and-fixed, or
  skipped) and no `ask-user` findings remain.

  Respond: "Working tree clean — N findings resolved this run. Run /carve to
  split into commits?"

  - If the user confirms (`yes` / `y` / `/carve`): invoke the `carve` skill.
  - If the user declines: respond "Done." and stop.
  ```

- [ ] **Step 2: Add Edge Cases and Context sections**

  Append, mirroring `carve`'s structure:

  ```markdown
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
  ```

- [ ] **Step 3: Verify full file end-to-end**

  Read the complete `skills/gate/SKILL.md` top to bottom. Confirm:
  - Frontmatter, Steps 1-7, Edge Cases, Context all present in order
  - No placeholder text ("TBD", "TODO", etc.)
  - Terminology consistent throughout: `auto-fix`/`ask-user` bucket names,
    `{source, file, line, problem, fix_description}` finding shape, same
    action verbs (`approve`/`fix`/`skip`/`stop`) used in both Step 6's
    definition and Step 5's footer hint text.

- [ ] **Step 4: Commit**

  ```bash
  git add skills/gate/SKILL.md
  git commit -m "feat(gate): add clean handoff to /carve and edge cases"
  ```

---

## Verification

Mental trace after all tasks are committed:

1. Dirty tree with: one lint-fixable issue (unused import), one obvious bug
   (off-by-one), one ambiguous review finding (missing backoff on retry
   loop), and project has `package.json` with `test`/`lint` scripts.
2. `/gate` → Step 1 reads diff (non-empty) → Step 2 detects and runs
   `npm test` + `npm run lint`, capturing results → Step 3 reviews diff,
   finds the ambiguous backoff issue → Step 4 classifies: unused import and
   off-by-one as `auto-fix`, backoff issue as `ask-user`.
3. Step 6 pass 1: auto-fixes the import and off-by-one, re-runs lint (now
   clean), displays per Step 5 format — one numbered `ask-user` finding
   remains.
4. User types `approve 1` → fix applied → pass 2 → no findings remain →
   Step 7: "Working tree clean — 3 findings resolved this run. Run /carve
   to split into commits?"
5. User confirms → `carve` skill invoked.

Also trace: empty working tree → "Nothing to gate." and stop before Step 2.
