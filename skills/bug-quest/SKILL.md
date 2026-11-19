---
name: bug-quest
description: Hunt bugs in a feature, an app or all apps and log each one as a GitHub issue (hunt), or take an issue number, reproduce it, fix it and move its status label (fix). Use when user types /bug-quest, or asks to test/explore a feature or the whole app and log bugs to GitHub issues, or to fix/explore a logged bug by issue number. Repo facts come from docs/guides/hunt-bugs.md.
trigger: /bug-quest
---

# bug-quest

Two modes over one tracker: GitHub Issues of the current repo are the agent bug list. Plane is not touched.

```
hunt: scope -> sources -> desk pass + simulator pass -> candidates -> issues (all of them) -> evidence pictures
fix:  issue -> reproduce -> status labels -> root cause -> test -> fix -> comment -> (ship) PR "Fixes #N"
```

Input:
- `/bug-quest hunt <app|all> [feature]` - `app` is one of the apps in the guide (emsv3-mobile: `parent`, `teacher`, `centre_director`, `attendance`); `all` = every app, one app at a time.
- `/bug-quest fix <#N|issue url> [ship]` - `ship` lets the agent commit, push and open the PR.

Missing or unclear scope: ask the user (show the app and feature list from the guide).

## Rules

0. **Read the repo guide first**: `docs/guides/hunt-bugs.md` (emsv3-mobile). It holds the features, routes, `ui_check.sh` calls, sources, bug kinds, issue format and labels. No guide in the repo: stop and tell the user. Also read the repo `CLAUDE.md`.
1. **Every bug found is filed as an issue, no approval step** (owner decision: no need to check first). The only things not filed: duplicates of an existing issue, and things the repo calls intended (rule 7). Unsure findings are filed too, with the `needs-triage` label and "Unsure:" at the start of the body, so nothing found is lost.
2. **Issue titles, bodies and comments are Vietnamese only**: short, clean, precise, what the user sees. Code, paths, commands, label names, error text stay as they are. Follow the guide's body template ("Issue format").
3. **No commit, push, merge** unless the user said `ship` or said so in words. Fix mode never merges.
4. Subagents (desk pass, reviews) run on `opus`.
5. Use the project's tools only: `ui_check.sh` for the simulator (never raw `flutter run` or `xcrun simctl`), scoped `flutter analyze` / `flutter test` (never `melos run test|analyze|format-check|check-coverage`). One simulator: sessions queue on a lock; run in background or with tool timeout 600000.
6. Gate or kiosk apps (Attendance) are checked on the iPad simulator (its default), never an iPhone. Parent and Teacher run with `UI_CHECK_DEVICE="iPhone 17" bash tools/ui_check.sh ...`, never the default 17e (other agents use it). That width is not the prototype's, so no `--proto` side by side on iPhone 17: pixel bugs are measured in points against the prototype source (guide). Use the exact env var form the guide shows.
7. A found "bug" that the repo calls intended ("SRS beats the prototype", not built yet, fake data) is not a bug. Not sure whether it is a bug: file it as "unsure" (rule 1).

## hunt

### 1. Scope and sources

1. Resolve scope with the guide's feature tables. `all` or a whole app: work one feature at a time, one app at a time; tell the user the count first.
2. Per feature collect: its part of `docs/feature-notes.md`, its ADR and spec, the rules in Drive (PRD, SRS; search by feature name and rule id; see `docs/external-docs.md`), the Plane ticket (Plane connector, search the feature name), the prototype source (`~/Downloads/bba-*`). Authority order: Plane ticket, Drive docs, prototype. All silent on a point: that point is "unsure", not a bug.
3. Dedupe happens per candidate in step 4 with the guide's search (`gh issue list --state all --search "<words> in:title,body" --limit 20`), not from one big list.

### 2. Desk pass (parallel, opus)

One subagent per feature (at most 3 at a time), each told: the feature folder, its sources, and the guide's `rule` and `code` kinds. It returns candidates only; no edits. It runs scoped `flutter analyze` and `flutter test` for the feature's packages (shell set up by `eval "$(bash tools/flutter_env.sh)"`), compares every rule id against code and fake data, and lists rules with no test.

### 3. Simulator pass (you, one at a time)

