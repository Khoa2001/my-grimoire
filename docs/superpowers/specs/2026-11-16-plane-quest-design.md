# plane-quest design

Replaces `plane-seal`. One command runs the whole Plane ticket loop.

## Problem

Per ticket the user typed: `/plane-seal` -> gaps -> `/conjure` -> build -> emulator test -> `/carve` ship -> `/plane-seal`. Also:

- Agent asked the user for decisions that Drive docs (PRD, SRS, prototype notes, open questions) already answer.
- First `/plane-seal` run wrote state + comment before any work.
- Comment held only one image; tickets ask for many screens.
- `/carve` waited for input even after "ship" intent.

## Design

Phases: Plane check (read only) -> conjure -> build -> emulator test -> carve ship -> Plane seal.

- **Source lookup** (Drive -> Plane -> prototype, earlier wins) before any question or "blocked" claim. All silent: stop and ask, showing sources searched and own pick.
- **Phase 1 read only.** No gaps, or `seal` arg: jump to seal.
- **Gates kept:** brainstorming design approval, writing-plans execution choice. Dropped: carve plan wait, pre-post confirm.
- **Emulator test** via `ui_check.sh ... --proto`, compare app vs prototype per UI item, fix to pixel match.
- **Images:** one per done item with visible UI, placed under its bullet. Chrome upload loop, then one comment update to place tags.
- **carve** gains `ship` arg: show plan once, run ship at once.

## Out of scope

No change to superpowers skills, conjure, Plane MCP. No auto Done/Released.
