# Native command editor — Inject bash and zsh integration (Phase 2 of 3)

**Date:** 2026-10-20
**Original plan:** ../execution-plan.md
**Prerequisite:** Phase 1 must be COMPLETE — all tests green, all commits created
**Contains tasks:** {T4, T6, T7, T9}
**Unlocks next:** Phase 3

---

## Task List

Total: 4 tasks | Prerequisite phases must be complete before starting

- **T4:** Inject bash and zsh integration [depends: T1, T2] [test-risk] → [tasks/T4-inject-bash-and-zsh-integration.md](tasks/T4-inject-bash-and-zsh-integration.md)
- **T6:** Schedule completion and record submit on the client [depends: T5] [parallel: T7] [test-risk] → [tasks/T6-schedule-completion-and-record-submit-on-the-client.md](tasks/T6-schedule-completion-and-record-submit-on-the-client.md)
- **T7:** Mount the command editor and keep its draft [depends: T5] → [tasks/T7-mount-the-command-editor-and-keep-its-draft.md](tasks/T7-mount-the-command-editor-and-keep-its-draft.md)
- **T9:** Send submit and completion on both transports [depends: T1, T3] [parallel: T6] → [tasks/T9-send-submit-and-completion-on-both-transports.md](tasks/T9-send-submit-and-completion-on-both-transports.md)

---

## Phase Completion Gate

DONE when ALL of the following:
- Every task in this phase: status DONE
- All tests pass
- All commits created with correct format
- No task has status BLOCKED or NEEDS_CONTEXT

Hand off to Phase 3 ONLY after this gate passes.
