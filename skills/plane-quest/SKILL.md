---
name: plane-quest
description: Take a Plane ticket (e.g. DOTB-550) from check to In Review in one run - Plane check, set In Progress when work starts, conjure for gaps, build, emulator test against prototype, carve ship, then Plane seal (state + short Vietnamese comment with one screenshot per done UI item). Use when user types /plane-quest <ticket> [seal] or asks to do/finish/report a Plane ticket end to end.
trigger: /plane-quest
---

# plane-quest

One command for the whole ticket loop. Steps that touch Plane are labeled **Plane**.

```
Plane check (read only) -> Plane start (In Progress) -> conjure -> build -> emulator test -> carve ship -> Plane seal
```

Input: `/plane-quest <ticket|link> [seal]`. Ticket id or link (`DOTB-550`, `https://plane.bba.dotb.vn/bba/browse/DOTB-550/`). A project issues-list link has no ticket: ask for the id.

`seal` arg: skip to Phase 6 (work already done by hand, only check + report). Same jump happens on its own when Phase 1 finds no gaps.

## Rules

- User typing `/plane-quest` is the go-ahead for: commit, push, PR, merge (via `carve` `ship`), Plane state changes (In Progress at start, In Review at seal), Plane comment, image upload. No extra confirm before posting.
- Gates that stay (do not skip): brainstorming design approval, writing-plans execution-mode choice. Carve plan view does not wait.
- Comment and ticket text: **Vietnamese, short, plain**. Reviewer has no source access: no file paths, class names, ADR numbers, commands, package names. Say what user sees and what is missing.
- Never claim done without checking code/tests/emulator. Facts from the repo, not from ticket text.
- Never set Done / Released / Delivered unless user says so.
- Plane writes: Phase 1b (state only, no comment) and Phase 6 (state, comment, images). Phase 1 is read only.

## Source lookup (before any question to user)

Applies in every phase, including brainstorming clarifying questions and any "waiting on khách/design/data" claim (palette, logo, font, PRD, SRS, prototype, wording, field list, flow).

Search in this order. Earlier wins on conflict (Drive docs win over the Plane ticket):

1. **Drive** - repo project: open `docs/external-docs.md`, then search the linked folders (PRD, SRS, Prototype, FRS, Open Question, Workshop, brand assets) with the Google Drive MCP: `ToolSearch select:mcp__claude_ai_Google_Drive__search_files,mcp__claude_ai_Google_Drive__read_file_content`. Read the app's PRD, SRS and prototype doc together.
2. **Plane** - ticket description, parent, comments (`workitem_comment list`), links, pages.
3. **Prototype** - Drive prototype notes, plus the running prototype (`ui_check.sh --proto` in the emsv3 repo).

Found: use it, note the source in the spec or comment. Never put an item under "Chưa xong" as blocked on others before this search is done.

All three silent: **stop and ask the user**. Show the question, which sources were searched, and the agent's own pick. Do not decide alone.

## Phases

### 1. Plane: check (read only)

1. `ToolSearch select:mcp__plane__workitem,mcp__plane__state,mcp__plane__workitem_comment,mcp__plane__workitem_attachment`.
2. `workitem retrieve_by_identifier` with `expand: "state"` (do not expand `parent`: it errors). Note `project_id`, `id`, description ("Cần làm" list = acceptance list). Read comments too.
3. Source lookup for the ticket's app (above).
4. Check each "Cần làm" item against the repo (grep, read, scoped tests; project CLAUDE.md rules apply). Sort into Đã xong / Chưa xong.
5. Report gaps in chat. No state change, no comment.
6. No gaps, or `seal` arg: go to Phase 6. Else Phase 1b.

### 1b. Plane: start

Gaps exist, no `seal` arg.

1. Current state comes from the Phase 1 retrieve. Backlog, Grooming, Ready or Todo: go on. Any other state (In Progress, In Review, Delivered, Verifying, UAT, Released, Done, Cancelled): do not change it, note the current state in chat, go to Phase 2.
2. `state list` for the In Progress id (project states, not guessed).
3. `workitem update` with `state: <In Progress id>`. State only, no comment.
4. Update fails: report in chat, go on. Does not block the build.

### 2. Conjure