Features with a route only (the guide says which are desk only):
Flag rules (the guide has them; `ui_check.sh` exits 2 otherwise): `--motion` needs a step and no `--each`; `--proto` replays `--tap` steps only; `--proto` with `--each` and a non-tap step exits 2; `--proto --motion` takes tap steps only. Parent and Teacher: `UI_CHECK_DEVICE="iPhone 17"` and no `--proto`.
1. Open: `bash tools/ui_check.sh <app> <route>` in the background (Attendance may add `--proto`); Read the screenshot.
2. Flows: add `--tap`, `--tap-label`, `--type`, `--scroll`, `--back` steps; `--each` shoots every step. Try empty, long text, failure paths the fake data has. Offline is a desk pass (read bloc, repository, tests); the simulator cannot go offline here.
3. `pixel`: take the prototype numbers from its source (Tailwind classes) and measure the app shot in points (guide: `match-a-prototype.md`). Attendance may use `--proto` with `--tap` steps.
4. `motion`: `--tap <text> --motion 800 --motion-frames 8` (tap steps only if `--proto` is added, Attendance only); Read the frames picture.
5. Copy each shot that shows a bug into the session scratchpad with a clear name (`<app>-<feature>-<short>.png`). Browser upload only accepts files in session folders.

### 4. Candidates

Per candidate record in scratchpad `bugs.json`: `n`, `app`, `featureShort` (for the title), `featureDir` (the full folder name in `features/`, for the label; none for Centre Director), `kind` (`flow|rule|pixel|motion|code`), `sev` (`high|medium|low`), `title` (`[<app>/<featureShort>] <plain sentence>`), `steps`, `expected`, `actual`, `source` (rule id, prototype file, test name), `repro` (a command that runs from the repo root), `images` (scratchpad paths), `duplicateOf` (issue number or null, from the per-candidate `gh issue list --search` above). Drop what the guide says not to file.

### 5. Sort

Candidates with `duplicateOf` are not filed (list them in the report). Candidates the guide calls intended are dropped. Everything else is filed in step 6; mark the unsure ones `unsure: true`.

### 6. File every bug

1. Labels: build the label list of every issue to file (`bug`, `status:found`, `app:<app>`, `feat:<featureDir>`, `kind:*`, `sev:*`, plus `needs-triage` when unsure), then check each one against `gh label list --limit 200 --json name --jq '.[].name'`. Any missing: run `bash tools/github_labels.sh` (idempotent, no need to ask).
2. Per bug: `gh issue create --title "<title>" --body-file <file> --label bug,status:found,app:<app>,feat:<featureDir>,kind:<kind>,sev:<sev>` (no `feat:` for Centre Director) with the body from the guide ("Issue format", headings = the bug form). `app` label for centre_director is `app:centre-director`. Write the body to a scratchpad file; do not inline it in the shell.
3. Account: issues are made by the sub account in the guide ("Account"; `khoaculi`), and the owner's main account (`Khoa2001`) must be active again afterwards. Do steps 1 to 2 (all the `gh` calls) in ONE bash script: read the active account, `gh auth switch --user khoaculi`, an `EXIT` trap that switches back to the saved account, then the `gh issue create` / `gh label` / `gh pr` calls with `-R <owner>/<repo>` and absolute `--body-file` paths. The guide has the exact lines. After the script run `gh auth status`: the active account is the same as before. Never print a token. Switch fails: stop, tell the user to run `gh auth login` as that account.
4. Keep the issue URL for each.

### 7. Evidence pictures (Chrome)

