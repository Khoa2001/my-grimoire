---
name: plane-seal
description: Check whether a Plane ticket (e.g. DOTB-540) is done against the code, then update its state, post a short Vietnamese comment with a screenshot inside it. Use when user types /plane-seal <ticket> or asks to update/close/report a Plane ticket after work is done.
trigger: /plane-seal
---

# plane-seal

Check a Plane ticket against the real code, move its state, post one short Vietnamese comment with evidence image inside.

Input: ticket id or link (`DOTB-540`, `https://plane.bba.dotb.vn/bba/browse/DOTB-540/`). A project issues-list link has no ticket: ask for the id.

## Rules

- Comment and ticket text: **Vietnamese, short, plain**. Reviewer has no source access: no file paths, class names, ADR numbers, commands, package names. Say what user sees and what is missing.
- State change, comment, image upload are outward-facing: user's request to update the ticket is the approval, but ask first when unsure which state (see below).
- Evidence image goes **inside the comment**, not only in Attachments.
- Never claim done without checking code/tests. Facts from the repo, not from the ticket text.

## Steps

1. Load tools: `ToolSearch select:mcp__plane__workitem,mcp__plane__state,mcp__plane__workitem_comment,mcp__plane__workitem_attachment`.
2. Read ticket: `workitem retrieve_by_identifier` with `expand: "state"` (do not expand `parent`: it errors). Note `project_id`, `id`, description (the "Cần làm" list is the acceptance list).
3. Check each "Cần làm" item against the repo (grep, read, run scoped tests; project CLAUDE.md rules apply). Sort into Đã xong / Chưa xong.
4. Pick state. List states with `state list` (project states, not guessed). BBA workflow: Backlog, Grooming, Ready, Todo, In Progress, In Review ("người làm xong, chờ lead soát"), Delivered, Verifying, UAT, Released, Done, Cancelled. Dev finishing work -> **In Review**. Never Done/Released/Delivered unless user says so. If gaps remain and user did not choose, ask.
5. Evidence image (only if UI visible). For this repo: `bash tools/ui_check.sh <parent|teacher> <route>` (takes simulator lock, run background, timeout 600000), screenshot lands in `~/.cache/bba-ui-check/shots/`. Copy it into the session scratchpad first (browser upload only accepts files inside session folders). Read the image to confirm it shows the claim.
6. Update state: `workitem update` with `state: <state id>`.
7. Post comment: `workitem_comment create` with short HTML (template below).
8. Put image inside the comment. Plane MCP cannot upload local files (`upload_from_url` needs a public URL), so use Chrome:
   - `tabs_context_mcp` (createIfEmpty), `navigate` to the ticket page, wait ~4 s.
   - `find` the comment "..." menu -> Edit (own comment), click the end of the text, `cmd+End`.
   - Click the editor toolbar **image** button: it inserts an "Add an image" block and creates a hidden file input. `find` "file input in comment editor", then `file_upload` the scratchpad PNG to that ref.
   - Click the green check to save.
   - Do not click file inputs directly (native picker is invisible).
9. Fix placement: the image lands inside the last bullet. Re-read comment (`workitem_comment retrieve`), then `update` with HTML that keeps the `<image-component src="<uuid>" width height aspectratio alignment="left" status="uploaded">` tag, moved to its own block after the first list. Reload the page and screenshot to verify it renders.
10. Close the browser tab you opened. Report to user: state set, what the comment says, where image is.

## Comment template (Vietnamese)

```html
<p><strong>Chuyển <State>: <lý do ngắn></strong></p>
<p><strong>Đã xong</strong></p>
<ul><li><p>...điều người dùng thấy được...</p></li></ul>
<image-component ...></image-component>
<p><strong>Chưa xong</strong></p>
<ul><li><p>...còn thiếu, cần ai cung cấp gì...</p></li></ul>
```

Keep to about 3 to 5 bullets per section. Plain words ("màu theo trường của bé", not "BrandTokens"). Mention blockers on others (khách, API, design) in plain terms.

## Gotchas

- `mcp__plane__workitem_relation list` returned 404: skip relation checks.
- Plane project id for DOTB Phase 1: `500fe30e-3793-4e6f-91ae-d0871d3f8969`. In Review state id: `b49a6834-5337-49ce-9d85-b238229e4776` (re-check with `state list` if it fails).
- Widgetbook (`packages/bba_design_system/widgetbook`) does not build on Flutter 3.47.8 (`accessibility_tools 2.7.1`): use `ui_check.sh` for screenshots instead.
- Edited comments show "(edited)" in Plane: normal.
- Tool call batching: classifier may reject a navigate sent in the same response as other calls; send `navigate` alone.
- Ticket may be internal-only ("Ticket nội bộ DotB"): same flow.
