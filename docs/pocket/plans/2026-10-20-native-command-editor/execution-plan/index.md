# Native command editor — Execution Index

**Date:** 2026-10-20
**Spec:** docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
**Source Plan:** ../execution-plan.md
**source-sha256:** 41a9f05a2f83ddebee7603f0cfae4924c03537626d73fa87e3692c9152748249
**Total Tasks:** 12
**Total Phases:** 3

---

## Execution Flow

```
T1→T2,T3,T5(PARALLEL)→T4,T6,T7,T9(PARALLEL)→T8,T10,T11(PARALLEL)→T12
```

---

## Phase Summary

- **Phase 1:** [phase-1.md](phase-1.md) — Define the session protocol contract (T1, T2, T3, T5)
- **Phase 2:** [phase-2.md](phase-2.md) — Inject bash and zsh integration (T4, T6, T7, T9)
- **Phase 3:** [phase-3.md](phase-3.md) — Accept a suggestion without running it (T8, T10, T11, T12)

---

## Task Index

| Task ID | Name | Phase | Task File | Annotation |
|---|---|---|---|---|
| T1 | Define the session protocol contract | Phase 1 | [T1-define-the-session-protocol-contract.md](tasks/T1-define-the-session-protocol-contract.md) | [prereq] |
| T2 | Strip session protocol bytes in the PTY reader | Phase 1 | [T2-strip-session-protocol-bytes-in-the-pty-reader.md](tasks/T2-strip-session-protocol-bytes-in-the-pty-reader.md) | [depends: T1] [parallel: T3] |
| T3 | Serialize submit and completion on the host | Phase 1 | [T3-serialize-submit-and-completion-on-the-host.md](tasks/T3-serialize-submit-and-completion-on-the-host.md) | [depends: T1] [test-risk] |
| T5 | Reduce authenticated shell events to an input mode | Phase 1 | [T5-reduce-authenticated-shell-events-to-an-input-mode.md](tasks/T5-reduce-authenticated-shell-events-to-an-input-mode.md) | [depends: T1] [parallel: T3] |
| T4 | Inject bash and zsh integration | Phase 2 | [T4-inject-bash-and-zsh-integration.md](tasks/T4-inject-bash-and-zsh-integration.md) | [depends: T1, T2] [test-risk] |
| T6 | Schedule completion and record submit on the client | Phase 2 | [T6-schedule-completion-and-record-submit-on-the-client.md](tasks/T6-schedule-completion-and-record-submit-on-the-client.md) | [depends: T5] [parallel: T7] [test-risk] |
| T7 | Mount the command editor and keep its draft | Phase 2 | [T7-mount-the-command-editor-and-keep-its-draft.md](tasks/T7-mount-the-command-editor-and-keep-its-draft.md) | [depends: T5] |
| T9 | Send submit and completion on both transports | Phase 2 | [T9-send-submit-and-completion-on-both-transports.md](tasks/T9-send-submit-and-completion-on-both-transports.md) | [depends: T1, T3] [parallel: T6] |
| T8 | Accept a suggestion without running it | Phase 3 | [T8-accept-a-suggestion-without-running-it.md](tasks/T8-accept-a-suggestion-without-running-it.md) | [depends: T6, T7] |
| T10 | Add the automatic completion refresh preference | Phase 3 | [T10-add-the-automatic-completion-refresh-preference.md](tasks/T10-add-the-automatic-completion-refresh-preference.md) | [depends: T6] |
| T11 | Route palette, assist, and mobile input through the editor | Phase 3 | [T11-route-palette-assist-and-mobile-input-through-the-editor.md](tasks/T11-route-palette-assist-and-mobile-input-through-the-editor.md) | [depends: T7] |
| T12 | Prove the reader delivery and submit binder | Phase 3 | [T12-prove-the-reader-delivery-and-submit-binder.md](tasks/T12-prove-the-reader-delivery-and-submit-binder.md) | [depends: T2, T3, T5, T6, T7, T8, T9, T11] [test-risk] |