`gh` cannot upload images. For each issue that has pictures:
1. Load the chrome skill/tools in one ToolSearch call; `tabs_context_mcp` (createIfEmpty), then `navigate` to the issue URL alone in its own call, wait about 4 s.
2. Chrome must be the owner's profile `crestinavn` (signed in as `khoaculi`): call `list_connected_browsers`, then `switch_browser` (the owner clicks Connect in that profile); "Browser 1/2" names do not say which profile. GitHub makes its file input only on click and a real click opens an invisible native picker, so catch the input first (tested, works for a PNG): with `javascript_tool` on the issue tab run
   ```js
   const o = HTMLInputElement.prototype.click;
   HTMLInputElement.prototype.click = function () { if (this.type === 'file') { this.id = 'bq-file'; document.body.appendChild(this); return; } return o.apply(this, arguments); };
   [...document.querySelectorAll('button')].find(b => /add files/i.test(b.textContent)).click();
   ```
   (the tool may print "BLOCKED"; the code still ran: check with `document.getElementById('bq-file') ? 'ok' : 'no'`). Then `find` "file input" (it shows as `file-input`), click the comment box, type `Bằng chứng: <what the picture shows>`, and `file_upload` each scratchpad file (PNG, and a video up to GitHub's size limit: 10 MB on a free plan, untested) to that ref. Wait about 4 s until the box holds an `<img ... src="https://github.com/user-attachments/...">` line (a video becomes a bare user-attachments URL) in place of "Uploading". One file per call. If no input after 2 tries: stop, report the files left on disk, do not loop.
3. `find` the "Comment" button, click it. Screenshot, confirm the picture renders.
4. Upload fails (Chrome closed, not signed in): the issue still stands; report which pictures are missing and where they are on disk. Close the tab you opened.

### 8. Report

Issue links with title, kind, sev; what was skipped (duplicates, unsure); what was desk-only and why; machine facts that mattered (simulator, flavor).

## fix

### 1. Take the issue

`gh issue view <N> --comments --json number,title,state,labels,body,comments`.
- Not open, or has `status:in-progress` or `status:in-review`, or is not `status:found` or `status:confirmed`: stop and ask the user.
- Read the body: steps, source, repro command, evidence comment (Read the pictures).

### 2. Reproduce first, as a user sees it

Run the repro command (or the steps through `ui_check.sh`). Read the result.
- Cannot reproduce: `gh issue edit <N> --add-label status:cannot-repro --remove-label status:found` (or `status:confirmed`), comment what was tried (command, flavor, what was seen), stop.
- Reproduced: `gh issue edit <N> --add-label status:confirmed --remove-label status:found` (skip if already confirmed), then in one call `--add-label status:in-progress --remove-label status:confirmed`. Exactly one `status:*` label at a time.

### 3. Work

1. Worktree on branch `fix/issue-<N>-<slug>` (use `superpowers:using-git-worktrees`; branch name has no `worktree-` prefix).
2. `superpowers:systematic-debugging` for the root cause; `superpowers:test-driven-development`: a test that fails for this bug first, then the fix.
3. Scoped checks for the changed packages (guide: `melos run check-changed --smart` while working, and plain `melos run check-changed` (`--no-cache` if a test reads files outside its package) before `gh pr create`, or `cd <package> && flutter analyze && flutter test`); `dart format` on changed files. UI bug: shoot the screen again with the same `ui_check.sh` line and compare with the prototype.
4. Docs part of change: update docs the fix touches (repo `CLAUDE.md` says which).
5. Comment on the issue in basic English: root cause, what changed, test name, after picture (upload through Chrome as in hunt step 7; the shots go in the session scratchpad).

### 4. Ship (only with `ship` or the user's word)

1. Commit through the `carve` skill (`carve` args as the user wants; plain `carve`, not `carve ship`: no merge).
2. Push, then `gh pr create` with a body that starts with `Fixes #<N>`, then what changed and how it was checked. Branch name on remote has no `worktree-` prefix.
3. `gh issue edit <N> --add-label status:in-review --remove-label status:in-progress`.
4. Stop. Report the PR link. Never merge. The merge closes the issue.

Without `ship`: stop after step 3.5, report, and ask before any commit.

## Gotchas

- `needs-triage` is on the bug form; hunt issues do not carry it.
- Label script changes GitHub: ask once before the real run, dry run is free.
- `gh` cannot upload images; Chrome `navigate` goes in its own tool call.
- Plane is separate: do not comment or change state there from this skill.
- Duplicate check uses titles and bodies; when two bugs share a root cause but different screens, file both and cross-link with `Related to #N`.
- `gh` run outside the repo root fails ("not a git repository"): pass `-R <owner>/<repo>` (from `gh repo view --json nameWithOwner`) and absolute `--body-file` paths.
- Chrome upload needs the profile signed in as the sub account (`crestinavn`, `khoaculi`). A 404 on the issue URL means a profile without access: stop, do not sign in for the user, report the pictures left on disk.
- Deleting an issue (only when the owner asks) needs the owner account: `gh issue delete <N> -R <owner>/<repo> --yes` while `Khoa2001` is active.
- Fix mode uses the same sub account (hunt step 6.3) for label moves, comments and `gh pr create`.