Invoke Skill `conjure`, passing: ticket id, gap list, facts found in Drive/Plane/prototype. Follow it as written, including design approval and the writing-plans execution-mode choice. Spec lists the source for each decision. Open question with all sources silent: ask (see Source lookup).

### 3. Build

Execute the plan with the mode the user chose. Fix lint/test failures on the way.

### 4. Emulator test

Use the project's UI check tool, never raw `flutter run` / `xcrun simctl`. emsv3-mobile: `bash tools/ui_check.sh <parent|teacher|attendance> <route> [--tap <text>]... --proto` (takes simulator lock, run in background, timeout 600000). Screenshots land in `~/.cache/bba-ui-check/shots/`.

- Per UI item of the ticket: shoot app and prototype, Read both images, compare. Mismatch: fix, shoot again. Pixel match is the bar.
- Non-UI items: run scoped tests instead.
- Copy one final shot per done UI item into the session scratchpad (browser upload only accepts files inside session folders). Name them by item.

### 5. Carve ship

Invoke Skill `carve` with args `ship`. Commit groups, push, PR, merge, cleanup run without waiting; its guards still stop on failure. On stop: report and ask, do not seal yet.

### 6. Plane: seal

1. Re-check every "Cần làm" item against the final code. Sort Đã xong / Chưa xong.
2. `state list` (project states, not guessed). BBA workflow: Backlog, Grooming, Ready, Todo, In Progress, In Review ("người làm xong, chờ lead soát"), Delivered, Verifying, UAT, Released, Done, Cancelled. Dev finishing work -> **In Review**. Gaps remain and user did not choose: ask.
3. `workitem update` with `state: <state id>`.
4. `workitem_comment create` with short HTML (template below), no images yet.
5. Add images with Chrome (Plane MCP cannot upload local files; `upload_from_url` needs a public URL). One image per done item that has visible UI; items without UI get none. Read each image first to confirm it shows the claim.
   - `tabs_context_mcp` (createIfEmpty), `navigate` to the ticket page alone, wait ~4 s.
   - `find` the comment "..." menu -> Edit (own comment), click the end of the text, `cmd+End`.
   - For each image: click the editor toolbar **image** button (inserts an "Add an image" block, creates a hidden file input), `find` "file input in comment editor", `file_upload` the scratchpad PNG to that ref. Do not click file inputs directly (native picker is invisible).
   - Click the green check to save.
6. Fix placement: images land inside the last bullet, in upload order. `workitem_comment retrieve`, then `update` with HTML that keeps every `<image-component src="<uuid>" width height aspectratio alignment="left" status="uploaded">` tag, each moved to its own block right under its matching "Đã xong" bullet (close the list, image, reopen the list). Reload the page, screenshot, verify all render in the right place.
7. Close the browser tab you opened. Report to user: state set, what the comment says, which image sits under which item, PR link.

## Comment template (Vietnamese)

```html
<p><strong>Chuyển <State>: <lý do ngắn></strong></p>
<p><strong>Đã xong</strong></p>
<ul><li><p>...điều người dùng thấy được...</p></li></ul>
<image-component ...></image-component>
<ul><li><p>...mục tiếp theo...</p></li></ul>
<image-component ...></image-component>
<p><strong>Chưa xong</strong></p>
<ul><li><p>...còn thiếu, cần ai cung cấp gì...</p></li></ul>
```

Each image sits right under its bullet. Keep about 3 to 5 bullets per section (more only when the ticket lists more screens). Plain words ("màu theo trường của bé", not "BrandTokens"). Mention blockers on others (khách, API, design) in plain terms, only after Drive search found nothing.

## Gotchas

- `mcp__plane__workitem_relation list` returned 404: skip relation checks.
- Plane project id for DOTB Phase 1: `500fe30e-3793-4e6f-91ae-d0871d3f8969`. In Review state id: `b49a6834-5337-49ce-9d85-b238229e4776` (re-check with `state list` if it fails). In Progress state id: not recorded, always take it from `state list`.
- Widgetbook (`packages/bba_design_system/widgetbook`) does not build on Flutter 3.47.8 (`accessibility_tools 2.7.1`): use `ui_check.sh` for screenshots instead.
- Edited comments show "(edited)" in Plane: normal.
- Tool call batching: classifier may reject a navigate sent in the same response as other calls; send `navigate` alone.
- Ticket may be internal-only ("Ticket nội bộ DotB"): same flow.
