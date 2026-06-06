# Carve Output Polish — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Polish `/carve` plan display — drop redundant Reason label, trim file paths to last 2 segments, hide actions menu behind `?`.

**Architecture:** Single-file edit to `~/.claude/skills/carve/SKILL.md`. No code, no tests — verify by reading the updated file and doing a dry-run mental trace.

**Tech Stack:** Markdown skill file interpreted by Claude Code.

---

### Task 1: Update Step 4 display format

**Files:**
- Modify: `~/.claude/skills/carve/SKILL.md` (lines 44–75)

- [ ] **Step 1: Open the file and locate Step 4**

  Read `~/.claude/skills/carve/SKILL.md`. Confirm Step 4 starts at the line `## Step 4 — Display ALL groups together`.

- [ ] **Step 2: Replace the format block in Step 4**

  Replace the entire code block inside Step 4 (the ` ``` ` block showing the plan format) with:

  ````
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
  ````

  Also update the prose above the block to describe the new rules:

  ```markdown
  ## Step 4 — Display ALL groups together

  Show this exact format:

  - No `Reason:` line — the commit message already conveys intent.
  - Files: show the **last 2 path segments** of each file path, space-separated on one line.
    If a group has more than 3 files, wrap to a second indented line.
  - Footer: `N groups · M files  (? for help)` — no `Total:` prefix.
  - Do NOT show the actions menu here. It appears only when the user types `?`.
  ```

- [ ] **Step 3: Verify the edit**

  Re-read lines 44–76 of `~/.claude/skills/carve/SKILL.md`. Confirm:
  - No `Reason:` line in the example block
  - Files show as `dir/file.ext` format (2 segments), no `Files:` label
  - Footer reads `N groups · M files  (? for help)`
  - No Actions block in Step 4

- [ ] **Step 4: Commit**

  ```bash
  git add ~/.claude/skills/carve/SKILL.md
  git commit -m "feat(carve): compact plan display — drop Reason, trim paths, hide actions"
  ```

---

### Task 2: Add `?` action handler to Step 5

**Files:**
- Modify: `~/.claude/skills/carve/SKILL.md` (Step 5 section)

- [ ] **Step 1: Locate Step 5**

  Find `## Step 5 — Process user action` in the file.

- [ ] **Step 2: Add `?` handler as the first action**

  Insert this block at the top of the Step 5 action list (before `commit all`):

  ```markdown
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
  ```

- [ ] **Step 3: Verify**

  Re-read Step 5. Confirm `?` handler appears first, shows the actions block verbatim, then re-displays plan.

- [ ] **Step 4: Commit**

  ```bash
  git add ~/.claude/skills/carve/SKILL.md
  git commit -m "feat(carve): add ? help command to show actions on demand"
  ```

---

## Verification

After both tasks, do a mental trace:

1. Invoke `/carve` on a dirty tree with 3 groups.
2. Expected output matches the new format: no `Reason:`, files as `dir/file.ext`, footer `N groups · M files  (? for help)`.
3. Type `?` → actions block appears, then plan re-displays.
4. Type `commit all` → all groups committed normally (Step 5 logic unchanged).
