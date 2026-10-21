# Native command editor — Define the session protocol contract (Phase 1 of 3)

**Date:** 2026-10-20
**Original plan:** ../execution-plan.md
**Prerequisite:** None (first phase)
**Contains tasks:** {T1, T2, T3, T5}
**Unlocks next:** Phase 2

---

## Task List

Total: 4 tasks | Prerequisite phases must be complete before starting

- **T1:** Define the session protocol contract [prereq] → [tasks/T1-define-the-session-protocol-contract.md](tasks/T1-define-the-session-protocol-contract.md)
- **T2:** Strip session protocol bytes in the PTY reader [depends: T1] [parallel: T3] → [tasks/T2-strip-session-protocol-bytes-in-the-pty-reader.md](tasks/T2-strip-session-protocol-bytes-in-the-pty-reader.md)
- **T3:** Serialize submit and completion on the host [depends: T1] [test-risk] → [tasks/T3-serialize-submit-and-completion-on-the-host.md](tasks/T3-serialize-submit-and-completion-on-the-host.md)
- **T5:** Reduce authenticated shell events to an input mode [depends: T1] [parallel: T3] → [tasks/T5-reduce-authenticated-shell-events-to-an-input-mode.md](tasks/T5-reduce-authenticated-shell-events-to-an-input-mode.md)

---

## Phase Completion Gate

DONE when ALL of the following:
- Every task in this phase: status DONE
- All tests pass
- All commits created with correct format
- No task has status BLOCKED or NEEDS_CONTEXT

Hand off to Phase 2 ONLY after this gate passes.
