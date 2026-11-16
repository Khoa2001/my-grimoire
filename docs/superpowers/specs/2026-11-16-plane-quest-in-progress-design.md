# plane-quest: In Progress on start

## Problem

plane-quest writes Plane state once, in Phase 6 seal, always In Review. While the agent conjures, builds and tests, the ticket keeps its old state (Backlog, Grooming, Ready, Todo). The board does not show that work started.

## Design

New **Phase 1b: Plane start**, between Phase 1 (check) and Phase 2 (conjure).

- Runs only when Phase 1 found gaps and no `seal` arg was given.
- Reads current state from the Phase 1 retrieve (`expand: "state"`).
- Current state is Backlog, Grooming, Ready or Todo: `state list` for the real id, then `workitem update` with the In Progress state id.
- Any other state (In Progress, In Review, Delivered, Verifying, UAT, Released, Done, Cancelled): no change. Note the current state in chat.
- State only. No comment.
- Update fails: report in chat, continue. Does not block the build.
- `/plane-quest` is already the go-ahead. No extra confirm.

Phase 1 stays read only. Phase 6 seal is unchanged (In Review on finish, ask when gaps remain).

## Approaches

- **A (picked):** after the gap report, before conjure. Matches the moment work starts.
- B: after design approval. Ticket stays in its old state during design talk. Rejected.
- C: at Phase 1 start. Moves to In Progress even when no gaps and the run jumps to seal. Rejected.

## Out of scope

- No Blocked state.
- No change to seal logic or comment template.
- No change to conjure or Plane MCP.
