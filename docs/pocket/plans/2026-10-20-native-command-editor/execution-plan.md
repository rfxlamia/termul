# EXECUTION PLAN — Native command editor

**Date:** 2026-10-20
**Spec:** docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
**Status:** draft
**Total tasks:** 12

---

## Preflight

```
PREFLIGHT COMPLETE
Codebase scanned: ConnectedTerminal.tsx, keymap.ts, MobileTerminalControls.tsx, use-terminal-clipboard.ts, use-command-history.ts, use-command-palette-data.ts, terminal-api.ts, tauri-terminal-api.ts, web-terminal-api.ts, ipc.types.ts, settings.ts, appearance-section.tsx, pty/manager/spawn.rs, pty/manager/lifecycle.rs, pty/da_filter.rs, pty/mod.rs, commands/terminal.rs, commands/mod.rs, lib.rs invoke_handler, web/terminal_ws.rs, remote/host.rs, trackers/exit_code_tracker.rs, trackers/cwd_tracker.rs, trackers/terminal_events.rs
Test framework: Vitest 4.1.6 via `bun run test -- <file> -t "<name>"`. React Testing Library is already used by ConnectedTerminal.test.tsx. Rust tests run with `cargo test --manifest-path src-tauri/Cargo.toml --lib <filter>`. Sibling tests follow `da_filter.rs` plus `da_filter/tests.rs`.
File conventions: host behavior behind terminalApi; log crate in Rust and logFrontendError in src/renderer/lib/log-api.ts; no tracing. New production files stay under about 800 lines. Do not split an existing file (issue #1487).
Existing helpers: useAddCommand, isAppOwnedTerminalShortcut, DaFilter, PtyManager::write, TerminalEventHub, isTauriContext, DEFAULT_APP_SETTINGS. Reuse them. Do not reimplement history trimming or the DA filter.
Library docs checked: @xterm/xterm@6.1.0-beta.287 (installed typings plus Context7 /websites/xtermjs on 2026-10-20). registerOscHandler fires once when the sequence finishes, holds chunks until then, and limits OSC payload to 10 MB. @codemirror/autocomplete@6.20.3 is transitive in bun.lock; CompletionConfig.defaultKeymap installs completionKeymap at Prec.highest unless false, and that keymap binds Enter to acceptCompletion. activateOnTyping defaults true. activateOnTypingDelay defaults to 100. selectOnOpen defaults to the first row. The Completion type has no ghost-text field. @codemirror/view@6.43.4 and @codemirror/state@6.7.0 are direct. Existing tests already construct EditorView and call runScopeHandlers.
Key findings: WebTerminalClient.write reports success for an offline buffer and flushes at-least-once. Web "write" is also ledgered in terminal_ws/commands.rs. Submit and completion must be new RPCs, not that path. PtyManager::write is write_all plus flush and does not report a byte count. Desktop shared-live sets allow_remote_writes false. TerminalEventHub is a 256-event broadcast. ConnectedTerminal.tsx is 2007 lines and web-terminal-api.ts is 1597. spawn.rs is 618 and lifecycle.rs is 466.
Unverified areas: a private OSC ident can still collide with a program the user runs. The chosen ident is 9477 only because it is not 133, 633, or 9278. A third-party completer can still escape a wrapper that does not exist yet. Both risks stay fail-closed in the packets that own them.
```

---

## Execution Overview

### Recommended Order

```
T1 → T2, T3, T5 (parallel) → T4, T6, T7, T9 → T8, T10, T11 → T12
```

> Dependency order above is **recommended**. pocket-development enforces the actual order.

Twelve tasks are one plan because every later seam uses the T1 frame. The work groups as protocol (T1–T4, T9), editor (T5–T8, T10, T11), then the cross-unit proof (T12). That grouping is for review. It does not split this file into phase files.

### Parallelizable Groups

| Group | Tasks | Unblocked After |
|-------|-------|-----------------|
| Group A | T2, T3, T5 | T1 completes |
| Group B | T4 | T1 and T2 complete |
| Group C | T6, T7 | T5 completes |
| Group D | T9 | T1 and T3 complete |
| Group E | T8 | T6 and T7 complete |
| Group F | T10 | T6 completes |
| Group G | T11 | T7 completes |
| Group H | T12 | T2, T3, T5, T6, T7, T8, T9, and T11 complete |

### Constraints Reminder

**Architecture:** New behavior lives in new files. An existing file gets only a call, a method, a match arm, or a settings field. Do not split, move, or reorganize `ConnectedTerminal.tsx`, `web-terminal-api.ts`, `spawn.rs`, `lifecycle.rs`, `lib.rs`, or `terminal_ws.rs`. Issue #1487 owns those splits. Protocol bytes are stripped in the host reader. Completion results return on the caller RPC, not on `TerminalEventHub`. `InitShell`, `PromptReady`, `CommandStarted`, `CommandFinished`, and `Desync` publish `TerminalEvent::SessionLifecycle` with `session_id` and no completion payload and no draft. Submit and completion do not use `inputBuffers` or the ledgered `write` path. `defaultKeymap: false` and `activateOnTyping: false`. Logs omit the draft, paste, env, and completion payload. Desktop invoke names and websocket `type` values are `command_editor_submit`, `command_editor_request_completion`, `command_editor_cancel_completion`, and `command_editor_restore_prompt`.

**Out-of-scope:** Output blocks, the AI box, a Node chip, replacing xterm or the PTY, a pretyping queue, fish/PowerShell/cmd/dash/SSH injection, a force-raw preference, the landing site, copying Warp source, and splitting an existing file.

**Assumptions at risk:** Automatic completion refresh defaults to true. The completion timeout has no numeric value in the spec. OSC 9477 is unverified against every program. The completer wrapper is unverified until it exists.

**Sequencing:** `[depends: TN]` is the recommended block. A task that cannot compile without a prior type really waits.

### File Structure Map

```
Rule: Trusted prompt
  Create: src/shared/types/command-session.ts (created by: T1)
  Create: src/shared/types/command-session.test.ts (created by: T1)
  Create: src-tauri/src/pty/session_protocol/mod.rs (created by: T1)
  Create: src-tauri/src/pty/session_protocol/frame.rs (created by: T1)
  Create: src-tauri/src/pty/session_protocol/tests.rs (created by: T1)
  Create: src-tauri/src/pty/session_protocol/inject.rs (created by: T4)
  Create: src-tauri/src/pty/session_protocol/inject_tests.rs (created by: T4)
  Create: src-tauri/resources/shell-integration/bash.sh (created by: T4)
  Create: src-tauri/resources/shell-integration/zsh.sh (created by: T4)
  Create: src/renderer/lib/command-lifecycle/transition.ts (created by: T5)
  Create: src/renderer/lib/command-lifecycle/transition.test.ts (created by: T5)
  Create: src/renderer/components/terminal/command-editor/TerminalInputSurface.tsx (created by: T7)
  Create: src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx (created by: T7)
  Create: src/renderer/lib/command-lifecycle/screen-mode.ts (created by: T7)
  Create: src/renderer/lib/command-lifecycle/screen-mode.test.ts (created by: T7)
  Read: src/renderer/utils/terminal-registry.ts (`getTerminalModes` only; do not edit)
  Modify: src-tauri/src/pty/mod.rs
  Modify: src-tauri/src/pty/manager/spawn.rs
  Test: src-tauri/src/pty/session_protocol/inject_tests.rs
  Test: src/renderer/lib/command-lifecycle/transition.test.ts
  Test: src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx
  Test: src/renderer/lib/command-lifecycle/screen-mode.test.ts

Rule: Submit once
  Create: src-tauri/src/pty/session_protocol/gate.rs (created by: T3)
  Create: src-tauri/src/pty/session_protocol/gate_tests.rs (created by: T3)
  Create: src-tauri/src/commands/command_editor.rs (created by: T3)
  Create: src-tauri/src/web/terminal_ws/command_editor.rs (created by: T3)
  Create: src-tauri/src/web/terminal_ws/command_editor_tests.rs (created by: T3)
  Create: src/renderer/lib/command-lifecycle/submit.ts (created by: T6)
  Create: src/renderer/lib/command-lifecycle/submit.test.ts (created by: T6)
  Modify: src/shared/types/ipc.types.ts
  Modify: src-tauri/src/lib.rs
  Modify: src-tauri/src/commands/mod.rs
  Modify: src-tauri/src/web/terminal_ws.rs
  Test: src-tauri/src/pty/session_protocol/gate_tests.rs
  Test: src/renderer/lib/command-lifecycle/submit.test.ts

Rule: Completion scheduling
  Create: src/renderer/lib/command-lifecycle/completion.ts (created by: T6)
  Create: src/renderer/lib/command-lifecycle/completion.test.ts (created by: T6)
  Create: src-tauri/src/pty/session_protocol/filter.rs (created by: T2)
  Create: src-tauri/src/pty/session_protocol/filter_tests.rs (created by: T2)
  Modify: src/renderer/types/settings.ts
  Modify: src/renderer/pages/AppPreferences.tsx
  Modify: src/renderer/pages/app-preferences/appearance-section.tsx
  Test: src/renderer/lib/command-lifecycle/completion.test.ts
  Test: src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx

Rule: Accept does not execute
  Create: src/renderer/components/terminal/command-editor/completion-view.ts (created by: T8)
  Create: src/renderer/components/terminal/command-editor/completion-view.test.ts (created by: T8)
  Modify: package.json
  Modify: bun.lock
  Test: src/renderer/components/terminal/command-editor/completion-view.test.ts

Rule: Editor isolation
  Create: src/renderer/components/terminal/command-editor/CommandEditor.tsx (created by: T7)
  Create: src/renderer/components/terminal/command-editor/CommandEditor.test.tsx (created by: T8)
  Create: src/renderer/components/terminal/command-editor/editor-keys.ts (created by: T7)
  Create: src/renderer/components/terminal/command-editor/editor-keys.test.ts (created by: T7)
  Create: src/renderer/components/terminal/command-editor/chips.tsx (created by: T7)
  Create: src/renderer/components/terminal/command-editor/chips.test.tsx (created by: T7)
  Create: src/renderer/lib/command-lifecycle/draft-store.ts (created by: T7)
  Create: src/renderer/lib/command-lifecycle/draft-store.test.ts (created by: T7)
  Create: src/renderer/components/terminal/command-editor/foreign-insert.ts (created by: T11)
  Create: src/renderer/components/terminal/command-editor/foreign-insert.test.ts (created by: T11)
  Modify: src/renderer/components/terminal/ConnectedTerminal.tsx
  Modify: src/renderer/components/mobile/MobileTerminalControls.tsx
  Modify: src/renderer/layouts/workspace-layout/use-command-palette-data.ts
  Test: src/renderer/components/terminal/command-editor/editor-keys.test.ts
  Test: src/renderer/components/terminal/command-editor/foreign-insert.test.ts

Rule: Surfaces
  Create: src/renderer/lib/command-editor-transport.ts (created by: T9)
  Create: src/renderer/lib/command-editor-transport.test.ts (created by: T9)
  Create: src-tauri/src/pty/session_protocol/deliver.rs (created by: T12)
  Create: src-tauri/src/pty/session_protocol/isolation_tests.rs (created by: T12)
  Create: src/renderer/lib/command-lifecycle/bind-session.ts (created by: T12)
  Create: src/renderer/lib/command-lifecycle/submit-integration.test.ts (created by: T12)
  Create: src/renderer/lib/command-lifecycle/session-store.ts (created by: T12)
  Create: src/renderer/lib/command-lifecycle/session-store.test.ts (created by: T12)
  Modify: src-tauri/src/trackers/terminal_events.rs
  Modify: src/shared/types/web-terminal-protocol.types.ts
  Modify: src/renderer/lib/tauri-terminal-api.ts
  Modify: src/renderer/lib/web-terminal-api.ts
  Modify: src/renderer/lib/__tests__/parity-checklist.test.ts
  Test: src/renderer/lib/command-editor-transport.test.ts
  Test: src/renderer/lib/__tests__/parity-checklist.test.ts
  Test: src-tauri/src/pty/session_protocol/isolation_tests.rs
  Test: src/renderer/lib/command-lifecycle/submit-integration.test.ts
  Test: src/renderer/lib/command-lifecycle/session-store.test.ts

Read only, do not modify:
  src-tauri/src/pty/da_filter.rs
  src-tauri/src/pty/env_refresh.rs
  src-tauri/src/pty/manager/lifecycle.rs
  src-tauri/src/remote/host.rs
  src-tauri/src/web/capabilities.rs
  src-tauri/src/web/terminal_ws/commands.rs
  src/renderer/components/terminal/keymap.ts
  src/renderer/hooks/use-codemirror.test.ts
  src/renderer/lib/terminal-api.ts
  src/renderer/hooks/use-command-history.ts
```

---

## Pocket Packets

---

### Task 1: Define the session protocol contract [prereq]

## OBJECTIVE

Add the shared frame and RPC names that Rust and TypeScript both use. No shell, no editor, and no PTY write.

Steps:
1. Write failing test for: private protocol ident
   Test file: `src-tauri/src/pty/session_protocol/tests.rs`
   Level: unit
   Test intent: Given the session protocol ident, When it is read, Then it equals 9477 and it is not 133, 633, or 9278
   Exercise through: the public `TERMUL_SESSION_OSC` constant
   Test doubles: none
   Expected RED: the module does not exist, so the test does not compile
2. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::protocol_ident_is_private`
3. Add `src-tauri/src/pty/session_protocol/mod.rs` and the constant. Re-run step 2 and verify PASS. Refactor while green. Commit: `git commit -m "feat(pty): add the private session protocol ident"`
4. Write failing test for: completion validity fields
   Test file: `src/shared/types/command-session.test.ts`
   Level: unit
   Test intent: Given a completion result missing session id, nonce, request id, buffer revision, cursor, prompt generation, or cwd, When it is parsed, Then it is rejected
   Exercise through: `parseSessionFrame` in `src/shared/types/command-session.ts`
   Test doubles: none
   Expected RED: `parseSessionFrame` does not exist
5. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "completion validity fields"`
6. Implement the parser and the frame names `InitShell`, `PromptReady`, `CommandStarted`, `CommandFinished`. Completion payload is a result struct, not a mode frame. A nonce check is not part of this step. Re-run step 5 and verify PASS. Refactor while green. Commit: `git commit -m "feat(shared): parse session protocol frames"`
7. Write failing test for: a prompt frame keeps its fields
   Test file: `src/shared/types/command-session.test.ts`
   Level: unit
   Test intent: Given a PromptReady payload with a nonce, a prompt generation, `line_empty` true, `prompt_suppressed` true, and a cwd, When it is parsed, Then those five values are still present
   Exercise through: `parseSessionFrame` in `src/shared/types/command-session.ts` and the same payload through `session_protocol::frame`
   Test doubles: none
   Expected RED: the parser drops prompt generation, `line_empty`, `prompt_suppressed`, or cwd
8. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "prompt frame keeps its fields"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::prompt_frame_keeps_its_fields`
9. Keep those five values on the TypeScript parser and on `frame.rs`. Also accept a PromptReady that omits any of them, so the next test fails. Re-run step 8 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::prompt_frame_keeps_its_fields`. Verify PASS. Commit: `git commit -m "feat(shared): keep the prompt frame fields"`
10. Write failing test for: a prompt frame requires its fields
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given a PromptReady payload that omits the nonce, the prompt generation, `line_empty`, `prompt_suppressed`, or cwd, When it is parsed, Then it is rejected
    Exercise through: `parseSessionFrame` and `session_protocol::frame`
    Test doubles: none
    Expected RED: the parser still accepts that payload as a prompt frame
11. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "prompt frame requires its fields"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::prompt_frame_requires_its_fields`
12. Reject a PromptReady that omits any of those five fields. Also reject a payload whose `prompt_suppressed` or `line_empty` is false, so the next test fails. Re-run step 11 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::prompt_frame_requires_its_fields`. Verify PASS. Commit: `git commit -m "feat(shared): reject a prompt frame that omits a field"`
13. Write failing test for: a false prompt flag still parses
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given a PromptReady payload with `prompt_suppressed` false, `line_empty` false, and its nonce, prompt generation, and cwd present, When it is parsed, Then the frame is accepted and both booleans stay false
    Exercise through: `parseSessionFrame` and `session_protocol::frame`
    Test doubles: none
    Expected RED: the parser rejects that payload
14. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "false prompt flag still parses"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::false_prompt_flag_still_parses`
15. Parse those false values and keep them on the frame. Do not decide raw mode in the parser. Re-run step 14 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::false_prompt_flag_still_parses`. Verify PASS. Commit: `git commit -m "feat(shared): parse a prompt frame whose flags are false"`
16. Write failing test for: command finished keeps its exit code
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given CommandFinished with exit code 2, When it is parsed, Then the code is 2
    Exercise through: `parseSessionFrame` and `session_protocol::frame`
    Test doubles: none
    Expected RED: the exit code is dropped
17. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "command finished keeps its exit code"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::command_finished_keeps_its_exit_code`
18. Store that code on the TypeScript parser and on `frame.rs`. Also store 0 when the code is absent, so the next test fails. Re-run step 17 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::command_finished_keeps_its_exit_code`. Verify PASS. Commit: `git commit -m "feat(shared): keep the command finished exit code"`
19. Write failing test for: command finished may omit its exit code
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given CommandFinished with no exit code, When it is parsed, Then the frame is accepted and the code is absent
    Exercise through: `parseSessionFrame` and `session_protocol::frame`
    Test doubles: none
    Expected RED: the missing code becomes 0
20. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "command finished may omit its exit code"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::command_finished_may_omit_its_exit_code`
21. Leave the code absent. Re-run step 20 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::command_finished_may_omit_its_exit_code`. Verify PASS. Commit: `git commit -m "feat(shared): allow a command finished frame without an exit code"`
22. Write failing test for: init shell keeps its registration
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given InitShell with session_id "s1" and nonce "n1", When it is parsed, Then both values are present
    Exercise through: `parseSessionFrame` and `session_protocol::frame`
    Test doubles: none
    Expected RED: `session_id` or the nonce is dropped
23. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "init shell keeps its registration"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::init_shell_keeps_its_registration`
24. Keep both values on the TypeScript parser and on `frame.rs`. Also accept an InitShell that omits `session_id`, so the next test fails. Re-run step 23 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::init_shell_keeps_its_registration`. Verify PASS. Commit: `git commit -m "feat(shared): keep the init shell registration"`
25. Write failing test for: init shell requires its registration
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given InitShell that omits `session_id` or the nonce, When it is parsed, Then it is rejected
    Exercise through: `parseSessionFrame` and `session_protocol::frame`
    Test doubles: none
    Expected RED: the parser still accepts that payload as InitShell
26. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "init shell requires its registration"` and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::init_shell_requires_its_registration`
27. Reject an InitShell that omits `session_id` or the nonce. Re-run step 26 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::init_shell_requires_its_registration`. Verify PASS. Commit: `git commit -m "feat(shared): reject an init shell frame that omits a field"`
28. Write failing test for: a nonce mismatch is not a mode signal
   Test file: `src/shared/types/command-session.test.ts`
   Level: unit
   Test intent: Given a prompt frame whose nonce does not match the registered session, When it is parsed, Then it is not a mode signal
   Exercise through: `parseSessionFrame`
   Test doubles: none
   Expected RED: the parser still returns that prompt as a mode signal
29. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "nonce mismatch is not a mode signal"`
30. Reject the mismatched nonce. Mirror the same rejection in `frame.rs` and `tests.rs`. Re-run step 29 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::nonce_mismatch_is_not_a_mode_signal`. Verify PASS. Commit: `git commit -m "feat(shared): reject a session frame with the wrong nonce"`
31. Write failing test for: RPC names on TerminalApi
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given the TerminalApi type, When its keys are listed, Then `submitCommand`, `requestCompletion`, and `cancelCompletion` exist and `write` remains the keystroke method
    Exercise through: a type-level fixture that imports `TerminalApi` from `src/shared/types/ipc.types.ts`
    Test doubles: none
    Expected RED: those three methods are not on the interface, so the fixture does not typecheck
32. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "TerminalApi session methods"`
33. Add the three methods to `TerminalApi` only. Declare `mod session_protocol` from `src-tauri/src/pty/mod.rs`. Re-run step 32 and the Rust ident test. Verify PASS. Commit: `git commit -m "feat(ipc): name the submit and completion RPCs"`
34. Write failing test for: one clear-line sequence
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given the shared contract, When `CLEAR_LINE_SEQUENCE` is read, Then it is the single byte string `\u001b[99;1~`, it contains no command text, and the Rust constant in `session_protocol` is the same bytes
    Exercise through: `CLEAR_LINE_SEQUENCE` in `command-session.ts` and `session_protocol::CLEAR_LINE_SEQUENCE`
    Test doubles: none
    Expected RED: the constant does not exist on either side
35. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "one clear-line sequence"`
36. Publish that exact sequence from the TypeScript contract and the Rust module. Re-run step 35 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::clear_line_sequence_matches`. Verify PASS. Commit: `git commit -m "feat(shared): define the clear-line sequence"`
37. Write failing test for: submit result names
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given the shared submit and completion results, When they are serialized, Then the names are exactly `Accepted`, `PreWrite`, `Partial`, and `NotStarted`
    Exercise through: the result enum in `command-session.ts` and `session_protocol::frame`
    Test doubles: none
    Expected RED: the result enum does not exist
38. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "submit result names"`
39. Add the enum on both sides with those serde names. Re-run step 38 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::submit_result_names`. Verify PASS. Commit: `git commit -m "feat(shared): name the submit and completion results"`
40. Write failing test for: the restore-prompt sequence
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given the shared contract, When `RESTORE_PROMPT_SEQUENCE` is read, Then it is `\u001b[99;2~`, it differs from `CLEAR_LINE_SEQUENCE`, and the Rust constant is the same bytes
    Exercise through: `RESTORE_PROMPT_SEQUENCE` in `command-session.ts` and `session_protocol`
    Test doubles: none
    Expected RED: the constant does not exist
41. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "restore-prompt sequence"`
42. Publish the sequence on both sides. Re-run step 41 and `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::tests::restore_prompt_sequence_matches`. Verify PASS. Commit: `git commit -m "feat(shared): define the restore-prompt sequence"`
43. Write failing test for: restorePrompt is a TerminalApi method
    Test file: `src/shared/types/command-session.test.ts`
    Level: unit
    Test intent: Given TerminalApi, When its keys are listed, Then `restorePrompt` exists beside the three session methods and it is not `write`
    Exercise through: the type-level fixture from step 31
    Test doubles: none
    Expected RED: `restorePrompt` is not on the interface
44. Run test — verify FAIL: `bun run test -- src/shared/types/command-session.test.ts -t "restorePrompt is a TerminalApi method"`
45. Add that method. Re-run step 44 and verify PASS. Refactor while green. Commit: `git commit -m "feat(ipc): name the restore-prompt RPC"`

## REFERENCES LOADED

docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md — Design Decision, protocol shape, invariants 1 and 3
src/shared/types/ipc.types.ts — `TerminalApi` at line 393
src-tauri/src/pty/mod.rs — module list
Installed `@xterm/xterm@6.1.0-beta.287` typings and Context7 `/websites/xtermjs` (2026-10-20) — OSC handling is not used as the stripper. The ident must not be 133 or 633.

## WHY THIS APPROACH

Complexity: standard
Justification: Rust and both clients share one frame. Putting the contract in a prerequisite stops later tasks from inventing different fields.

## SANDWICH CONTEXT

[CRITICAL: Do not split an existing file. Add only the TerminalApi methods and the pty module declaration.]
You are implementing the session protocol contract for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A, a Termul session protocol with in-band completion.
Files in scope: the Create and Modify paths for this task in the file map.
Available after: none (prereq)
Architecture rule: `src/shared/` stays runtime-neutral. The parser does not write the PTY.
[RESTATE: Do not split an existing file. Add only the TerminalApi methods and the pty module declaration.]

## DELIVERABLE

Given the protocol ident, When it is read, Then it is 9477 and not 133, 633, or 9278
Given a completion result missing any validity field, When it is parsed, Then it is rejected
Given PromptReady carries a nonce, prompt generation, `line_empty`, `prompt_suppressed`, and cwd, When it is parsed, Then those values remain
Given PromptReady omits any of those five fields, When it is parsed, Then it is rejected
Given PromptReady has `prompt_suppressed` false and `line_empty` false, and its other fields are present, When it is parsed, Then the frame is accepted and both booleans stay false
Given CommandFinished carries exit code 2, When it is parsed, Then the code is 2
Given CommandFinished carries no exit code, When it is parsed, Then the frame is accepted and no code is invented
Given InitShell carries `session_id` and a nonce, When it is parsed, Then both remain
Given InitShell omits `session_id` or the nonce, When it is parsed, Then it is rejected
Given a frame whose nonce does not match, When it is parsed, Then it is not a mode signal
Given TerminalApi, When its methods are listed, Then submit, completion start, and completion cancel are present beside `write`
Given the shared contract, When the clear-line sequence is read, Then TypeScript and Rust export `\u001b[99;1~`
Given the shared contract, When the restore-prompt sequence is read, Then TypeScript and Rust export `\u001b[99;2~` and it differs from the clear-line sequence
Given TerminalApi, When its methods are listed, Then `restorePrompt` is present and it is not `write`
Given a submit or completion result, When it is serialized, Then its name is `Accepted`, `PreWrite`, `Partial`, or `NotStarted`

## QUALITY BAR

Must-have:
  - One ident constant used by later Rust tasks
  - Frame names from the spec, including the PromptReady, InitShell, and CommandFinished fields
  - Tests live in a sibling test file, not an inline test body

Must-not-have:
  - Shell scripts, editor UI, or a call to `PtyManager::write`
  - Output blocks, Warp source, or a split of an existing file

Open question risks:
  - OSC 9477 collides in the field → keep the constant in this module so one change replaces it. Do not reopen the forbidden idents.

Rollback note:
  - Removing the module and the session interface methods leaves today's `write` path unchanged.

Red flags:
  - Work outside listed files → DONE_WITH_CONCERNS
  - A mode change on a nonce mismatch → STOP

## STOP CONDITIONS

Done when: every test command named in this task passes and no out-of-scope file was reorganized
Uncertain when: the ident constant cannot live in one module
Escalate when: the task needs to split `ipc.types.ts` or `pty/mod.rs`

---

### Task 2: Strip session protocol bytes in the PTY reader [depends: T1] [parallel: T3]

## OBJECTIVE

Remove protocol frames from the byte stream that clients receive, including frames split across reads. Deliver the parsed frame to an in-process listener. Leave ordinary output, including ambient OSC 133, in the stream.

Steps:
1. Write failing test for: a complete frame is removed
   Test file: `src-tauri/src/pty/session_protocol/filter_tests.rs`
   Level: unit
   Test intent: Given output bytes that contain text plus one complete OSC 9477 frame, When the filter runs, Then the forwarded bytes are the text only and the listener receives one parsed frame
   Exercise through: `SessionFilter::process`
   Test doubles: none. The listener is the filter's return value
   Expected RED: `SessionFilter` does not exist
2. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::filter_tests::complete_frame_is_removed`
3. Implement the filter. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(pty): strip a complete session protocol frame"`
4. Write failing test for: a frame split across chunks
   Test file: `src-tauri/src/pty/session_protocol/filter_tests.rs`
   Level: unit
   Test intent: Given a frame whose bytes arrive in two reads, When both reads are processed, Then neither forwarded chunk contains a protocol byte and the listener receives the frame once, after the second read
   Exercise through: two `SessionFilter::process` calls on the same filter
   Test doubles: none
   Expected RED: the split bytes are forwarded because the filter does not hold a partial frame
5. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::filter_tests::split_frame_is_not_forwarded`
6. Hold a partial frame in the filter. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(pty): hold a session frame split across reads"`
7. Write failing test for: a malformed frame is dropped
   Test file: `src-tauri/src/pty/session_protocol/filter_tests.rs`
   Level: unit
   Test intent: Given a truncated 9477 payload that can be abandoned without losing sync, When the filter finishes the sequence, Then no frame is emitted and the payload is not forwarded
   Exercise through: `SessionFilter::process`
   Test doubles: none
   Expected RED: malformed 9477 bytes are still forwarded
8. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::filter_tests::malformed_frame_is_dropped`
9. Drop that 9477 payload by discarding every OSC sequence, including OSC 133. Do not call `SessionGate`. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(pty): drop a malformed session frame"`
10. Write failing test for: ambient terminal bytes stay in the stream
    Test file: `src-tauri/src/pty/session_protocol/filter_tests.rs`
    Level: unit
    Test intent: Given OSC 133 and `__TERMUL_EXIT__` text, When the filter runs, Then those bytes are forwarded unchanged and no session frame is emitted
    Exercise through: `SessionFilter::process`
    Test doubles: none
    Expected RED: the filter drops those non-9477 bytes
11. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::filter_tests::ambient_bytes_are_forwarded`
12. Forward every byte that is not a 9477 frame. In `reader_loop`, call the filter after the exit-code parse and before the pending-buffer push, beside the existing `DaFilter` call. Do not edit `DaFilter`. The filter returns parsed frames to its caller. It does not call `SessionGate`. Re-run steps 2, 5, 8, and 11. Verify PASS. Commit: `git commit -m "feat(pty): keep ambient terminal bytes in the client stream"`
13. Write failing test for: an unsynchronizable stream reports desync
    Test file: `src-tauri/src/pty/session_protocol/filter_tests.rs`
    Level: unit
    Test intent: Given a 9477 frame that exceeds the hold limit or cannot be resynced, When the filter finishes it, Then the result is `Desync`, no payload is forwarded, and no session frame is emitted
    Exercise through: `SessionFilter::process`
    Test doubles: none
    Expected RED: the oversized frame is only dropped, with no `Desync` result
14. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::filter_tests::unsynchronizable_stream_reports_desync`
15. Return `Desync` from the filter. Re-run step 14 and verify PASS. Refactor while green. Commit: `git commit -m "feat(pty): report a session stream that cannot be resynced"`

## REFERENCES LOADED

Spec — protocol isolation, fail-safe malformed frames
src-tauri/src/pty/manager/spawn.rs — `reader_loop` lines 410–454
src-tauri/src/pty/da_filter.rs — pass-through is the wrong pattern for protocol bytes
`@xterm/xterm@6.1.0-beta.287` typings — OSC callback runs only after the sequence finishes and stores up to 10 MB. That is why the host filter, not `registerOscHandler`, owns stripping.

## WHY THIS APPROACH

Complexity: standard
Justification: The reader already filters bytes before they reach desktop and web clients. A separate filter keeps `DaFilter` behavior intact.

## SANDWICH CONTEXT

[CRITICAL: Drop protocol bytes from the forwarded stream. Do not extend DaFilter and do not split spawn.rs.]
You are implementing the host byte filter for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. The host strips the private channel.
Files in scope: filter.rs, filter_tests.rs, and the `reader_loop` call in spawn.rs.
Available after: T1 frame parser and ident
Architecture rule: shared-live subscribers read the same forwarded bytes, so stripping has to happen here.
[RESTATE: Drop protocol bytes from the forwarded stream. Do not extend DaFilter and do not split spawn.rs.]

## DELIVERABLE

Given text plus a complete 9477 frame, When the filter runs, Then only the text is forwarded and one frame is parsed
Given a frame split across reads, When both reads finish, Then the frame is parsed once and neither read forwards protocol bytes
Given a truncated 9477 payload that can be abandoned, When the filter finishes it, Then no frame is emitted and that payload is not forwarded
Given OSC 133 or `__TERMUL_EXIT__`, When the filter runs, Then those bytes stay in the stream
Given a 9477 frame that cannot be resynced, When the filter finishes it, Then the result is `Desync` and the payload is not forwarded
[must-not] Given protocol bytes, When they are filtered, Then they are not left for xterm to hide

## QUALITY BAR

Must-have:
  - The filter is invoked from `reader_loop` before `pending_buf`
  - `DaFilter` still runs and still answers DA

Must-not-have:
  - A renderer OSC handler as the only stripper
  - A rewrite of `reader_loop` beyond the filter call
  - Logging the frame payload

Open question risks:
  - A frame larger than the hold limit returns `Desync` from this filter. T5 turns that result into raw mode. This task does not call the session gate.

Rollback note:
  - Skipping the filter call restores today's byte stream. The prompt stays visible because no integration is required for this task to roll back.

Red flags:
  - Forwarding a 9477 payload → STOP
  - Editing `da_filter.rs` behavior → STOP

## STOP CONDITIONS

Done when: every cargo filter named in this task passes and a manual read of `reader_loop` shows one new call
Uncertain when: a partial frame cannot be distinguished from ordinary ESC text
Escalate when: the call cannot be added without reorganizing `spawn.rs`

---

### Task 3: Serialize submit and completion on the host [depends: T1] [test-risk]

## OBJECTIVE

Add the host RPCs that submit a draft once and run at most one in-band completion. Enforce write authority, pre-write versus partial failure, and the generator lock.

Steps:
1. Write failing test for: one accepted submit write
   Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
   Level: unit
   Test intent: Given a trusted AtPrompt session and draft "git sta", When submit is accepted, Then the captured writer receives `CLEAR_LINE_SEQUENCE` from T1 and then "git sta" exactly once
   Exercise through: `SessionGate::submit`
   Test doubles: an in-memory PTY writer. Do not mock `SessionGate`
   Expected RED: `SessionGate` does not exist
2. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::submit_writes_the_draft_once`
3. Implement the accepted path. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(pty): submit a draft through the session gate"`
4. Write failing test for: pre-write failure restores nothing on the host
   Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
   Level: unit
   Test intent: Given no PTY writer, When submit is called, Then the result is `PreWrite` and the writer captured zero bytes
   Exercise through: `SessionGate::submit`
   Test doubles: a missing writer
   Expected RED: the missing writer is reported as a generic write error after a write attempt
5. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::missing_writer_is_pre_write`
6. Return `PreWrite` before any write. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(pty): report a submit that never reached the PTY"`
7. Write failing test for: partial write
   Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
   Level: unit
   Test intent: Given a writer whose `write_all` or `flush` returns an error, When submit is called, Then the result is `Partial` and it is not `Accepted`
   Exercise through: `SessionGate::submit`
   Test doubles: a writer that fails on `write_all` or `flush`
   Expected RED: a write error is indistinguishable from `PreWrite`
8. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::write_error_is_partial`
9. Map `write_all` and `flush` errors to `Partial`. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(pty): keep a partial submit out of the editor"`
10. Write failing test for: one generator
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a generator is active, When a second completion is requested, Then it is `NotStarted` and the writer receives no second generator command
    Exercise through: `SessionGate::request_completion`
    Test doubles: an in-memory writer
    Expected RED: `request_completion` does not exist, so the test does not compile
11. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::one_generator`
12. Hold one generator slot. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(pty): allow one completion generator per PTY"`
13. Write failing test for: a viewer is refused
    Test file: `src-tauri/src/web/terminal_ws/command_editor_tests.rs`
    Level: unit
    Test intent: Given the session RPC names, When `terminal_request_is_view` is called for each, Then every name returns false. Given `allow_write` is false, When `SessionGate::request_completion` or `submit` is called, Then the writer receives nothing
    Exercise through: `terminal_request_is_view` in `src-tauri/src/web/capabilities.rs` and `SessionGate`
    Test doubles: an in-memory writer. Do not mock `terminal_request_is_view` or `SessionGate`
    Expected RED: the RPC name constant does not exist, so the test does not compile
14. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib terminal_ws::command_editor_tests::session_rpcs_are_not_view_requests`
15. Add the names in the new command module. Do not add them to the view-request match. Pass `ctx.capabilities.write` into the gate from the new `terminal_ws.rs` arms. The existing view-only check above the match then returns FORBIDDEN. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(pty): refuse session RPCs from a view-only connection"`
16. Write failing test for: an unrestorable provider is not started
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a completion provider whose wrapper cannot prove CWD, env, history, and shell options are restored, When completion is requested, Then that provider is not started and the writer receives no generator command. Another provider that can prove the restore may still run
    Exercise through: `SessionGate::request_completion`
    Test doubles: two provider descriptors, one marked unrestorable
    Expected RED: `request_completion` has no provider argument, so the test does not compile
17. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::unrestorable_provider_is_not_started`
18. Add the provider argument. Skip the unrestorable provider and still start one that can prove restore. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(pty): skip a completion provider that cannot restore state"`
19. Write failing test for: the generator slot frees after it finishes
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a generator has finished or been cancelled, When another completion is requested, Then one new generator command is written
    Exercise through: `SessionGate::request_completion`
    Test doubles: an in-memory writer
    Expected RED: the second request stays `NotStarted` after the first generator has finished
20. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::generator_slot_frees_after_finish`
21. Release the slot when the generator finishes or is cancelled. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(pty): release the completion generator slot"`
22. Write failing test for: a completion result is not broadcast
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a completion result for the caller, When the gate finishes it, Then the RPC waiter receives it and a `TerminalEventHub` subscriber receives no completion payload
    Exercise through: `SessionGate` and `TerminalEventHub`
    Test doubles: an in-memory hub subscriber. Do not mock `SessionGate` or the hub
    Expected RED: the completion return path does not exist, so the test does not compile
23. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::completion_result_is_not_broadcast`
24. Return the active result only to its RPC waiter. Do not publish it on `TerminalEventHub`. Also return a finished request's payload as the active waiter's result, so the next test fails. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(pty): return a completion result to its caller"`
25. Write failing test for: a finished request id is not reused
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given request A has finished and request B is the active request, When a payload for A arrives, Then it is not returned as B's result
    Exercise through: `SessionGate`
    Test doubles: an in-memory writer. Do not mock `SessionGate`
    Expected RED: A's payload is returned to the waiter for B
26. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::finished_request_id_is_not_reused`
27. Ignore a payload for a finished request. Keep returning the active request's result only to its RPC waiter, and do not publish it on the hub. Re-run steps 23 and 26 and verify PASS. Commit: `git commit -m "feat(pty): ignore a completion result for a finished request"`
28. Write failing test for: a failed restore drops trust
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a generator was started and the wrapper then reports that CWD, env, or shell options were not restored, When the gate observes that, Then the session is raw and the draft is not written
    Exercise through: `SessionGate`
    Test doubles: a wrapper report. Do not mock `SessionGate`
    Expected RED: the gate still returns a completion result and stays AtPrompt
29. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::failed_restore_drops_trust`
30. Mark the session raw and do not write the draft. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(pty): drop trust when a completion cannot restore state"`
31. Write failing test for: submit drains the generator first
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a generator is active, When submit arrives, Then the generator is cancelled or drained before the draft bytes, and the draft is written once
    Exercise through: `SessionGate::submit`
    Test doubles: an in-memory writer that records order
    Expected RED: the draft bytes are written while the generator command is still the active write
32. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::submit_drains_the_generator_first`
33. Cancel or drain, then write the draft, under the same writer lock as `PtyManager::write`. Expose the RPCs from new command modules and add match arms in `lib.rs` and `terminal_ws.rs`. The invoke names and websocket `type` values are `command_editor_submit`, `command_editor_request_completion`, and `command_editor_cancel_completion`. Each arm returns before it calls `SessionGate`, so the dispatch test fails. Do not register them on the ledgered `write` path. Re-run step 32 and verify PASS. Commit: `git commit -m "feat(pty): expose submit and completion RPCs"`
34. Write failing test for: outcome logs omit the draft
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a draft "git status", When submit returns `PreWrite`, `Partial`, or `NotStarted`, Then one log line names that result and the line does not contain "git status"
    Exercise through: `SessionGate`
    Test doubles: a log recorder. Do not mock `SessionGate`
    Expected RED: no log line names the result
35. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::outcome_logs_omit_the_draft`
36. Log the result name only. Re-run step 35 and verify PASS. Commit: `git commit -m "feat(pty): log a submit outcome without the draft"`
37. Write failing test for: restore prompt writes the sequence
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a session whose draft is "git status", When `SessionGate::restore_prompt` runs with write permission, Then the writer receives `RESTORE_PROMPT_SEQUENCE` and does not receive "git status". The invoke and websocket type name is `command_editor_restore_prompt`
    Exercise through: `SessionGate::restore_prompt`
    Test doubles: an in-memory writer. Do not mock `SessionGate`
    Expected RED: `restore_prompt` does not exist, so the test does not compile
38. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::restore_prompt_writes_the_sequence`
39. Add the gate method and the `command_editor_restore_prompt` arm. That arm returns before it calls `SessionGate`, so the dispatch test fails. Also write the sequence when `allow_write` is false, so the next test fails. Re-run step 38 and verify PASS. Commit: `git commit -m "feat(pty): restore the shell prompt without the draft"`
40. Write failing test for: a viewer cannot restore the prompt
    Test file: `src-tauri/src/web/terminal_ws/command_editor_tests.rs`
    Level: unit
    Test intent: Given `allow_write` is false, When `restore_prompt` is requested, Then the writer receives nothing and `terminal_request_is_view("command_editor_restore_prompt")` is false
    Exercise through: `terminal_request_is_view` and `SessionGate::restore_prompt`
    Test doubles: an in-memory writer. Do not mock `SessionGate`
    Expected RED: the view-only caller still receives the restore sequence
41. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib terminal_ws::command_editor_tests::viewer_cannot_restore_the_prompt`
42. Refuse the view-only call. Do not add the name to the view-request match. Re-run steps 14 and 41. Verify PASS. Commit: `git commit -m "feat(pty): refuse restore-prompt from a view-only connection"`
43. Write failing test for: the generator slot is visible
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: unit
    Test intent: Given a completion generator has started, When `SessionGate::generator_active` is read, Then it is true. Given that generator has finished or been cancelled, When it is read again, Then it is false
    Exercise through: `SessionGate::generator_active`
    Test doubles: an in-memory writer. Do not mock `SessionGate`
    Expected RED: `generator_active` does not exist, so the test does not compile
44. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::generator_slot_is_visible`
45. Return true only while the slot is held. Re-run step 44 and verify PASS. Commit: `git commit -m "feat(pty): report whether a completion generator is active"`
46. Write failing test for: session RPC names reach the gate
    Test file: `src-tauri/src/web/terminal_ws/command_editor_tests.rs`
    Level: integration
    Test intent: Given the invoke handler, the websocket match, and a trusted AtPrompt session with write permission and draft "git sta", When `command_editor_submit`, `command_editor_request_completion`, `command_editor_cancel_completion`, and `command_editor_restore_prompt` are each dispatched, Then the real `SessionGate` writes one clear sequence plus the draft, starts one generator, cancels that generator, and writes one restore sequence that does not contain the draft. `request_completion` is dispatched while that prompt is trusted and the generator slot is free. A raw gate is not this session
    Exercise through: the invoke handler and the websocket match, to the real `SessionGate`
    Test doubles: an in-memory writer. Do not mock `SessionGate` or the match arms
    Expected RED: the arms return before calling `SessionGate`, so the writer receives nothing
47. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib terminal_ws::command_editor_tests::session_rpc_names_reach_the_gate`
48. Call `SessionGate` from each arm. Do not register the names on the ledgered `write` path and do not add them to `terminal_request_is_view`. When `request_completion` runs on a raw gate that still has write permission, still write a generator command, so the next test fails. Re-run steps 14, 41, and 47 and verify PASS. Commit: `git commit -m "feat(pty): dispatch session RPCs to the session gate"`
49. Write failing test for: completion while raw is not started
    Test file: `src-tauri/src/pty/session_protocol/gate_tests.rs`
    Level: integration
    Test intent: Given the gate is raw after a failed restore, When `request_completion` is called, Then the result is `NotStarted` and the writer receives no generator command
    Exercise through: `SessionGate::request_completion`
    Test doubles: an in-memory writer. Do not mock `SessionGate`
    Expected RED: the raw gate still writes a generator command
50. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::gate_tests::completion_while_raw_is_not_started`
51. Return `NotStarted` when the gate is not a trusted AtPrompt, and write no generator command. A trusted AtPrompt with a free slot still starts one generator. Re-run steps 11, 29, 47, and 50 and verify PASS. Refactor while green. Commit: `git commit -m "feat(pty): refuse completion when the prompt is not trusted"`

## REFERENCES LOADED

Spec — submit once, serialized invariant, owner-only invariant, state-preserving wrapper contract
src-tauri/src/pty/manager/lifecycle.rs — `write_all` and `flush`, lines 5–26
src-tauri/src/remote/host.rs — `allow_remote_writes: false` at line 382
src-tauri/src/lib.rs — invoke handler around line 1816
src-tauri/src/web/terminal_ws.rs — `"write"` arm at line 463. The view-only check is the `ctx.capabilities.write` gate at lines 399–410.
src-tauri/src/web/capabilities.rs — `terminal_request_is_view` at line 280. New RPC names stay out of that match.
src-tauri/src/web/terminal_ws/commands.rs — ledgered write. Do not reuse it.

## WHY THIS APPROACH

Complexity: standard
Justification: The writer lock, the authority check, and the partial-write result are one state machine. Splitting them would let a client submit through a path that does not see the generator.

## SANDWICH CONTEXT

[CRITICAL: A completion must not overlap a submit, and a viewer must not write.]
You are implementing the host session gate for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. One in-band generator on the session PTY.
Files in scope: gate.rs, gate_tests.rs, commands/command_editor.rs, web/terminal_ws/command_editor.rs, plus the invoke and match wiring.
Available after: T1
Architecture rule: completion output is the RPC result. Do not publish it on TerminalEventHub.
[RESTATE: A completion must not overlap a submit, and a viewer must not write.]

## DELIVERABLE

Given draft "git sta" at a trusted prompt, When submit is accepted, Then the clear sequence and "git sta" are written once
Given no writer, When submit is called, Then the result is PreWrite and zero bytes were written
Given write_all or flush fails, When submit is called, Then the result is Partial and not Accepted
Given an active generator, When another completion is requested, Then it is not started
[must-not] Given a view-only caller, When completion or submit is requested, Then the host writes nothing
Given an active generator, When submit arrives, Then the generator is drained or cancelled before the one draft write
Given a provider whose wrapper cannot prove restore, When completion is requested, Then that provider is not started
Given a generator has finished, When another completion is requested, Then one new generator starts
[must-not] Given a completion result, When the gate finishes it, Then `TerminalEventHub` does not receive that payload
Given a restore check fails after the generator starts, When the gate observes it, Then the session is raw and the draft is not written
Given a generator has started, When `generator_active` is read, Then it is true until that generator finishes or is cancelled
Given a draft, When submit returns `PreWrite`, `Partial`, or `NotStarted`, Then the log names that result and does not contain the draft
Given a draft and write permission, When `restore_prompt` runs, Then the writer receives the restore sequence and not the draft
Given a trusted AtPrompt session with write permission and draft "git sta", When each session RPC name is dispatched through the invoke handler and the websocket match, Then the real `SessionGate` performs that operation and a restore sequence does not contain the draft
Given the gate is raw after a failed restore, When completion is requested, Then the result is `NotStarted` and the writer receives no generator command
[must-not] Given a view-only caller, When `restore_prompt` is requested, Then the host writes nothing

## QUALITY BAR

Must-have:
  - The wrapper refuses a provider that cannot prove CWD, env, history, and shell options are restored. That provider is not started
  - A failed restore that leaves the session untrusted marks the session raw and does not write the draft
  - Logs name `PreWrite`, `Partial`, or `NotStarted` and omit the draft

Must-not-have:
  - Ledgered replay of submit or completion
  - A second PTY
  - History storage on the host

Open question risks:
  - The wrapper cannot contain a completer → do not start that provider. If trust is lost, return a raw-fallback result and do not execute the draft.

Rollback note:
  - The new RPCs can remain unused. Today's `terminal_write` path stays as it is.

Red flags:
  - Submit while a generator is still writing → STOP
  - Viewer write succeeds → STOP

## STOP CONDITIONS

Done when: every cargo filter named in this task passes
Uncertain when: `write_all` cannot be classified as partial versus pre-write with the in-memory writer
Escalate when: the only way to hook the writer is to rewrite `lifecycle.rs`

---

### Task 4: Inject bash and zsh integration [depends: T1, T2] [test-risk]

## OBJECTIVE

Install original bash and zsh integration for a local interactive login shell. Every other program stays raw. The scripts speak the T1 frames and keep a generator out of the user prompt cycle.

Steps:
1. Write failing test for: bash and zsh are injected
   Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
   Level: unit
   Test intent: Given bash or zsh with a login arg and no custom program, When the injection plan is built, Then it includes the matching script and a fresh session id and nonce
   Exercise through: `injection_plan`
   Test doubles: none
   Expected RED: `injection_plan` does not exist
2. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::bash_and_zsh_are_injected`
3. Implement `injection_plan` so every program, including fish, dash, ssh, PowerShell, cmd, and a direct program, receives a script. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(pty): choose bash and zsh for session integration"`
4. Write failing test for: other programs stay raw
   Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
   Level: unit
   Test intent: Given fish, dash, ssh, PowerShell, cmd, or a direct program, When the injection plan is built, Then it is raw and adds no script
   Exercise through: `injection_plan`
   Test doubles: none
   Expected RED: those programs still receive a script
5. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::other_programs_stay_raw`
6. Return raw for those programs. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(pty): leave non-bash shells on the raw path"`
7. Write failing test for: a generator is not a user command
   Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
   Level: unit
   Test intent: Given the bash and zsh script sources, When they are scanned, Then a generator path sets an internal flag and does not emit user `CommandStarted`, and prompt frames carry the nonce and `prompt_suppressed`
   Exercise through: reading `src-tauri/resources/shell-integration/bash.sh` and `zsh.sh`
   Test doubles: none
   Expected RED: the script files do not exist
8. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::generator_is_not_a_user_command`
9. Write original scripts that flag the generator and emit the prompt fields. Do not add the clear-line binding or the bootstrap-failure path in this step. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(pty): add bash and zsh session integration scripts"`
10. Write failing test for: bootstrap failure leaves the shell command unchanged
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the script sources, When the bootstrap-failure path is scanned, Then it does not replace the user's shell command
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts have no bootstrap-failure path that preserves the shell command
11. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::bootstrap_failure_keeps_the_shell`
12. Add that failure path. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(pty): keep a normal shell when integration cannot start"`
13. Write failing test for: every keymap binds the clear-line sequence
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the bash and zsh scripts, When their keymap bindings are scanned, Then every keymap they declare binds the T1 `CLEAR_LINE_SEQUENCE` and no command text
    Exercise through: reading both script files and `CLEAR_LINE_SEQUENCE`
    Test doubles: none
    Expected RED: the scripts do not bind `\u001b[99;1~`
14. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::clear_line_sequence_is_bound`
15. Bind that sequence on every keymap the scripts declare. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(pty): bind the clear-line sequence in bash and zsh"`
16. Write failing test for: a live bash handshake hides the protocol and reports suppression
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: integration
    Test intent: Given bash is on PATH, When an interactive login bash starts with the injection, Then the first session frame is InitShell for the issued nonce, the following PromptReady has `prompt_suppressed` and `line_empty`, and the bytes after the real T2 `SessionFilter` contain no 9477 payload. If bash is not on PATH, the test fails rather than passing
    Exercise through: a real bash process and `SessionFilter`
    Test doubles: none for bash. Do not mock the filter
    Expected RED: spawned bash emits no InitShell frame
17. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::bash_handshake_reports_suppression -- --nocapture`
18. Wire the plan into `spawn_pty` with a small call. Re-run steps 2, 5, 8, 11, 14, and 17. Verify PASS. If zsh is on PATH, run the same live assertion against zsh in this test module. If zsh is absent, that zsh process assertion is not run and the static zsh scan still is. Commit: `git commit -m "feat(pty): inject session integration into local login shells"`
19. Write failing test for: the visible prompt is zero-width
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the bash and zsh scripts, When they are scanned, Then they save the original prompt and install a zero-width visible prompt while integration is active
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts do not save and replace the visible prompt
20. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::visible_prompt_is_zero_width`
21. Save the original prompt and insert the zero-width assignment before the prompt hook's last statement, so a live shell still shows the control prompt. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(pty): hide the shell prompt while integration is active"`
22. Write failing test for: the restore sequence brings the prompt back
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the scripts, When they are scanned, Then `RESTORE_PROMPT_SEQUENCE` restores the saved prompt and does not contain command text
    Exercise through: reading both script files and `RESTORE_PROMPT_SEQUENCE`
    Test doubles: none
    Expected RED: the scripts do not bind `\u001b[99;2~`
23. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::restore_sequence_brings_the_prompt_back`
24. Bind that sequence. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(pty): restore the shell prompt from the saved value"`
25. Write failing test for: injected bash does not show the control prompt
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: integration
    Test intent: Given bash is on PATH, When one interactive bash starts without injection and another starts with injection, Then the filtered injected stream does not contain the prompt line captured from the control bash. If bash is not on PATH, the test fails rather than passing
    Exercise through: two real bash processes and `SessionFilter`
    Test doubles: none. Do not mock the filter
    Expected RED: the injected stream still contains the control prompt line
26. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::injected_bash_hides_the_control_prompt -- --nocapture`
27. Move the zero-width assignment to the end of the prompt hook. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(pty): keep the injected prompt off the screen"`
28. Write failing test for: a user command emits lifecycle frames
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the bash and zsh scripts, When a user command path is scanned, Then it emits `CommandStarted` before the command and `CommandFinished` with the captured exit code after it, and the generator path emits neither
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts do not emit user `CommandStarted`
29. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::user_command_emits_lifecycle_frames`
30. Add those two emissions on the user path only. Also print completer stdout to the terminal, so the next test fails. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(pty): emit command lifecycle from the user prompt hooks"`
31. Write failing test for: completer output stays off the terminal
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the scripts, When the generator path is scanned, Then completer stdout and stderr are captured into the 9477 result frame and are not written to the terminal
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts print completer output to the terminal
32. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::completer_output_stays_off_the_terminal`
33. Redirect that output into the result frame only. Re-run steps 26, 29, and 32. Verify PASS. Refactor while green. Commit: `git commit -m "feat(pty): keep completer output off the terminal"`

## REFERENCES LOADED

Spec — trusted prompt, v1 shells, prompt suppression, exactly-once echo steps 1 and 2
src-tauri/src/pty/env_refresh.rs — `shell_wants_login_arg`
src-tauri/src/pty/manager/spawn.rs — login arg around line 271
Warp was read only as behavior. No Warp source is copied.

## WHY THIS APPROACH

Complexity: standard
Justification: The shell is the only place that can hide its own prompt and skip Preexec for a generator. A static scan proves the contract markers. One live bash process proves the frame actually appears.

## SANDWICH CONTEXT

[CRITICAL: Inject only local interactive login bash and zsh. Do not copy Warp source.]
You are implementing shell integration for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. The shell cooperates. A failed handshake stays raw.
Files in scope: inject.rs, inject_tests.rs, the two scripts, and the spawn call.
Available after: T1 and T2. The live bash test runs the real `SessionFilter`.
Architecture rule: ambient OSC 133 is not a mode signal.
[RESTATE: Inject only local interactive login bash and zsh. Do not copy Warp source.]

## DELIVERABLE

Given local interactive login bash or zsh, When a PTY is spawned, Then the integration script is installed with a new session id and nonce
Given fish, dash, ssh, PowerShell, cmd, or a direct program, When a PTY is spawned, Then no script is installed
Given the scripts, When a generator runs, Then it does not emit user `CommandStarted`
Given the scripts, When their keymaps are scanned, Then every declared keymap binds the T1 clear-line sequence
Given the scripts, When they are scanned, Then the original prompt is saved and the visible prompt is zero-width
Given the scripts, When they are scanned, Then the restore sequence puts the saved prompt back
Given bash on PATH, When it handshakes, Then PromptReady reports suppression and the filtered stream has no 9477 payload
Given bash on PATH, When an uninjected control prompt is compared with the filtered injected stream, Then the injected stream does not contain that prompt line
Given the scripts, When a user command runs, Then the user path emits `CommandStarted` and `CommandFinished` and the generator path emits neither
[must-not] Given a generator runs, When completer stdout or stderr is produced, Then those bytes are not written to the terminal
[must-not] Given bootstrap cannot prove suppression, When the shell starts, Then the spawn stays a normal shell

## QUALITY BAR

Must-have:
  - Scripts are original
  - The prompt hook re-applies suppression at the end
  - The clear-line binding matches the T1 sequence on every keymap the script can identify

Must-not-have:
  - fish `no-mark-prompt` injection in v1
  - A pretyping queue
  - A persisted force-raw setting

Open question risks:
  - zsh is absent in CI → the static zsh scan still runs. The live zsh assertion runs only when zsh is on PATH. Bash absence fails the live test rather than passing.
  - The wrapper is unproven → a provider that cannot restore state is not invoked, as required by T3.

Rollback note:
  - `injection_plan` returning raw for every shell returns startup to today's PTY.

Red flags:
  - A script copied from warpdotdev/warp → STOP
  - Injection into ssh or fish → STOP

## STOP CONDITIONS

Done when: the injector test, the script scan, and the live bash test pass
Uncertain when: bash exists but cannot be started as an interactive login shell in the test harness
Escalate when: spawn cannot take the plan without a rewrite of `spawn.rs`

---

### Task 5: Reduce authenticated shell events to an input mode [depends: T1] [parallel: T3]

## OBJECTIVE

Add a pure reducer from session frames to Editor or PTY. `CompletionPending` stays inside AtPrompt.

Steps:
1. Write failing test for: trusted prompt opens the editor
   Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
   Level: unit
   Test intent: Given a registered session and a PromptReady frame with the nonce, `line_empty`, and `prompt_suppressed`, When it is applied, Then the mode is Editor and the next buffer is empty
   Exercise through: `reduceSession`
   Test doubles: none
   Expected RED: `reduceSession` does not exist
2. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "trusted prompt opens the editor"`
3. Implement that transition. Also treat OSC 133 text as a trusted prompt, so the next test fails. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(terminal): open the editor on a trusted prompt"`
4. Write failing test for: ambient OSC stays raw
   Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
   Level: unit
   Test intent: Given OSC 133 text and no Termul handshake, When it is applied, Then the mode stays raw
   Exercise through: `reduceSession`
   Test doubles: none
   Expected RED: OSC 133 text opens Editor
5. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "ambient OSC stays raw"`
6. Ignore OSC 133. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(terminal): ignore ambient OSC 133"`
7. Write failing test for: a missing nonce stays raw
   Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
   Level: unit
   Test intent: Given a PromptReady frame without this session's nonce, When it is applied, Then the mode stays raw
   Exercise through: `reduceSession`
   Test doubles: none
   Expected RED: a nonce-less frame opens Editor
8. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "missing nonce stays raw"`
9. Ignore that frame. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a prompt with the wrong nonce"`
10. Write failing test for: a visible prompt stays raw
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given PromptReady with the right nonce and `prompt_suppressed` false, When it is applied, Then the mode stays raw
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the unsuppressed prompt opens Editor
11. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "visible prompt stays raw"`
12. Keep that prompt raw. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(terminal): stay raw when the prompt glyph is visible"`
13. Write failing test for: lost integration does not execute the draft
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Editor holds draft "git status", When the session becomes untrusted but the process is still alive, Then the mode is raw, the result says the draft must not be written, the same draft remains stored for a later trusted prompt, and the log event names the fallback and does not contain "git status"
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the lost-integration result says to write the draft
14. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "lost integration does not execute the draft"`
15. Return a do-not-write result and keep the draft on the live session. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(terminal): keep an unsent draft when trust is lost"`
16. Write failing test for: process exit drops the draft
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Editor holds a draft, When the shell process exits, Then the mode is exited and the draft is dropped
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the exited result still keeps the draft
17. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "process exit drops the draft"`
18. Drop the draft only on process exit. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(terminal): drop the draft when the shell exits"`
19. Write failing test for: a duplicate prompt keeps the draft
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Editor already holds an unsent draft, When a second PromptReady arrives, Then the draft and Editor remain
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the second prompt clears the draft
20. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "duplicate prompt keeps the draft"`
21. Ignore that duplicate. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a duplicate prompt"`
22. Write failing test for: completion pending stays at the prompt
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given CompletionPending, When the mode is read, Then it is still AtPrompt and the composer stays mounted
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: CompletionPending reports Executing
23. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "completion pending stays at the prompt"`
24. Keep CompletionPending inside AtPrompt. Also open Editor when CommandFinished arrives, so the next test fails. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(terminal): keep completion inside the prompt state"`
25. Write failing test for: command finished does not open the editor
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Submitting or Executing, When CommandFinished arrives before the matching PromptReady, Then the mode stays out of Editor
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: CommandFinished opens Editor
26. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "command finished does not open the editor"`
27. Leave the mode unchanged. Do not update the exit code in this step. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(terminal): keep the editor closed after a command finishes"`
28. Write failing test for: an exit code is recorded
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given CommandFinished carries exit code 2, When it is applied, Then last exit code becomes 2 and the mode stays out of Editor
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the exit code stays at its previous value
29. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "exit code is recorded"`
30. Store that code. Also clear the stored code when CommandFinished has no code, so the next test fails. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(terminal): record the command exit code"`
31. Write failing test for: a missing exit code keeps the previous code
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given last exit code is 2, When CommandFinished arrives without a code, Then last exit code stays 2
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the missing code clears last exit code
32. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "missing exit code keeps the previous code"`
33. Keep the previous code when the frame has no code. Also clear the code when the same code arrives again, so the duplicate test fails. Re-run step 32 and verify PASS. Commit: `git commit -m "feat(terminal): keep the previous exit code"`
34. Write failing test for: a duplicate exit code does not clear
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given last exit code is 2, When CommandFinished arrives with 2 again, Then last exit code stays 2
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the duplicate code clears last exit code
35. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "duplicate exit code does not clear"`
36. Leave the code in place. Re-run step 35 and verify PASS. Commit: `git commit -m "feat(terminal): keep a repeated exit code"`
37. Write failing test for: a nested prompt stays in submitting
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Submitting, When an intermediate PromptReady arrives, Then the mode stays Submitting
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the intermediate prompt opens Editor
38. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "nested prompt stays in submitting"`
39. Ignore that prompt. Also leave Editor when CommandStarted arrives with no new submit, so the next test fails. Re-run step 38 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a prompt inside a submission"`
40. Write failing test for: command started without a submit is ignored
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Editor and no new submit, When CommandStarted arrives, Then the mode stays Editor and the draft stays
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: CommandStarted leaves Editor
41. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "command started without a submit is ignored"`
42. Ignore that frame. Re-run step 41 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a command start that was not submitted"`
43. Write failing test for: a committed submit leaves the editor
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given the mode is Editor and this session has a committed submit, When CommandStarted arrives, Then the mode leaves Editor
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: CommandStarted leaves the mode in Editor
44. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "committed submit leaves the editor"`
45. Leave Editor only when the session state records a committed submit. CommandStarted with no committed submit still stays in Editor. Do not edit `session-store.ts`. Re-run step 44 and verify PASS. Commit: `git commit -m "feat(terminal): leave the editor when a submitted command starts"`
46. Write failing test for: desync falls back to raw
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Editor holds a draft, When a filter `Desync` is applied, Then the mode is raw and the draft is not marked for execution
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: `Desync` leaves the mode in Editor
47. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "desync falls back to raw"`
48. Map `Desync` to raw. Re-run step 47 and verify PASS. Commit: `git commit -m "feat(terminal): leave the editor when the session stream desyncs"`
49. Write failing test for: a later trusted prompt restores the unsent draft
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given a live process kept draft "make test" after a pre-write failure or a raw fallback, When the next trusted PromptReady arrives on that same session, Then the mode is Editor and the buffer is "make test"
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: that prompt opens an empty buffer
50. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "later trusted prompt restores the unsent draft"`
51. Restore that stored draft. Re-run step 50 and verify PASS. Commit: `git commit -m "feat(terminal): restore an unsent draft on the same live session"`
52. Write failing test for: the next editor after a submit is empty
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given a submit was committed, When the matching PromptReady arrives, Then the mode is Editor and the buffer is empty
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the matching prompt restores the submitted text
53. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "next editor after a submit is empty"`
54. Open an empty buffer. Re-run step 53 and verify PASS. Commit: `git commit -m "feat(terminal): open an empty editor after a submitted command"`
55. Write failing test for: alternate screen enters the PTY
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Editor, When an alternate-screen enter event arrives, Then the mode is PTY
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the alternate screen leaves the mode in Editor
56. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "alternate screen enters the PTY"`
57. Force PTY on alternate-screen enter. Treat leave as opening Editor, so the next test fails. Re-run step 56 and verify PASS. Commit: `git commit -m "feat(terminal): leave the editor when an alternate screen opens"`
58. Write failing test for: leaving the alternate screen waits for a prompt
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given PTY because of an alternate screen, When the leave event arrives without PromptReady, Then the mode stays PTY
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the leave event opens Editor
59. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "leaving the alternate screen waits for a prompt"`
60. Stay on PTY until PromptReady. Re-run step 59 and verify PASS. Commit: `git commit -m "feat(terminal): wait for a prompt after an alternate screen"`
61. Write failing test for: an unregistered session does not change state
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given a frame whose session id is not registered, When it is applied, Then the mode and the draft stay as they were
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: the unregistered frame opens Editor
62. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "unregistered session does not change state"`
63. Ignore that frame. Also open Editor when a time value is supplied while Executing, so the next test fails. Re-run step 62 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a frame for an unregistered session"`
64. Write failing test for: executing does not time out into the editor
    Test file: `src/renderer/lib/command-lifecycle/transition.test.ts`
    Level: unit
    Test intent: Given Executing, When reduce is called with no PromptReady after any elapsed time, Then the mode stays Executing
    Exercise through: `reduceSession`
    Test doubles: none
    Expected RED: a time value changes Executing into Editor
65. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/transition.test.ts -t "executing does not time out into the editor"`
66. Ignore elapsed time. Re-run step 65 and verify PASS. Refactor while green. Commit: `git commit -m "feat(terminal): wait for the shell prompt instead of a timer"`

## REFERENCES LOADED

Spec — Story: Trusted prompt, rules R1, R4, R7, R8, R9, and CompletionPending
src/shared/types/command-session.ts from T1

## WHY THIS APPROACH

Complexity: standard
Justification: The mode rules are a pure function. UI and the host should not each invent a transition.

## SANDWICH CONTEXT

[CRITICAL: Only a nonce-matched suppressed prompt mounts the editor.]
You are implementing the session reducer for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. CompletionPending is not Executing.
Files in scope: transition.ts and transition.test.ts
Available after: T1
Architecture rule: the reducer does not write the PTY and does not render a block.
[RESTATE: Only a nonce-matched suppressed prompt mounts the editor.]

## DELIVERABLE

Given an authenticated suppressed empty prompt, When it is applied, Then the mode is Editor and the next buffer is empty
Given ambient OSC 133, When it arrives, Then the mode stays raw
Given a prompt without this session's nonce, When it arrives, Then the mode stays raw
Given `prompt_suppressed` is false, When PromptReady arrives, Then the mode stays raw
[must-not] Given integration is lost while a draft exists on a live process, When the session falls back, Then the draft is not marked for execution and the log omits the draft
Given the shell process exits, When the session ends, Then the draft is dropped
Given a second prompt while a draft is open, When it arrives, Then the draft stays
Given CompletionPending, When the mode is read, Then the composer is still allowed and the session is not Executing
Given CommandFinished before the matching prompt, When it arrives, Then the editor stays closed
Given CommandFinished carries a code, When it arrives, Then last exit code becomes that code
Given CommandFinished carries no code, When it arrives, Then the previous code stays
Given the same exit code again, When it arrives, Then the code is not cleared
Given Submitting, When an intermediate PromptReady arrives, Then the mode stays Submitting
Given Editor and no new submit, When CommandStarted arrives, Then the mode stays Editor
Given a committed submit, When CommandStarted arrives, Then the mode leaves Editor
Given a filter Desync, When it is applied, Then the mode is raw and the draft is not marked for execution
Given a live session still holds an unsent draft, When the next trusted prompt arrives, Then that draft is restored
Given a submit was committed, When the matching prompt arrives, Then the editor opens empty
Given an alternate screen opens, When the event is applied, Then the mode is PTY
Given the alternate screen closes without PromptReady, When the leave event is applied, Then the mode stays PTY
Given a frame for an unregistered session, When it is applied, Then state does not change
Given Executing and no PromptReady, When time passes, Then the mode stays Executing

## QUALITY BAR

Must-have:
  - Duplicate and unregistered frames do not mutate state
  - CommandFinished alone does not open the editor

Must-not-have:
  - A pretyping queue
  - Output-block rendering
  - A 50ms timer copied as a second source of truth. The shell's `line_empty` flag is the signal

Open question risks:
  - none identified beyond the spec assumptions

Rollback note:
  - If this reducer is not mounted, the terminal stays on the xterm key path.

Red flags:
  - Opening Editor from OSC 133 → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes
Uncertain when: a frame shape from T1 cannot represent `prompt_suppressed`
Escalate when: the reducer needs to read xterm screen cells to decide suppression

---

### Task 6: Schedule completion and record submit on the client [depends: T5] [parallel: T7] [test-risk]

## OBJECTIVE

Own client completion state and the Enter sequence. The client invalidates stale results, coalesces one replacement, and calls the host only through the T1 RPC names.

Steps:
1. Write failing test for: repeated Tab keeps the request
   Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
   Level: unit
   Test intent: Given an in-flight request for the same revision and cursor and nothing is showing, When Tab is pressed, Then the same request id stays in flight and no second RPC is made
   Exercise through: `CompletionScheduler`
   Test doubles: a fake host that records RPC calls. Do not mock the scheduler
   Expected RED: Tab starts a second request
2. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "repeated Tab keeps the request"`
3. Consume Tab and keep the request. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a repeated completion request"`
4. Write failing test for: a context change coalesces one replacement
   Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
   Level: unit
   Test intent: Given refresh is on and a generator is active for revision 4, request id A, and prompt generation 3, When the cursor or draft changes, Then request A can no longer apply and exactly one replacement for the new revision, cursor, and prompt generation is issued after the fake host reports the generator finished
   Exercise through: `CompletionScheduler`
   Test doubles: a fake host. Do not use the idle clock to start a second generator while one is active
   Expected RED: the old result still applies after the cursor moves
5. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "context change coalesces one replacement"`
6. Invalidate and coalesce. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(terminal): refresh completion from the latest context"`
7. Write failing test for: refresh off waits for Tab
   Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
   Level: unit
   Test intent: Given refresh is off and a generator is active, When the user types, Then the previous result is invalidated and no replacement is issued. When Tab is later pressed and nothing is showing, Then one request is issued for the latest context
   Exercise through: `CompletionScheduler`
   Test doubles: a fake host
   Expected RED: the keystroke starts a replacement while refresh is off
8. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "refresh off waits for Tab"`
9. Suppress the replacement until Tab. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(terminal): wait for Tab when completion refresh is off"`
10. Write failing test for: typing requests one completion
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given refresh is on, the shell is AtPrompt, and no generator is active, When the user types "git sta" and the 100ms idle window elapses, Then exactly one completion RPC is started for that revision and cursor and the buffer stays "git sta"
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host and a fake clock
    Expected RED: no completion RPC is started after the idle window
11. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "typing requests one completion"`
12. Start one request after the idle window. Also write every completion payload into the draft, so the mismatch test fails. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(terminal): request completion after the idle window"`
13. Write failing test for: a matching completion leaves the draft
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given the draft is "git sta", the request is still in flight, and the result matches the session, mode, revision, cursor, prompt generation, request id, and cwd, When that result arrives, Then the draft is still "git sta"
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host. Do not mock the scheduler
    Expected RED: the matching result changes the draft
14. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "matching completion leaves the draft"`
15. Leave a matching in-flight result out of the draft. Keep writing a mismatched result, a timeout replacement, a late payload, and a `NotStarted` payload into the draft for now, so those later tests fail. Later steps remove those writes one at a time. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(terminal): keep the draft when a completion result matches"`
16. Write failing test for: idle typing with refresh off does not request
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given refresh is off and no generator is active, When the user types and the 100ms window elapses, Then no completion RPC is started
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host and a fake clock
    Expected RED: the idle window starts an RPC while refresh is off
17. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "idle typing with refresh off does not request"`
18. Do not start that RPC. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(terminal): skip idle completion when refresh is off"`
19. Write failing test for: a mismatched result is ignored
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given a result whose session, mode, revision, cursor, prompt generation, request id, or cwd does not match, When it arrives, Then the draft is unchanged
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host
    Expected RED: the mismatched result changes the draft
20. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "mismatched result is ignored"`
21. Drop a mismatched result. Also replace the draft when the generator times out, so the next test fails. Re-run steps 14 and 20 and verify PASS. Commit: `git commit -m "feat(terminal): drop a stale completion result"`
22. Write failing test for: a timeout leaves the draft
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given the buffer is "git sta" and the shell is still a trusted prompt, When the generator timeout fires, Then the draft is still "git sta", Editor remains, and the history recorder is not called
    Exercise through: `CompletionScheduler`
    Test doubles: a fake clock and a fake history recorder
    Expected RED: the timeout changes the draft or records history
23. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "timeout leaves the draft"`
24. Cancel the generator and leave the draft on timeout. Put the duration in one named constant. When a payload for a finished request arrives, write it into the draft, so the late-result test fails. Re-run steps 14 and 23 and verify PASS. Commit: `git commit -m "feat(terminal): keep the draft when completion times out"`
25. Write failing test for: a finished request frees the client slot
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given the host reports the active generator finished, When Tab is pressed for a new context with nothing showing, Then one new request starts
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host
    Expected RED: the next Tab is refused
26. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "finished request frees the client slot"`
27. Release the slot and keep the late-payload write from step 24. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(terminal): accept one new completion after the generator finishes"`
28. Write failing test for: a late result is ignored
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given request A has finished, When a result for A arrives, Then the draft is unchanged
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host
    Expected RED: the late result changes the draft
29. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "late result is ignored"`
30. Ignore that result. Re-run steps 14 and 29 and verify PASS. Commit: `git commit -m "feat(terminal): ignore a completion result after its request finishes"`
31. Write failing test for: path completion uses the chip cwd
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given the chip cwd is "/repo", When completion is requested, Then the RPC cwd is "/repo"
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host
    Expected RED: the RPC cwd is empty or differs from "/repo"
32. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "path completion uses the chip cwd"`
33. Send that cwd. Re-run step 32 and verify PASS. Commit: `git commit -m "feat(terminal): bind completion to the cwd chip"`
34. Write failing test for: a missing cwd skips path completion
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given the chip cwd is absent, When completion is requested, Then the path provider is not asked and another eligible provider may still be requested
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host that records providers
    Expected RED: the path provider is still requested
35. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "missing cwd skips path completion"`
36. Omit the path provider. Also write a `NotStarted` payload into the draft, so the next test fails. Re-run steps 14 and 35 and verify PASS. Commit: `git commit -m "feat(terminal): skip path completion when cwd is missing"`
37. Write failing test for: NotStarted leaves the draft
    Test file: `src/renderer/lib/command-lifecycle/completion.test.ts`
    Level: unit
    Test intent: Given this client's request returns `NotStarted`, When the result arrives, Then the draft is unchanged and no replacement starts until a later explicit request
    Exercise through: `CompletionScheduler`
    Test doubles: a fake host
    Expected RED: `NotStarted` changes the draft or starts another request immediately
38. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/completion.test.ts -t "NotStarted leaves the draft"`
39. Keep the draft. Re-run steps 14 and 38 and verify PASS. Commit: `git commit -m "feat(terminal): keep the draft when completion is not started"`
40. Write failing test for: Enter submits the buffer
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given buffer "git sta", highlight "git status", and an active completion, When Enter is pressed, Then pending results are discarded, the replacement is suppressed, cancel runs before submit, and submit is called once with "git sta"
    Exercise through: `submitDraft`
    Test doubles: a fake host. Do not mock `submitDraft`
    Expected RED: Enter submits the highlight or skips cancel
41. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "Enter submits the buffer"`
42. Implement that Enter path. Do not record history in this step. Re-run step 41 and verify PASS. Commit: `git commit -m "feat(terminal): submit the draft after completion is drained"`
43. Write failing test for: accepted submit records history
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given the host returns `Accepted` for "git sta", terminal name "local", terminal id "t1", and project id "p1", When submit finishes, Then the function returned by `useAddCommand()` is called once with those four arguments and `useCommandHistoryStore` contains one matching entry
    Exercise through: `submitDraft` and the function returned by `useAddCommand()`. The test renders a component that calls the hook and passes that function into `submitDraft`
    Test doubles: a fake host. Mock only `persistenceApi.write`. Do not mock `useAddCommand`, the returned function, or the history store
    Expected RED: the store has no entry after `Accepted`
44. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "accepted submit records history"`
45. `submitDraft` accepts the function returned by `useAddCommand()` and calls it once with the draft, terminal name, terminal id, and project id. Do not call the hook inside `submitDraft`. Do not trim inside `submitDraft`. When the draft is empty, whitespace-only, or contains a newline, also write that raw draft into the store, so the next history tests fail. On `Accepted`, set a local committed-submit flag on the result `submitDraft` returns. On `PreWrite`, clear the draft, call the history function, and set that same flag, so the next test fails. Do not create `session-store.ts` and do not read `sessionStore`. Re-run step 44 and verify PASS. Commit: `git commit -m "feat(terminal): record history after an accepted submit"`
46. Write failing test for: a pre-write failure restores the draft
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given the host returns `PreWrite` for "make test" while the session is a trusted prompt, When submit finishes, Then the draft is "make test", history is not called, and the local committed-submit flag on the `submitDraft` result is unset. The test reads that flag from the result. It does not use `sessionStore`
    Exercise through: `submitDraft`
    Test doubles: a fake host and a fake history recorder. Do not mock `submitDraft`
    Expected RED: the draft is cleared, history is called, and that flag is set
47. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "pre-write failure restores the draft"`
48. On `PreWrite`, restore the draft "make test", do not call history, and leave the local committed-submit flag unset. `Accepted` still sets that flag and still calls history. On `Partial`, restore the editor and call history, so the next test fails. Re-run step 47 and verify PASS. Commit: `git commit -m "feat(terminal): restore the draft when submit never reached the PTY"`
49. Write failing test for: a partial submit stays down
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given the host returns `Partial`, When submit finishes, Then the editor stays unmounted and history is not called
    Exercise through: `submitDraft`
    Test doubles: a fake host and a fake history recorder
    Expected RED: `Partial` restores the editor or records history
50. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "partial submit stays down"`
51. For `Partial`, leave the editor unmounted and do not call history. When the history recorder rejects after `Accepted`, call submit a second time and write one log line that contains the command text, so the next test fails. Leave the local committed-submit flag set on `Accepted`. Re-run step 50 and verify PASS. Commit: `git commit -m "feat(terminal): stay out of the editor after a partial submit"`
52. Write failing test for: a history failure does not resubmit
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given the host already returned `Accepted` for "git sta", `submitDraft` has set its local committed-submit flag, and the history recorder then rejects, When submit finishes, Then submit is not called again, that flag stays set, and the log sink does not contain "git sta". The test does not set the flag itself and does not use `sessionStore`
    Exercise through: `submitDraft`
    Test doubles: a rejecting history recorder and a log sink. Do not mock `submitDraft`
    Expected RED: the recorder failure calls submit a second time and the log contains "git sta"
53. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "history failure does not resubmit"`
54. Log the failure without the command text. Do not call submit again. Leave the local committed-submit flag set for that `Accepted` result. Re-run step 53 and verify PASS. Commit: `git commit -m "feat(terminal): keep a running command when history cannot be saved"`
55. Write failing test for: an empty enter is not history
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given an empty buffer, When Enter is pressed and the host returns `Accepted`, Then `useCommandHistoryStore` has no new entry
    Exercise through: `submitDraft` and the function returned by `useAddCommand()`
    Test doubles: a fake host. Mock only `persistenceApi.write`
    Expected RED: the store contains an empty command
56. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "empty enter is not history"`
57. Remove the extra store write when the draft is empty. Keep the extra raw write for a whitespace-only draft and for a draft that contains a newline. Re-run step 56 and verify PASS. Commit: `git commit -m "feat(terminal): skip history for an empty enter"`
58. Write failing test for: whitespace is not history
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given a buffer of only spaces, When Enter is pressed and the host returns `Accepted`, Then `useCommandHistoryStore` has no new entry
    Exercise through: `submitDraft` and the function returned by `useAddCommand()`
    Test doubles: a fake host. Mock only `persistenceApi.write`
    Expected RED: the store contains the spaces
59. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "whitespace is not history"`
60. Remove the extra store write for a whitespace-only draft. Keep the extra raw write when the draft contains a newline. Re-run step 59 and verify PASS. Commit: `git commit -m "feat(terminal): skip history for a blank command"`
61. Write failing test for: history keeps internal newlines
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given the buffer is "  git\nstatus  " and the host returns `Accepted`, When history is recorded, Then the store contains one command "git\nstatus"
    Exercise through: `submitDraft` and the function returned by `useAddCommand()`
    Test doubles: a fake host. Mock only `persistenceApi.write`
    Expected RED: the store contains the untrimmed buffer or more than one entry
62. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "history keeps internal newlines"`
63. Delete the extra store write. Leave only the call to the injected function. `useAddCommand()` already trims and skips a blank command. Do not call the hook inside `submitDraft` and do not add a trim function there. Re-run step 62 and verify PASS. Commit: `git commit -m "feat(terminal): trim history and keep internal newlines"`
64. Write failing test for: the client does not paint the command
    Test file: `src/renderer/lib/command-lifecycle/submit.test.ts`
    Level: unit
    Test intent: Given an accepted submit of "git remote -v", When `handleAcceptedSubmit` runs, Then the xterm write spy receives zero calls containing that command
    Exercise through: `handleAcceptedSubmit`
    Test doubles: an xterm write spy. Do not mock `handleAcceptedSubmit`
    Expected RED: `handleAcceptedSubmit` does not exist, so the test does not compile
65. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit.test.ts -t "the client does not paint the command"`
66. Add `handleAcceptedSubmit` so it does not call xterm write. Re-run step 65 and verify PASS. Refactor while green. Commit: `git commit -m "feat(terminal): leave the submitted command echo to the shell"`

## REFERENCES LOADED

Spec — scheduling section and the submit-once acceptance rule
src/renderer/hooks/use-command-history.ts — `useAddCommand()` takes no arguments and returns the function that stores `command`, `terminalName`, `terminalId`, and `projectId`. The hook trims and skips a blank command. `submitDraft` receives that function from a React caller and does not call the hook
`@codemirror/autocomplete@6.20.3` `index.d.ts` — `activateOnTypingDelay` defaults to 100. The scheduler may use that window only while no generator is active. The library must not start the generator. `activateOnTyping` stays false in T8.

## WHY THIS APPROACH

Complexity: standard
Justification: The Tab, coalesce, and Enter rules are one client state machine. The host lock in T3 is the other half. This task proves the client order.

## SANDWICH CONTEXT

[CRITICAL: Invalidate on a context change. A repeated Tab for the same context does not restart the request.]
You are implementing the client completion scheduler and submit coordinator.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. Enter beats completion.
Files in scope: completion.ts, completion.test.ts, submit.ts, submit.test.ts. Call the function returned by `useAddCommand()`. Do not call the hook from `submitDraft`. Do not edit the hook. The committed-submit flag in this task is a field on the result `submitDraft` returns. Task 12 stores a separate session flag from `bindSessionClient`. Do not create `session-store.ts` and do not read `sessionStore`.
Available after: T5 reducer. The host methods are the T1 names. T9 supplies the real transport later.
Architecture rule: do not call `terminalApi.write` for submit or completion.
[RESTATE: Invalidate on a context change. A repeated Tab for the same context does not restart the request.]

## DELIVERABLE

Given an in-flight request for the same revision and cursor, When Tab is pressed and nothing is showing, Then that request keeps running
Given a draft or cursor change while pending and refresh is on, When the edit happens, Then the old request id is invalid and at most one replacement runs after the generator is safe
Given refresh is off, When the draft changes, Then no replacement starts until the next explicit Tab
Given refresh is on and no generator is active, When typing settles for 100ms, Then one completion RPC starts and the buffer is unchanged
Given refresh is off and no generator is active, When the idle window elapses, Then no RPC starts
Given a result whose session, mode, revision, cursor, prompt generation, request id, or cwd does not match, When it arrives, Then it is ignored
Given a matching in-flight completion result for "git sta", When it arrives, Then the draft is still "git sta"
Given a generator timeout, When it fires, Then the draft is unchanged and history is not called
Given the chip cwd is present, When completion is requested, Then the RPC carries that cwd
Given the chip cwd is absent, When completion is requested, Then the path provider is not asked
Given `NotStarted`, When it arrives, Then the draft is unchanged
Given buffer "git sta" and highlight "git status", When Enter is pressed, Then "git sta" is submitted once after cancel
Given `Accepted` for "git sta" with a terminal name, terminal id, and project id, When history is recorded, Then the function returned by `useAddCommand()` stores one matching entry
Given an empty buffer, When Enter is accepted, Then the history store gains no entry
Given a whitespace-only buffer, When Enter is accepted, Then the history store gains no entry
Given a buffer with surrounding spaces and an internal newline, When it is accepted, Then the store keeps one trimmed command and the internal newline
Given `PreWrite` on a trusted prompt, When submit returns, Then the draft returns, history is unchanged, and the local committed-submit flag on the `submitDraft` result stays unset
[must-not] Given `Partial`, When submit returns, Then the editor stays down and no history is written
Given history persistence fails after `Accepted`, When submit finishes, Then the command is not submitted again, the local committed-submit flag on the `submitDraft` result stays set, and the log omits the command text
[must-not] Given `Accepted`, When the client handles it, Then xterm is not given a second copy of the command

## QUALITY BAR

Must-have:
  - The idle coalesce uses one 100ms window and never overlaps a generator
  - Timeout behavior follows the spec even though the duration is an implementation parameter. Put the duration in one named constant

Must-not-have:
  - Applying a suggestion inside the scheduler
  - Writing the draft from a completion result
  - Logging the draft or the payload
  - Reading `sessionStore` or creating `session-store.ts`

Open question risks:
  - A timeout that is too short makes completion fail closed. Do not change the fail-safe outcome to "fix" the duration.

Rollback note:
  - If the scheduler is not mounted, Tab and Enter stay on the raw PTY path.

Red flags:
  - Two in-flight request ids for one client → STOP
  - History recorded on PreWrite or Partial → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes
Uncertain when: the fake host cannot represent drain-then-submit order
Escalate when: the scheduler needs to parse PTY bytes itself

---

### Task 7: Mount the command editor and keep its draft [depends: T5]

## OBJECTIVE

Show the CodeMirror buffer when the reducer says Editor, and show the existing xterm key path otherwise. Chips, history recall, paste, and the per-terminal draft live here.

Steps:
1. Write failing test for: the surface follows the mode
   Test file: `src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx`
   Level: unit
   Test intent: Given mode Editor, When the surface renders, Then the composer is present and the xterm slot is not the keyboard target. Given mode PTY or Executing, When the surface renders, Then the composer is absent and the child slot is the keyboard target
   Exercise through: `TerminalInputSurface`
   Test doubles: a child marker for the xterm slot. Do not mock the surface
   Expected RED: `TerminalInputSurface` does not exist
2. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx -t "surface follows the mode"`
3. Implement the surface and `CommandEditor.tsx` as its composer child. In `ConnectedTerminal.tsx`, render the surface around the existing terminal body with a small branch. Do not move existing helpers out of that file. Do not create `CommandEditor.test.tsx` in this task. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(terminal): mount the editor only at a trusted prompt"`
4. Write failing test for: a background prompt does not steal focus
   Test file: `src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx`
   Level: unit
   Test intent: Given mode Editor and `focused` is false, When the surface renders, Then the composer is not the document keyboard target. Given `focused` is true, When it renders, Then the composer is the keyboard target
   Exercise through: `TerminalInputSurface`
   Test doubles: none
   Expected RED: an unfocused surface still focuses the composer
5. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx -t "background prompt does not steal focus"`
6. Focus the composer only when `focused` is true. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(terminal): focus the composer only on the visible terminal"`
7. Write failing test for: a scrollback click keeps editor focus
   Test file: `src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx`
   Level: unit
   Test intent: Given Editor, When the scrollback slot is clicked, Then keyboard focus stays on the composer
   Exercise through: `TerminalInputSurface`
   Test doubles: none
   Expected RED: the click moves focus to the xterm slot
8. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx -t "scrollback click keeps editor focus"`
9. Keep focus on the composer. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(terminal): keep editor focus when scrollback is clicked"`
10. Write failing test for: multiline paste stays in the editor
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given a paste of "echo hi\necho there", When the editor stores it, Then both lines are in the document and the PTY write spy is not called
    Exercise through: `applyEditorPaste` with `EditorState` from `@codemirror/state@6.7.0`
    Test doubles: a PTY write spy. Do not mock CodeMirror state
    Expected RED: paste calls the write spy
11. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "multiline paste stays in the editor"`
12. Store both lines and do not write the PTY. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(terminal): paste a multiline draft without writing the PTY"`
13. Write failing test for: paste drops hidden controls
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given a paste that contains ESC, NUL, another C0 control, a newline, a tab, and a bracketed-paste wrapper, When it is stored, Then ESC, NUL, the other C0 control, and the wrapper are absent, and the newline and tab remain
    Exercise through: `applyEditorPaste`
    Test doubles: none
    Expected RED: the document still contains ESC
14. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "paste drops hidden controls"`
15. Drop those bytes. When the paste is an image, write `0x16` and change the document, so the next test fails. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(terminal): store only visible paste text"`
16. Write failing test for: an image paste does nothing
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given an image paste, When it is offered to the editor, Then the document is unchanged and `0x16` is not written
    Exercise through: `applyEditorPaste`
    Test doubles: a PTY write spy
    Expected RED: the image paste writes `0x16` or changes the document
17. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "image paste does nothing"`
18. Ignore the image. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(terminal): ignore an image paste in the editor"`
19. Write failing test for: Up recalls history on the first line
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the first line is empty and the previous history entry is "git status", When Up is pressed, Then the document becomes "git status" and submit is not called
    Exercise through: `editorKeymap` and `runScopeHandlers` from `@codemirror/view@6.43.4`
    Test doubles: a history list and a submit spy
    Expected RED: Up does not change the document
20. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Up recalls history on the first line"`
21. Recall the previous entry. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(terminal): recall history with Up"`
22. Write failing test for: Up on the first line uses the prefix
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the first line is "git" and history contains "git status" then "make test", When Up is pressed, Then the document becomes "git status" and submit is not called
    Exercise through: `editorKeymap`
    Test doubles: a history list and a submit spy
    Expected RED: Up recalls "make test"
23. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Up on the first line uses the prefix"`
24. Filter by the prefix. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(terminal): recall history by prefix"`
25. Write failing test for: Down restores the unsent draft
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given Up replaced the draft "git" with "git status", When Down moves past the newest match, Then the document is "git" again and submit is not called
    Exercise through: `editorKeymap`
    Test doubles: a history list and a submit spy
    Expected RED: Down leaves the document on the history entry
26. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Down restores the unsent draft"`
27. Restore the draft. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(terminal): restore the draft after history recall"`
28. Write failing test for: Up and Down move between multiline rows
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given a two-line document and the caret on the second line, and no dropdown or history popup is open, When Up is pressed, Then the caret moves to the first line and the text stays the same
    Exercise through: `editorKeymap`
    Test doubles: none
    Expected RED: Up recalls history instead of moving the caret
29. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Up and Down move between multiline rows"`
30. Move the caret. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(terminal): move the caret between draft rows"`
31. Write failing test for: the history popup owns Up and Down
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the history popup is open, When Up or Down is pressed, Then only the popup highlight moves and the document text stays the same
    Exercise through: `editorKeymap`
    Test doubles: a popup model
    Expected RED: Up changes the document while the popup is open
32. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "history popup owns Up and Down"`
33. Move only the highlight. Re-run step 32 and verify PASS. Commit: `git commit -m "feat(terminal): move the history popup highlight"`
34. Write failing test for: Enter on a history row fills the buffer
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the history popup highlight is "make test", When Enter is pressed, Then the document becomes "make test", the popup closes, and submit is not called
    Exercise through: `editorKeymap`
    Test doubles: a submit spy
    Expected RED: Enter submits "make test"
35. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Enter on a history row fills the buffer"`
36. Fill the buffer and close the popup. Re-run step 35 and verify PASS. Commit: `git commit -m "feat(terminal): fill the draft from the history popup"`
37. Write failing test for: Ctrl+R stays an app shortcut
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given commandHistory is ctrl+r, When Ctrl+R is pressed, Then the app shortcut is reported and the document is unchanged
    Exercise through: `editorKeymap` and `isAppOwnedTerminalShortcut`
    Test doubles: none
    Expected RED: Ctrl+R is inserted into the document
38. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Ctrl+R stays an app shortcut"`
39. Report the shortcut. Re-run step 38 and verify PASS. Commit: `git commit -m "feat(terminal): keep Ctrl+R for app history"`
40. Write failing test for: Ctrl+C with a selection copies
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given xterm has a non-empty selection, When Ctrl+C is pressed in the editor, Then copy is requested and the document is unchanged
    Exercise through: `editorKeymap`
    Test doubles: a selection probe and a copy spy
    Expected RED: Ctrl+C clears the document
41. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Ctrl+C with a selection copies"`
42. Copy and leave the document. Re-run step 41 and verify PASS. Commit: `git commit -m "feat(terminal): copy an xterm selection from the editor"`
43. Write failing test for: Ctrl+C clears an unselected buffer
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the editor document is "git status" and xterm's selection is empty, When Ctrl+C is pressed, Then the document becomes empty and the PTY write spy is not called
    Exercise through: `editorKeymap`
    Test doubles: a PTY write spy
    Expected RED: Ctrl+C writes to the PTY
44. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Ctrl+C clears an unselected buffer"`
45. Clear the document and send nothing. Re-run step 44 and verify PASS. Commit: `git commit -m "feat(terminal): clear the draft on Ctrl+C"`
46. Write failing test for: Ctrl+D on an empty buffer sends EOF
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given an empty editor buffer, When Ctrl+D is pressed, Then the editor requests leave and one EOF write, and the document stays empty
    Exercise through: `editorKeymap`
    Test doubles: an EOF spy
    Expected RED: Ctrl+D does not request EOF
47. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Ctrl+D on an empty buffer sends EOF"`
48. Request EOF and leave when the buffer is empty. When the caret is at the end of a non-empty buffer, delete one character, so the next test fails. Do not delete when the caret is in the middle. Re-run step 47 and verify PASS. Commit: `git commit -m "feat(terminal): send EOF from an empty editor"`
49. Write failing test for: Ctrl+D at the end does not delete
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the caret is at the end of a non-empty buffer, When Ctrl+D is pressed, Then the document is unchanged and EOF is not requested
    Exercise through: `editorKeymap`
    Test doubles: an EOF spy
    Expected RED: Ctrl+D deletes a character or sends EOF
50. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Ctrl+D at the end does not delete"`
51. Ignore Ctrl+D when the caret is at the end of a non-empty buffer. Do not delete a character and do not send EOF. Re-run step 50 and verify PASS. Commit: `git commit -m "feat(terminal): ignore Ctrl+D at the end of the draft"`
52. Write failing test for: Ctrl+D in the middle deletes one character
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the caret is in the middle of "git status", When Ctrl+D is pressed, Then the character under the caret is deleted and EOF is not requested
    Exercise through: `editorKeymap`
    Test doubles: an EOF spy
    Expected RED: the document is unchanged
53. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Ctrl+D in the middle deletes one character"`
54. Delete the character under a caret in the middle. Do not delete and do not send EOF when the caret is at the end of a non-empty buffer. Re-run steps 50 and 53 and verify PASS. Commit: `git commit -m "feat(terminal): delete the character under the caret"`
55. Write failing test for: Shift+Enter inserts a newline
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the editor document, When Shift+Enter is pressed, Then one newline is inserted and submit is not called
    Exercise through: `editorKeymap`
    Test doubles: a submit spy
    Expected RED: Shift+Enter submits the buffer
56. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "Shift+Enter inserts a newline"`
57. Insert the newline. Also, when an IME composition is open, insert that composition into the document and submit it on Enter, so the next test fails. Re-run step 56 and verify PASS. Commit: `git commit -m "feat(terminal): insert a newline on Shift+Enter"`
58. Write failing test for: uncommitted IME text is not submitted
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given IME composition is open, When Enter is pressed, Then the document does not include the uncommitted text and submit is not called
    Exercise through: `editorKeymap`
    Test doubles: an IME composition state and a submit spy
    Expected RED: Enter submits the uncommitted text
59. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "uncommitted IME text is not submitted"`
60. Leave the composition out of the document. Re-run step 59 and verify PASS. Commit: `git commit -m "feat(terminal): keep uncommitted IME text out of the draft"`
61. Write failing test for: submitting swallows keystrokes
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the mode is Submitting, When a character is pressed, Then the draft is unchanged and the PTY write spy is not called
    Exercise through: `editorKeymap`
    Test doubles: a PTY write spy
    Expected RED: the character is appended or written
62. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "submitting swallows keystrokes"`
63. Drop the key. Re-run step 62 and verify PASS. Commit: `git commit -m "feat(terminal): discard keys while a command is submitting"`
64. Write failing test for: keys outside the editor are discarded
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the mode is PTY, When a character is offered to the editor store, Then the stored draft is unchanged
    Exercise through: `commandDraftStore`
    Test doubles: none
    Expected RED: the character is copied into the draft
65. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "keys outside the editor are discarded"`
66. Discard it. Re-run step 65 and verify PASS. Commit: `git commit -m "feat(terminal): discard keystrokes that arrive outside the editor"`
67. Write failing test for: a live editor remounts its draft
    Test file: `src/renderer/lib/command-lifecycle/draft-store.test.ts`
    Level: unit
    Test intent: Given terminal A is Editor on a live process with unsent draft "make test", When its surface unmounts and mounts again, Then the draft is "make test". The store has no kill function
    Exercise through: `commandDraftStore`
    Test doubles: none
    Expected RED: the second mount starts empty
68. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/draft-store.test.ts -t "live editor remounts its draft"`
69. Keep the draft and mount the composer whenever the store has text, including when the mode is raw, so the next test fails. Re-run step 68 and verify PASS. Commit: `git commit -m "feat(terminal): keep an unsent draft on the same terminal"`
70. Write failing test for: a raw remount does not show the draft
    Test file: `src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx`
    Level: unit
    Test intent: Given the store still holds "make test" and the reducer says raw, When the surface mounts, Then the composer is absent
    Exercise through: `TerminalInputSurface`
    Test doubles: the draft store
    Expected RED: the composer is mounted with "make test"
71. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx -t "raw remount does not show the draft"`
72. Hide the composer. Re-run step 71 and verify PASS. Commit: `git commit -m "feat(terminal): hide the draft while the session is raw"`
73. Write failing test for: process exit drops the stored draft
    Test file: `src/renderer/lib/command-lifecycle/draft-store.test.ts`
    Level: unit
    Test intent: Given terminal A holds "make test", When the store is told the process exited, Then that draft is gone
    Exercise through: `commandDraftStore`
    Test doubles: none
    Expected RED: the draft remains after exit
74. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/draft-store.test.ts -t "process exit drops the stored draft"`
75. Drop it. Re-run step 74 and verify PASS. Commit: `git commit -m "feat(terminal): drop the stored draft when the shell exits"`
76. Write failing test for: an accepted submit clears the stored draft
    Test file: `src/renderer/lib/command-lifecycle/draft-store.test.ts`
    Level: unit
    Test intent: Given the store holds "git sta" and submit is accepted, When the next Editor mount happens, Then the document is empty
    Exercise through: `commandDraftStore`
    Test doubles: none
    Expected RED: the next mount shows "git sta"
77. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/draft-store.test.ts -t "accepted submit clears the stored draft"`
78. Clear the stored draft on acceptance. Re-run step 77 and verify PASS. Commit: `git commit -m "feat(terminal): clear the stored draft after an accepted submit"`
79. Write failing test for: a missing chip is hidden
    Test file: `src/renderer/components/terminal/command-editor/chips.test.tsx`
    Level: unit
    Test intent: Given cwd is "/repo" and git branch is absent, When chips render, Then the cwd chip shows "/repo" and the branch chip is absent
    Exercise through: the chips component
    Test doubles: none
    Expected RED: the chips component does not exist
80. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/chips.test.tsx -t "missing chip is hidden"`
81. Render only present values. Re-run step 80 and verify PASS. Commit: `git commit -m "feat(terminal): hide a chip whose value is missing"`
82. Write failing test for: a chip update stays in the editor
    Test file: `src/renderer/components/terminal/command-editor/chips.test.tsx`
    Level: unit
    Test intent: Given the branch chip is hidden, When git branch becomes "main", Then the branch chip shows "main" and the surface mode is still Editor
    Exercise through: the chips component inside `TerminalInputSurface`
    Test doubles: none
    Expected RED: the branch update unmounts the composer
83. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/chips.test.tsx -t "chip update stays in the editor"`
84. Update the chip in place. Re-run step 83 and verify PASS. Commit: `git commit -m "feat(terminal): update context chips without leaving the editor"`
85. Write failing test for: readline keys edit the draft
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given Ctrl+A, Ctrl+E, Ctrl+K, Ctrl+U, and Ctrl+W are not the active app shortcut, When each is pressed, Then it edits the buffer and the PTY write spy is not called
    Exercise through: `editorKeymap` and `isAppOwnedTerminalShortcut`
    Test doubles: a PTY write spy
    Expected RED: one of those keys is written to the PTY
86. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "readline keys edit the draft"`
87. Bind those edits. Re-run step 86 and verify PASS. Commit: `git commit -m "feat(terminal): edit the draft with readline keys"`
88. Write failing test for: editor paste does not call terminalApi.write
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given Editor mode, When `handleTerminalPaste` receives "echo hi", Then `applyEditorPaste` stores it and `terminalApi.write` is not called. `ConnectedTerminal.tsx` `pasteText` calls `handleTerminalPaste` when the mode prop already passed into `TerminalInputSurface` is Editor. That call site does not mention `sessionStore` or `session-store.ts`
    Exercise through: `handleTerminalPaste` and a source read of `pasteText` around line 346
    Test doubles: a write spy. Do not mock `handleTerminalPaste`
    Expected RED: `handleTerminalPaste` does not exist, so the test does not compile
89. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "editor paste does not call terminalApi.write"`
90. Add `handleTerminalPaste` and call it from `pasteText` only when the mode prop already passed into `TerminalInputSurface` is Editor. Do not create `session-store.ts`. Do not read `sessionStore`. Re-run step 89 and verify PASS. Commit: `git commit -m "feat(terminal): paste into the editor instead of the PTY"`
91. Write failing test for: an editor image paste does not write 0x16
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given Editor mode, When `handleTerminalImagePaste` runs, Then the document is unchanged and the write spy does not receive `0x16`. `onImagePaste` around line 363 contains a call to `handleTerminalImagePaste`
    Exercise through: `handleTerminalImagePaste` and a source read of `onImagePaste`
    Test doubles: a write spy
    Expected RED: `handleTerminalImagePaste` does not exist, so the test does not compile
92. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "editor image paste does not write 0x16"`
93. Add that function and the call. Re-run step 92 and verify PASS. Commit: `git commit -m "feat(terminal): ignore an image paste while the editor is mounted"`
94. Write failing test for: a PTY paste still writes
    Test file: `src/renderer/components/terminal/command-editor/editor-keys.test.ts`
    Level: unit
    Test intent: Given the mode is PTY, When `handleTerminalPaste` receives text, Then today's `terminalApi.write` still runs
    Exercise through: `handleTerminalPaste`
    Test doubles: a write spy
    Expected RED: the PTY paste is swallowed by the editor
95. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/editor-keys.test.ts -t "PTY paste still writes"`
96. Keep the PTY write. Re-run step 95 and verify PASS. Commit: `git commit -m "feat(terminal): keep PTY paste on the existing write path"`
97. Write failing test for: alternate screen comes from the mode tracker
    Test file: `src/renderer/lib/command-lifecycle/screen-mode.test.ts`
    Level: unit
    Test intent: Given `getTerminalModes` from `src/renderer/utils/terminal-registry.ts` reports `alternateScreen` true, When `applyScreenMode` runs, Then `reduceSession` receives an alternate-screen enter and the mode is PTY
    Exercise through: `applyScreenMode`
    Test doubles: a fake modes object. Do not mock `reduceSession` or reimplement CSI parsing
    Expected RED: `applyScreenMode` does not exist, so the test does not compile
98. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/screen-mode.test.ts -t "alternate screen comes from the mode tracker"`
99. Add `screen-mode.ts`. Call `applyScreenMode` from `ConnectedTerminal.tsx` with one call beside the existing terminal body. Treat `alternateScreen` false as Editor so the next test fails. Re-run step 98 and verify PASS. Commit: `git commit -m "feat(terminal): follow the existing alternate-screen tracker"`
100. Write failing test for: leaving the alternate screen does not open the editor
    Test file: `src/renderer/lib/command-lifecycle/screen-mode.test.ts`
    Level: unit
    Test intent: Given the mode is PTY because `alternateScreen` was true, When `getTerminalModes` then reports false and no PromptReady has arrived, Then the mode stays PTY
    Exercise through: `applyScreenMode`
    Test doubles: a fake modes object
    Expected RED: `alternateScreen` false opens Editor
101. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/screen-mode.test.ts -t "leaving the alternate screen does not open the editor"`
102. Wait for PromptReady. Re-run step 101 and verify PASS. Refactor while green. Commit: `git commit -m "feat(terminal): wait for a prompt after the alternate screen closes"`

## REFERENCES LOADED

Spec — composer is the prompt, paste rule, history recall, chips, project switch
src/renderer/components/terminal/keymap.ts — `isAppOwnedTerminalShortcut`
src/renderer/components/terminal/ConnectedTerminal.tsx — mount point. The file stays 2007 lines plus the wiring hunk
`@codemirror/state@6.7.0` and `@codemirror/view@6.43.4` — existing tests construct `EditorState` and call `runScopeHandlers`
docs/design/forms.md — semantic tokens for the composer field

## WHY THIS APPROACH

Complexity: standard
Justification: The visible editor is separable from completion math. A new surface keeps the large terminal file on the xterm path it already owns.

## SANDWICH CONTEXT

[CRITICAL: Do not split ConnectedTerminal.tsx. The composer is mounted only in Editor.]
You are implementing the command editor surface.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. Continuous xterm scrollback stays underneath.
Files in scope: the command-editor view files, the draft store, and the ConnectedTerminal wiring hunk.
Available after: T5
Architecture rule: semantic tokens only. No Tailwind palette primitives.
[RESTATE: Do not split ConnectedTerminal.tsx. The composer is mounted only in Editor.]

## DELIVERABLE

Given a trusted Editor mode, When the surface renders, Then the composer is mounted
Given a TUI or a running command, When the surface renders, Then the composer is gone and the xterm slot keeps the keyboard
Given Editor and the terminal is not focused, When the surface renders, Then the composer does not take keyboard focus
Given a multiline paste, When it is stored, Then nothing is written to the PTY until Enter
Given a paste contains ESC, NUL, or a bracketed-paste wrapper, When it is stored, Then those bytes are dropped and newline and tab remain
Given an image paste, When it is offered, Then the document is unchanged and `0x16` is not written
Given an empty first line, When Up is pressed, Then the previous history entry fills the buffer and does not run
Given a history popup row, When Enter is pressed, Then the buffer is filled and the command does not run
Given xterm has a selection, When Ctrl+C is pressed, Then the buffer is unchanged
Given no selection, When Ctrl+C is pressed, Then the buffer is cleared and nothing is written
Given an empty buffer, When Ctrl+D is pressed, Then EOF is sent and the editor leaves
Given the caret at the end of a non-empty buffer, When Ctrl+D is pressed, Then the buffer is unchanged and EOF is not sent
Given the caret in the middle, When Ctrl+D is pressed, Then one character is deleted
Given Submitting, When a key arrives, Then it is discarded
Given a live Editor remounts, When the surface returns, Then the unsent draft is still there and no kill was requested
Given the reducer says raw, When the surface mounts, Then the composer is absent
Given an accepted submit, When the next Editor mounts, Then the buffer is empty
Given Editor mode, When `pasteText` or `onImagePaste` runs, Then it calls the editor paste handler and does not call `terminalApi.write`. `pasteText` chooses that handler from the mode prop already passed into `TerminalInputSurface` and does not read `sessionStore`
Given PTY mode, When paste runs, Then `terminalApi.write` still runs
Given `getTerminalModes` reports `alternateScreen`, When `applyScreenMode` runs, Then the mode is PTY
Given `alternateScreen` returns to false and no PromptReady has arrived, When `applyScreenMode` runs, Then the mode stays PTY
[derived] Given cwd or git branch is missing, When chips render, Then that chip is absent

## QUALITY BAR

Must-have:
  - New files stay under about 800 production lines. Split a new file into further new files before it lands
  - Image paste does not write `0x16` and does not change the document
  - Shift+Enter inserts a newline. Uncommitted IME text is not part of the document

Must-not-have:
  - Cases added to `ConnectedTerminal.test.tsx`
  - An output block
  - A Node version chip
  - Creating `session-store.ts` or reading `sessionStore`

Open question risks:
  - none identified

Rollback note:
  - Leaving the surface unmounted keeps today's xterm input.

Red flags:
  - A PTY write from paste or Up → STOP
  - A kill on project switch → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes and ConnectedTerminal changed only by the surface mount
Uncertain when: `runScopeHandlers` cannot see the editor keymap
Escalate when: the mount requires moving existing ConnectedTerminal helpers into a new file

---

### Task 8: Accept a suggestion without running it [depends: T6, T7]

## OBJECTIVE

Show ghost text and the completion dropdown from the scheduler's current valid result. Tab, click, and end-of-line Right accept into the buffer and do not submit.

Steps:
1. Write failing test for: Tab accepts the highlighted row
   Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
   Level: unit
   Test intent: Given buffer "git rem" and highlighted row "git remote" with matching ghost text, When Tab is pressed, Then the document becomes "git remote", submit is not called, and the ghost and dropdown are cleared before the next key
   Exercise through: `acceptHighlighted`
   Test doubles: a submit spy. Do not mock `acceptHighlighted`
   Expected RED: Tab does not change the document
2. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Tab accepts the highlighted row"`
3. Add `@codemirror/autocomplete@6.20.3` as a direct dependency. Implement Tab accept. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(terminal): accept a completion into the draft"`
4. Write failing test for: a click accepts the row
   Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
   Level: unit
   Test intent: Given an open dropdown, When any row is clicked, Then that row is inserted, the dropdown and ghost clear, and submit is not called
   Exercise through: `acceptHighlighted`
   Test doubles: a submit spy
   Expected RED: the click does not change the document
5. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "click accepts the row"`
6. Accept the clicked row. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(terminal): accept a clicked completion"`
7. Write failing test for: the editor installs the completion extension
   Test file: `src/renderer/components/terminal/command-editor/CommandEditor.test.tsx`
   Level: unit
   Test intent: Given `CommandEditor` mounts, When its state is read, Then the autocomplete extension from `completion-view.ts` is installed with `defaultKeymap: false` and `activateOnTyping: false`, and Enter is not bound to `acceptCompletion`
   Exercise through: rendering `CommandEditor`
   Test doubles: none. Do not mock `CommandEditor`
   Expected RED: `CommandEditor.test.tsx` fails because the extension is not installed
8. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/CommandEditor.test.tsx -t "editor installs the completion extension"`
9. Install `autocompletion({ override, defaultKeymap: false, activateOnTyping: false, selectOnOpen: true })` inside `CommandEditor.tsx`. The override reads the scheduler result. It does not start an RPC. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(terminal): install completion inside the command editor"`
10. Write failing test for: dropdown arrows move only the highlight
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given the dropdown is open, When Up or Down is pressed, Then only the highlighted row changes, the caret stays, and history recall does not run
    Exercise through: `runScopeHandlers` on the editor keymap
    Test doubles: a history spy
    Expected RED: Up moves the caret or recalls history
11. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "dropdown arrows move only the highlight"`
12. Rebind Up and Down while the dropdown is open. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(terminal): move the completion highlight with the arrow keys"`
13. Write failing test for: Enter submits the buffer while the dropdown is open
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given an open dropdown whose highlight is "git status" and whose buffer is "git sta", When Enter is pressed, Then submit is called with "git sta" and `acceptCompletion` does not run
    Exercise through: `runScopeHandlers` on the editor keymap
    Test doubles: a submit spy
    Expected RED: Enter accepts the highlight
14. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Enter submits the buffer"`
15. Bind Enter to `submitDraft`. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(terminal): submit the buffer while suggestions are open"`
16. Write failing test for: Esc closes the overlay
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given a dropdown or ghost text, When Esc is pressed, Then the overlay clears, the document is unchanged, and the PTY write spy is not called
    Exercise through: `runScopeHandlers`
    Test doubles: a PTY write spy
    Expected RED: Esc writes to the PTY or changes the document
17. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Esc closes the overlay"`
18. Clear the dropdown and ghost text. Leave an open history popup open, so the next test fails. Do not change the document. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(terminal): close suggestions without writing the PTY"`
19. Write failing test for: Esc closes the history popup
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given the history popup is open and the document is "git sta", When Esc is pressed, Then the popup closes, the document stays "git sta", and the PTY write spy is not called
    Exercise through: `runScopeHandlers`
    Test doubles: a PTY write spy
    Expected RED: the history popup stays open
20. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Esc closes the history popup"`
21. Close the history popup. Do not change the document. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(terminal): close the history popup without writing the PTY"`
22. Write failing test for: the highlighted row shows ghost text
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given buffer "git sta" and highlighted row "git status", When the ghost is shown, Then the editor view state contains the suffix "tus"
    Exercise through: the ghost presentation in `completion-view.ts`
    Test doubles: none
    Expected RED: no ghost suffix is shown
23. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "highlighted row shows ghost text"`
24. Show the suffix by inserting it into the document, so the next test fails. Do not insert it from the Esc handler. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(terminal): show the highlighted completion suffix"`
25. Write failing test for: ghost text is not document text
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given ghost text for the highlighted row, When the document is read before accept, Then the document text does not include the ghost suffix
    Exercise through: the ghost decoration
    Test doubles: none
    Expected RED: the ghost suffix is part of the document
26. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "ghost text is not document text"`
27. Draw the ghost as a decoration and remove it from the document. `selectOnOpen` stays true so the first row and the ghost match. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(terminal): show ghost text for the highlighted completion"`
28. Write failing test for: end-of-line Right accepts ghost text
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given ghost text and the cursor at the end of the buffer, When Right is pressed, Then the ghost is inserted and submit is not called
    Exercise through: the editor keymap
    Test doubles: a submit spy
    Expected RED: Right at the end does not insert the ghost
29. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Right accepts ghost text"`
30. Insert the ghost. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(terminal): accept ghost text at the end of the draft"`
31. Write failing test for: Right elsewhere moves the caret
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given ghost text and the cursor is not at the end, When Right is pressed, Then the cursor moves and the document text is unchanged
    Exercise through: the editor keymap
    Test doubles: a submit spy
    Expected RED: Right inserts the ghost before the end
32. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Right elsewhere moves the caret"`
33. Move the caret. Re-run step 32 and verify PASS. Commit: `git commit -m "feat(terminal): move the caret when ghost text is not accepted"`
34. Write failing test for: Tab with nothing showing requests completion
    Test file: `src/renderer/components/terminal/command-editor/completion-view.test.ts`
    Level: unit
    Test intent: Given no ghost text, no dropdown, and no in-flight request for this context, When Tab is pressed, Then one completion request is made, no tab character is inserted, and focus does not move
    Exercise through: the editor keymap and `CompletionScheduler`
    Test doubles: a fake host
    Expected RED: Tab inserts a tab character
35. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/completion-view.test.ts -t "Tab requests completion"`
36. Request completion and consume the key. Re-run step 35 and verify PASS. Refactor while green. Commit: `git commit -m "feat(terminal): request completion when nothing is showing"`

## REFERENCES LOADED

Spec — accept does not execute
`@codemirror/autocomplete@6.20.3` `index.d.ts`, confirmed in the lockfile and on Context7 `/codemirror/autocomplete` (2026-10-20). `completionKeymap` binds Enter to `acceptCompletion` unless `defaultKeymap` is false. No ghost API exists on `Completion`.
`src/renderer/hooks/use-codemirror.test.ts` — `runScopeHandlers` is the existing key test entry.

## WHY THIS APPROACH

Complexity: standard
Justification: The library owns the dropdown list. Ghost text and the Enter binding are ours, so they stay in a new module instead of a CodeMirror fork.

## SANDWICH CONTEXT

[CRITICAL: Enter submits the buffer. Accepting a suggestion does not run it.]
You are implementing completion presentation for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. `defaultKeymap: false` and `activateOnTyping: false`.
Files in scope: completion-view.ts, its test, CommandEditor.tsx, CommandEditor.test.tsx, package.json, and bun.lock
Available after: T6 scheduler result and T7 editor state
Architecture rule: the scheduler, not the autocomplete extension, decides when an RPC starts.
[RESTATE: Enter submits the buffer. Accepting a suggestion does not run it.]

## DELIVERABLE

Given a highlighted row, When Tab accepts it, Then the buffer changes and the command does not run
Given a dropdown row, When it is clicked, Then that row is inserted and the command does not run
Given `CommandEditor` is mounted, When its extensions are read, Then autocomplete is installed with `defaultKeymap: false` and `activateOnTyping: false`
Given an open dropdown, When Up or Down is pressed, Then only the highlight moves
Given an open dropdown, When Enter is pressed, Then the buffer text is submitted
Given a dropdown or ghost text, When Esc is pressed, Then the overlay closes and the buffer is unchanged
Given a history popup is open, When Esc is pressed, Then the popup closes, the buffer is unchanged, and Esc is not written to the PTY
[derived] Given highlighted row "git status" and buffer "git sta", When the ghost is shown, Then the editor view contains the suffix "tus"
Given ghost text before accept, When the document is read, Then the ghost is not part of the document
[derived] Given the cursor at the end and ghost text showing, When Right is pressed, Then the ghost is inserted and the command does not run
Given the cursor is not at the end, When Right is pressed, Then the caret moves and the text stays
Given nothing is showing and no request is in flight, When Tab is pressed, Then one completion is requested and no tab character is inserted
[must-not] Given the library default keymap, When Enter is pressed, Then `acceptCompletion` must not run

## QUALITY BAR

Must-have:
  - Direct dependency on `@codemirror/autocomplete@6.20.3`
  - Ghost text is a decoration and is not part of the document revision until accept
  - After accept, the dropdown and ghost are gone before the next key

Must-not-have:
  - A second editor for highlighting
  - `activateOnTyping: true`

Open question risks:
  - none identified

Rollback note:
  - Removing the completion extension leaves the T7 buffer working without suggestions.

Red flags:
  - Enter inserting the highlight → STOP
  - Accept calling submit → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes
Uncertain when: `runScopeHandlers` reports both the library Enter binding and the editor binding
Escalate when: ghost text requires a CodeMirror API that the installed `Completion` type does not provide and a decoration cannot represent

---

### Task 9: Send submit and completion on both transports [depends: T1, T3] [parallel: T6]

## OBJECTIVE

Implement the three TerminalApi methods for Tauri IPC and the browser socket. The browser methods use a live correlated request and never the offline input buffer.

Steps:
1. Write failing test for: a closed browser socket does not queue submit
   Test file: `src/renderer/lib/command-editor-transport.test.ts`
   Level: unit
   Test intent: Given the browser socket is not open, When `submitCommand`, `requestCompletion`, or `cancelCompletion` is called, Then the result is a pre-write failure and `inputBuffers` is unchanged. Given the socket is open, When submit is called, Then one correlated request is sent, its `type` is `command_editor_submit`, and it is not a ledgered `write`
   Exercise through: `createWebCommandEditorTransport`
   Test doubles: a fake socket. Do not mock the transport function
   Expected RED: submit reports success and pushes the draft into `inputBuffers`
2. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-editor-transport.test.ts -t "closed socket does not queue submit"`
3. Implement the web methods in the new module. `web-terminal-api.ts` delegates with a thin method. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(web): send session RPCs on the live socket"`
4. Write failing test for: desktop invoke names match the host
   Test file: `src/renderer/lib/command-editor-transport.test.ts`
   Level: unit
   Test intent: Given the Tauri adapter, When submit, completion, and cancel are called, Then they invoke `command_editor_submit`, `command_editor_request_completion`, and `command_editor_cancel_completion`, and none calls `terminal_write`
   Exercise through: `createTauriTerminalApi` after the thin delegation
   Test doubles: a fake `invoke`. Do not mock the adapter method
   Expected RED: the methods are missing on the Tauri adapter
5. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-editor-transport.test.ts -t "desktop invoke names match the host"`
6. Delegate from `tauri-terminal-api.ts`. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(desktop): invoke the session RPCs"`
7. Write failing test for: a lost reply is unknown delivery
   Test file: `src/renderer/lib/command-editor-transport.test.ts`
   Level: unit
   Test intent: Given the browser socket accepted the submit frame and then closed before a reply, When the transport settles, Then the result is `Partial`, `inputBuffers` is unchanged, and the method does not report success
   Exercise through: `createWebCommandEditorTransport`
   Test doubles: a fake socket that accepts one frame and then closes
   Expected RED: the sent request with no reply is reported as success or as `PreWrite`
8. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-editor-transport.test.ts -t "lost reply is unknown delivery"`
9. Map that outcome to `Partial`. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(web): treat a lost session reply as unknown delivery"`
10. Write failing test for: parity checklist names both adapters
   Test file: `src/renderer/lib/__tests__/parity-checklist.test.ts`
   Level: unit
   Test intent: Given the parity checklist, When it scans the Tauri and web terminal adapters, Then both mention `submitCommand`, `requestCompletion`, and `cancelCompletion`
   Exercise through: the existing checklist runner
   Test doubles: none. The checklist reads source, which is its current pattern
   Expected RED: the checklist does not list the three methods
11. Run test — verify FAIL: `bun run test -- src/renderer/lib/__tests__/parity-checklist.test.ts -t "session RPC parity"`
12. Add one describe block for the three methods. Do not reorganize the file. Re-run step 11 and verify PASS. Commit: `git commit -m "test(terminal): pin session RPC parity"`
13. Write failing test for: a closed socket does not queue restorePrompt
    Test file: `src/renderer/lib/command-editor-transport.test.ts`
    Level: unit
    Test intent: Given the browser socket is closed, When `restorePrompt` is called, Then the result is `PreWrite` and `inputBuffers` is unchanged
    Exercise through: `createWebCommandEditorTransport`
    Test doubles: a closed fake socket
    Expected RED: `restorePrompt` is missing, so the test does not compile
14. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-editor-transport.test.ts -t "closed socket does not queue restorePrompt"`
15. Add the web method for the closed-socket failure only. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(web): reject restorePrompt before the socket is open"`
16. Write failing test for: an open socket sends restorePrompt
    Test file: `src/renderer/lib/command-editor-transport.test.ts`
    Level: unit
    Test intent: Given the browser socket is open, When `restorePrompt` is called, Then one correlated request is sent, its `type` is `command_editor_restore_prompt`, and it is not a `write` frame
    Exercise through: `createWebCommandEditorTransport`
    Test doubles: an open fake socket
    Expected RED: the open socket reports `PreWrite` because only the closed path exists
17. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-editor-transport.test.ts -t "open socket sends restorePrompt"`
18. Send the live request. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(web): send restorePrompt on the live socket"`
19. Write failing test for: desktop invoke sends restorePrompt
    Test file: `src/renderer/lib/command-editor-transport.test.ts`
    Level: unit
    Test intent: Given the Tauri adapter, When `restorePrompt` is called, Then it invokes `command_editor_restore_prompt` and does not call `terminal_write`
    Exercise through: `createTauriTerminalApi`
    Test doubles: a fake `invoke`
    Expected RED: the Tauri method is missing
20. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-editor-transport.test.ts -t "desktop invoke sends restorePrompt"`
21. Delegate from `tauri-terminal-api.ts`. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(desktop): invoke restorePrompt"`
22. Write failing test for: parity checklist names restorePrompt
    Test file: `src/renderer/lib/__tests__/parity-checklist.test.ts`
    Level: unit
    Test intent: Given the parity checklist, When it scans both adapters, Then both mention `restorePrompt`
    Exercise through: the existing checklist runner
    Test doubles: none
    Expected RED: the checklist does not list `restorePrompt`
23. Run test — verify FAIL: `bun run test -- src/renderer/lib/__tests__/parity-checklist.test.ts -t "restorePrompt parity"`
24. Add `restorePrompt` to the session describe. Re-run step 23 and verify PASS. Refactor while green. Commit: `git commit -m "test(terminal): pin restore-prompt parity"`

## REFERENCES LOADED

Spec — surfaces rule and the web at-least-once constraint
src/renderer/lib/web-terminal-api.ts — `write` around line 799 and `flushInputBuffer` around line 867
src/renderer/lib/terminal-api.ts — `isTauriContext` switch
src/renderer/lib/__tests__/parity-checklist.test.ts — source scan pattern

## WHY THIS APPROACH

Complexity: standard
Justification: The host gate is useless if the browser queues the draft on the keystroke path. The transport is a separate deliverable from the scheduler.

## SANDWICH CONTEXT

[CRITICAL: Do not queue submit or completion in inputBuffers or the ledgered write path.]
You are implementing the session RPC transports.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. A lost reply after send is unknown delivery, not a second submit.
Files in scope: command-editor-transport.ts, its test, the two adapter delegations, and the parity checklist block.
Available after: T1 method names and T3 command names
Architecture rule: both surfaces stay in parity.
[RESTATE: Do not queue submit or completion in inputBuffers or the ledgered write path.]

## DELIVERABLE

Given desktop, termul-server, and the browser adapters, When submit or completion is requested, Then both adapters expose the same three methods
Given a closed browser socket, When submit, completion, or cancel is called, Then it fails before send and the offline buffer is unchanged
[must-not] Given a session RPC, When it is sent, Then it is not a `write` frame and it is not replayed by the input buffer
Given a reply lost after the request was sent, When the transport settles, Then it reports `Partial` and the offline buffer is unchanged
Given a closed browser socket, When `restorePrompt` is called, Then the result is `PreWrite` and `inputBuffers` is unchanged
Given an open browser socket, When `restorePrompt` is called, Then the request type is `command_editor_restore_prompt` and it is not `write`
Given the Tauri adapter, When `restorePrompt` is called, Then it invokes `command_editor_restore_prompt` and does not call `terminal_write`

## QUALITY BAR

Must-have:
  - Tauri and web delegations are thin
  - Unknown delivery maps to the Partial client outcome in T6

Must-not-have:
  - A reorganization of `web-terminal-api.ts`
  - A new application-root provider

Open question risks:
  - none identified

Rollback note:
  - Unmounted clients never call the new methods, so today's write path remains the raw terminal.

Red flags:
  - Submit returning success while the socket is closed → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes
Uncertain when: the parity checklist cannot see a method that is only re-exported
Escalate when: the web client cannot send a correlated request without using `inputBuffers`

---

### Task 10: Add the automatic completion refresh preference [depends: T6]

## OBJECTIVE

Default automatic completion refresh to true and let the existing preferences page turn it off. The scheduler reads that value.

Steps:
1. Write failing test for: the default is on
   Test file: `src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx`
   Level: unit
   Test intent: Given `DEFAULT_APP_SETTINGS`, When `automaticCompletionRefresh` is read, Then it is true
   Exercise through: `DEFAULT_APP_SETTINGS`
   Test doubles: none
   Expected RED: the settings field does not exist
2. Run test — verify FAIL: `bun run test -- src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx -t "refresh defaults on"`
3. Add the field to `AppSettings` and `DEFAULT_APP_SETTINGS` next to `terminalScreenReaderMode`. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(settings): default automatic completion refresh to on"`
4. Write failing test for: the appearance switch renders on
   Test file: `src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx`
   Level: unit
   Test intent: Given the appearance section receives `automaticCompletionRefresh` true, When it renders, Then the switch is checked. When the user turns it off, Then the change handler is called with false
   Exercise through: rendering the appearance section
   Test doubles: a change handler. Do not mock the switch
   Expected RED: the section has no switch for this setting
5. Run test — verify FAIL: `bun run test -- src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx -t "appearance switch renders on"`
6. Add the switch beside the screen-reader row and pass the value through `AppPreferences.tsx`. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(settings): show the completion refresh switch"`
7. Write failing test for: the scheduler reads the setting
   Test file: `src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx`
   Level: unit
   Test intent: Given `automaticCompletionRefresh` is false on the settings object, When `CompletionScheduler` is constructed from that object and the draft changes with no generator active, Then no replacement is scheduled
   Exercise through: `CompletionScheduler`
   Test doubles: a fake host. Do not mock the settings object
   Expected RED: the scheduler still schedules from its own flag and ignores the settings field
8. Run test — verify FAIL: `bun run test -- src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx -t "scheduler reads the setting"`
9. Read `automaticCompletionRefresh` from settings. Re-run step 8 and the T6 refresh-off filter. Verify PASS. Refactor while green. Commit: `git commit -m "feat(settings): prefer automatic command completion"`

## REFERENCES LOADED

Spec — `automaticCompletionRefresh` defaults to true
src/renderer/types/settings.ts — `AppSettings` line 62 and the default near line 282
src/renderer/pages/app-preferences/appearance-section.tsx — screen-reader switch at lines 109–123
src/renderer/pages/AppPreferences.tsx — the prop passed near line 305

## WHY THIS APPROACH

Complexity: lightweight
Justification: The off branch is a real acceptance rule. One boolean on the existing settings object is enough.

## SANDWICH CONTEXT

[CRITICAL: The default is on. Do not create a new settings architecture.]
You are implementing the completion refresh preference.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: refresh is on unless the user turns it off.
Files in scope: settings.ts, appearance-section.tsx, AppPreferences.tsx, and the new test.
Available after: T6
Architecture rule: follow the existing settings row pattern.
[RESTATE: The default is on. Do not create a new settings architecture.]

## DELIVERABLE

[derived] Given a new profile, When settings load, Then automatic completion refresh is on
Given the appearance section, When it renders with the default, Then the refresh switch is on
Given the settings value is false, When the scheduler sees a draft change, Then no replacement starts until the next explicit request

## QUALITY BAR

Must-have:
  - The scheduler reads this setting rather than a second flag

Must-not-have:
  - A force-raw preference
  - A rewrite of AppPreferences

Open question risks:
  - The assumed default is wrong → the field stays, and only the default literal changes.

Rollback note:
  - Removing the field and treating a missing value as true matches the spec default.

Red flags:
  - Default false → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes and the T6 refresh-off test still passes
Uncertain when: old persisted settings without the field crash the store
Escalate when: the preference cannot be added without splitting `AppPreferences.tsx`

---

### Task 11: Route palette, assist, and mobile input through the editor [depends: T7]

## OBJECTIVE

When the draft store says Editor, history insert, assist insert, and the mobile bar edit the draft. They do not call `terminalApi.write`. A view-only session never mounts the composer. PTY mode keeps today's writes except for a view-only client, which writes nothing.

Steps:
1. Write failing test for: assist text inserts into the editor
   Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
   Level: unit
   Test intent: Given Editor mode and assist text with no control character, When insert is requested, Then the draft gains that text and `terminalApi.write` is not called
   Exercise through: `routeForeignInsert`
   Test doubles: a write spy and the draft store. Do not mock the router
   Expected RED: the router calls `terminalApi.write`
2. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "assist text inserts into the editor"`
3. Implement `routeForeignInsert` for assist text. Call it from `insertAssistCommand` with a small branch. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(terminal): insert assist text into the editor"`
4. Write failing test for: history text inserts into the editor
   Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
   Level: unit
   Test intent: Given Editor mode and history text with no control character, When insert is requested, Then the draft becomes that text, submit is not called, and `terminalApi.write` is not called
   Exercise through: `routeForeignInsert`
   Test doubles: a write spy, a submit spy, and the draft store
   Expected RED: history insert calls `terminalApi.write`
5. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "history text inserts into the editor"`
6. Route history insert. Call it from `handleInsertCommand` with a small branch. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(terminal): insert history text into the editor"`
7. Write failing test for: the mobile bar edits the draft
   Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
   Level: unit
   Test intent: Given Editor mode, When the mobile bar sends an arrow, Tab, Esc, Ctrl+C, or a paste, Then the editor key or paste path runs and `terminalApi.write` is not called
   Exercise through: `routeMobileKey`
   Test doubles: a write spy
   Expected RED: the mobile bar calls `terminalApi.write`
8. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "mobile bar edits the draft"`
9. Route those keys. Call `routeMobileKey` from `MobileTerminalControls` with a small branch. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(terminal): route the mobile bar through the editor"`
10. Write failing test for: assist control characters are refused
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given Editor mode and assist text that contains a control character, When insert is requested, Then the draft is unchanged and write is not called
    Exercise through: `routeForeignInsert`
    Test doubles: a write spy
    Expected RED: the control character is inserted
11. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "assist control characters are refused"`
12. Refuse that insert. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(terminal): refuse assist text that contains a control character"`
13. Write failing test for: history control characters are refused
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given Editor mode and history text that contains a newline, When insert is requested, Then the draft is unchanged and write is not called
    Exercise through: `routeForeignInsert`
    Test doubles: a write spy
    Expected RED: the newline is inserted into the draft
14. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "history control characters are refused"`
15. Refuse that insert. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(terminal): refuse history text that contains a newline"`
16. Write failing test for: a view-only client does not complete or submit
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given a view-only session, When the surface is asked to mount, When Tab or Enter is pressed, and When a mobile key arrives, Then the composer is absent, `terminalApi.write` is not called, and `requestCompletion` is not called
    Exercise through: `TerminalInputSurface` and `routeMobileKey`
    Test doubles: a write spy, a completion spy, and `allowWrite: false`
    Expected RED: the view-only client calls `requestCompletion` or write
17. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "view-only client does not complete or submit"`
18. Refuse view-only mount, write, and completion. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(terminal): keep viewers off the command editor"`
19. Write failing test for: page keys scroll and do not write
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given Editor mode, When Page Up or Page Down is pressed on the mobile bar, Then xterm scrolls and `terminalApi.write` is not called
    Exercise through: `routeMobileKey`
    Test doubles: a scroll spy and a write spy
    Expected RED: the page key calls `terminalApi.write`
20. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "page keys scroll and do not write"`
21. Scroll xterm. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(terminal): scroll the terminal from the mobile page keys"`
22. Write failing test for: a writable PTY keeps today's write
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given a writable PTY mode, When a mobile key arrives, Then today's direct `terminalApi.write` still happens
    Exercise through: `routeMobileKey`
    Test doubles: a write spy
    Expected RED: the PTY key is swallowed by the editor router
23. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "writable PTY keeps today's write"`
24. Keep the direct write. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(terminal): keep direct writes while the editor is unmounted"`
25. Write failing test for: PTY assist still refuses control characters
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given PTY mode and assist text that contains a control character, When insert is requested, Then today's refusal still applies and the text is not written
    Exercise through: `routeForeignInsert`
    Test doubles: a write spy
    Expected RED: the control character is written to the PTY
26. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "PTY assist still refuses control characters"`
27. Keep that refusal. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(terminal): keep the PTY control-character refusal"`
28. Write failing test for: a view-only PTY sends nothing
    Test file: `src/renderer/components/terminal/command-editor/foreign-insert.test.ts`
    Level: unit
    Test intent: Given a view-only client and PTY mode, When a mobile key arrives, Then `terminalApi.write` is not called and `requestCompletion` is not called
    Exercise through: `routeMobileKey`
    Test doubles: a write spy, a completion spy, and `allowWrite: false`
    Expected RED: the PTY branch still writes for the view-only client
29. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/foreign-insert.test.ts -t "view-only PTY sends nothing"`
30. Check `allowWrite` before the PTY write. Re-run step 29 and verify PASS. Refactor while green. Commit: `git commit -m "feat(terminal): keep a view-only PTY from writing"`

## REFERENCES LOADED

Spec — editor isolation, remote authority, mobile rule
src/renderer/components/terminal/ConnectedTerminal.tsx — `insertAssistCommand` at line 416
src/renderer/layouts/workspace-layout/use-command-palette-data.ts — `handleInsertCommand` at line 44
src/renderer/components/mobile/MobileTerminalControls.tsx — `write` at line 40

## WHY THIS APPROACH

Complexity: standard
Justification: Three existing call sites share one decision. A new router keeps each call site to a branch.

## SANDWICH CONTEXT

[CRITICAL: Editor mode does not use the PTY keystroke path. A viewer sends no bytes.]
You are implementing foreign insert and mobile routing for the command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. The same authority governs keys and completion.
Files in scope: foreign-insert.ts, its test, and the three call-site branches.
Available after: T7 draft store and surface
Architecture rule: PTY mode for a writable client keeps today's writes.
[RESTATE: Editor mode does not use the PTY keystroke path. A viewer sends no bytes.]

## DELIVERABLE

Given Editor mode, When assist text is clean, Then it is inserted and `terminalApi.write` is not called
Given Editor mode, When history text is clean, Then it is inserted and the command does not run
Given Editor mode, When the mobile bar sends an editing key, Then `terminalApi.write` is not called
Given a control character in assist or history text, When insert is requested in Editor, Then the buffer is unchanged
Given a view-only client, When it mounts, presses Tab or Enter, or uses the mobile bar, Then the composer is absent, nothing is written, and `requestCompletion` is not called
Given Editor mode, When Page Up or Page Down is pressed, Then xterm scrolls and nothing is written
Given a writable PTY session, When the mobile bar sends a key, Then `terminalApi.write` is still used
Given PTY mode and assist text with a control character, When insert is requested, Then it is still refused

## QUALITY BAR

Must-have:
  - PTY-mode control-character refusal for assist stays in place
  - The palette branch checks the draft store for the active terminal

Must-not-have:
  - A shared caret between two writable clients
  - A rewrite of the command palette

Open question risks:
  - none identified

Rollback note:
  - The branches can call the old write path when the store says the editor is absent.

Red flags:
  - Editor paste or mobile Tab calling `terminalApi.write` → STOP
  - A viewer write → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes
Uncertain when: the palette has no active terminal id to look up in the store
Escalate when: routing requires splitting `ConnectedTerminal.tsx` or `MobileTerminalControls.tsx`

---

### Task 12: Prove the reader delivery and submit binder [depends: T2, T3, T5, T6, T7, T8, T9, T11] [test-risk]

## OBJECTIVE

Prove the seams that earlier tasks leave unconnected. T2 returns frames to its caller and does not call `SessionGate`. T6 calls an injected host. This task adds `process_reader_chunk` and `bindSessionClient`.

Steps:
1. Write failing test for: the reader delivers a frame to the gate
   Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
   Level: integration
   Test intent: Given a reader chunk that contains terminal text plus a 9477 completion frame, When `process_reader_chunk` runs, Then the pending output contains the text and no 9477 payload, the real `SessionGate` waiter receives the parsed result, and `reader_loop` in `spawn.rs` calls `process_reader_chunk`
   Exercise through: `process_reader_chunk` in `deliver.rs`, the real `SessionFilter`, and the real gate waiter
   Test doubles: an in-memory pending buffer. Do not mock the filter, the parser, or `SessionGate`
   Expected RED: `process_reader_chunk` does not exist, so the test does not compile
2. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::reader_delivers_a_frame_to_the_gate`
3. Add `process_reader_chunk` and replace the inline filter-plus-push in `reader_loop` with one call to it. The test reads `spawn.rs` and fails when that call is missing. Also construct a new `SessionFilter` on every call, so a frame split across two calls is forwarded and never delivered, so the next test fails. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(pty): deliver session frames from the reader to the gate"`
4. Write failing test for: a split frame is delivered once
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given one 9477 frame split across two calls on the same reader state, When `process_reader_chunk` runs for each half, Then the pending output contains no protocol bytes and the real gate waiter receives that frame once, on the second call
    Exercise through: `process_reader_chunk`, one `SessionFilter` kept for that reader, and the real gate waiter
    Test doubles: an in-memory pending buffer. Do not mock the filter or `SessionGate`
    Expected RED: the split bytes are forwarded and the waiter does not receive the frame
5. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::split_frame_is_delivered_once`
6. Keep one `SessionFilter` for that reader across calls. Deliver the frame on the call that completes it. Do not forward the partial bytes. Re-run steps 2 and 5 and verify PASS. Commit: `git commit -m "feat(pty): deliver a session frame split across reads"`
7. Write failing test for: Enter is one ordered submit
   Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
   Level: integration
   Test intent: Given the real `submitDraft`, the real web transport over a fake socket, and `bindSessionClient`, When Enter is pressed during an active completion, Then the socket frames are cancel, then one submit of the draft, and no `write` frame
   Exercise through: `bindSessionClient`
   Test doubles: a fake socket and a host double that records order. Do not mock `submitDraft`, the transport, or `bindSessionClient`
   Expected RED: `bindSessionClient` is not exported, so the test does not compile
8. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "Enter is one ordered submit"`
9. Create `bind-session.ts` and export `bindSessionClient` for the accepted path only. Wire cancel before the one submit. Pass an `addCommand` function into `submitDraft`. Treat `Partial` as `Accepted` and record a committed submit for that result, so the next test fails. Also treat `PreWrite` as `Accepted` and record a committed submit for it. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(terminal): bind submit to the live session transport"`
10. Write failing test for: a partial result stays down across the binder
   Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
   Level: integration
   Test intent: Given `bindSessionClient` and the real web transport, When the transport returns `Partial`, Then history is not called, the editor stays unmounted, and the session has no committed submit
   Exercise through: `bindSessionClient`
   Test doubles: a fake socket whose reply is `Partial`, and a history recorder. Do not mock `bindSessionClient` or the transport
   Expected RED: the binder treats `Partial` as `Accepted`, so history is called and a committed submit is recorded
11. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "partial result stays down across the binder"`
12. Map `Partial` through the binder to the reducer result that stays out of Editor, skips history, and clears the committed-submit flag. Leave `PreWrite` treated as `Accepted`, including its committed-submit flag. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(terminal): keep a partial submit down across the session binder"`
13. Write failing test for: a pre-write result restores the draft across the binder
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given `bindSessionClient` and the real web transport, When the transport returns `PreWrite` for "make test" on a trusted prompt, Then the session store returns to Editor with "make test", history is not called, and no committed submit is recorded
    Exercise through: `bindSessionClient` and `sessionStore`
    Test doubles: a fake socket whose reply is `PreWrite`, and a history recorder. Do not mock the binder, the transport, or `sessionStore`
    Expected RED: `sessionStore` does not exist, so the test does not compile
14. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "pre-write result restores the draft across the binder"`
15. Add `session-store.ts`. Map `PreWrite` through the binder into that store so Editor holds "make test", history is not called, and the committed-submit flag stays unset. Also leave that flag unset after `Accepted`, so the next test fails. Do not handle PromptReady in this step. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(terminal): restore the draft after a pre-write failure"`
16. Write failing test for: an accepted submit records the committed flag
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given `bindSessionClient`, the real `submitDraft`, and no committed submit yet, When the transport returns `Accepted`, Then the session store records a committed submit. The test does not set that flag itself
    Exercise through: `bindSessionClient` and `sessionStore`
    Test doubles: a fake socket whose reply is `Accepted`. Do not mock the binder, `submitDraft`, or `sessionStore`
    Expected RED: `Accepted` leaves the committed-submit flag unset
17. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "accepted submit records the committed flag"`
18. Record the committed-submit flag only for `Accepted`. Add `sessionStore.apply` and ignore PromptReady, so the next test fails. Re-run step 17 and verify PASS. Commit: `git commit -m "feat(terminal): record a committed submit after acceptance"`
19. Write failing test for: a prompt event opens the editor through the session store
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given InitShell registered session_id "s1" and nonce "n1", When `sessionStore.apply` then receives PromptReady for "s1" with nonce "n1", `line_empty`, and `prompt_suppressed`, Then `reduceSession` runs and the stored mode is still Editor after the caller returns
    Exercise through: `sessionStore.apply`
    Test doubles: none. Do not mock `reduceSession`
    Expected RED: PromptReady is ignored, so the mode stays raw
20. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "prompt event opens the editor"`
21. `apply` opens Editor for every PromptReady, including a session_id that InitShell never registered. Ignore CommandStarted and CommandFinished, so the next tests fail. Do not wire the web or Tauri adapters in this step. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(terminal): keep session mode across an unmount"`
22. Write failing test for: an unregistered prompt does not open the editor
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given no InitShell for session_id "s2", When `sessionStore.apply` receives PromptReady for "s2", Then the mode stays raw
    Exercise through: `sessionStore.apply`
    Test doubles: none. Do not mock `reduceSession`
    Expected RED: the unregistered prompt opens Editor
23. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "unregistered prompt does not open the editor"`
24. Require an InitShell for that session_id before PromptReady opens Editor. Leave CommandStarted ignored, so the next test fails. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(terminal): register a session before opening the editor"`
25. Write failing test for: a user command leaves the editor
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given InitShell registered "s1", the mode is Editor, and `bindSessionClient` has just returned `Accepted` for that session, When `sessionStore.apply` receives CommandStarted for "s1", Then the mode leaves Editor. The test does not set the committed-submit flag except through that `Accepted` result
    Exercise through: `sessionStore.apply`
    Test doubles: none. Do not mock `reduceSession`
    Expected RED: CommandStarted leaves the mode in Editor
26. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "user command leaves the editor"`
27. Apply CommandStarted through `reduceSession` and pass the session's committed-submit flag. Do not edit `transition.ts`. The T5 reducer leaves Editor only when that flag is set. Leave the stored exit code unchanged when CommandFinished arrives, so the next test fails. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(terminal): leave the editor when a user command starts"`
28. Write failing test for: command finished records the exit code
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given the registered session is out of Editor, When `sessionStore.apply` receives CommandFinished for "s1" with code 2, Then last exit code is 2 and the mode stays out of Editor
    Exercise through: `sessionStore.apply`
    Test doubles: none. Do not mock `reduceSession`
    Expected RED: the exit code stays empty
29. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "command finished records the exit code"`
30. Apply CommandFinished through `reduceSession`. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(terminal): record the exit code from the session event"`
31. Write failing test for: the surface reads the session store
    Test file: `src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx`
    Level: integration
    Test intent: Given `ConnectedTerminal.tsx`, When its source is read, Then it passes the mode from `sessionStore` into `TerminalInputSurface` and does not declare a second input-mode `useState`
    Exercise through: a source read of `ConnectedTerminal.tsx`
    Test doubles: none
    Expected RED: `sessionStore` does not appear in `ConnectedTerminal.tsx`
32. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/TerminalInputSurface.test.tsx -t "surface reads the session store"`
33. Add that one read and one prop. Do not move other helpers. Re-run step 32 and verify PASS. Commit: `git commit -m "feat(terminal): render the editor from the session store"`
34. Write failing test for: Enter uses the session binder
    Test file: `src/renderer/components/terminal/command-editor/CommandEditor.test.tsx`
    Level: integration
    Test intent: Given `ConnectedTerminal.tsx` and `CommandEditor.tsx`, When their sources are read, Then CommandEditor calls `useBindSessionClient` from `bind-session.ts`, that hook calls `useAddCommand()` once with no arguments, and it passes the returned function into `submitDraft`. CommandEditor does not call `terminalApi.write`
    Exercise through: a source read of those three files
    Test doubles: none
    Expected RED: CommandEditor does not call `useBindSessionClient`
35. Run test — verify FAIL: `bun run test -- src/renderer/components/terminal/command-editor/CommandEditor.test.tsx -t "Enter uses the session binder"`
36. Pass `useBindSessionClient` into CommandEditor. Inside that hook, call `useAddCommand()` once and pass the returned function to `submitDraft`. Also call that function again after `submitDraft` returns, so the next test fails. Do not add a second trim. Re-run step 35 and verify PASS. Commit: `git commit -m "feat(terminal): submit the editor through the session binder"`
37. Write failing test for: one enter records one history entry
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given `useBindSessionClient`, the real `submitDraft`, and `Accepted` for "git sta", terminal name "local", terminal id "t1", and project id "p1", When Enter runs, Then `useCommandHistoryStore` contains exactly one matching entry
    Exercise through: `useBindSessionClient` and `submitDraft`
    Test doubles: a fake socket whose reply is `Accepted`. Mock only `persistenceApi.write`. Do not mock `useAddCommand`, `submitDraft`, or the store. Render a component so the hook can run
    Expected RED: the store contains two entries
38. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "one enter records one history entry"`
39. Delete the extra call. `submitDraft` is the only call to the returned function. Also call `term.write` with the draft after `Accepted`, so the next test fails. Re-run step 38 and verify PASS. Commit: `git commit -m "feat(terminal): record one history entry for a bound submit"`
40. Write failing test for: accepted enter does not paint the command
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given `useBindSessionClient`, the real `submitDraft`, and `Accepted` for "git remote -v", When Enter runs, Then the xterm write spy receives zero calls containing that command
    Exercise through: `bindSessionClient` and the Enter path
    Test doubles: a fake socket whose reply is `Accepted`, and an xterm write spy. Do not mock `bindSessionClient`, `submitDraft`, or `handleAcceptedSubmit`
    Expected RED: `term.write` receives "git remote -v"
41. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "accepted enter does not paint the command"`
42. Do not call `term.write` with the draft. `submitDraft` remains the only caller of the history function. Re-run steps 38 and 41 and verify PASS. Commit: `git commit -m "feat(terminal): leave the bound command echo to the shell"`
43. Write failing test for: completion uses the session binder
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given `bind-session.ts`, When its source is read, Then `CompletionScheduler` requests go through `requestCompletion` on the transport and not through `terminalApi.write`
    Exercise through: a source read of `bind-session.ts`
    Test doubles: none
    Expected RED: `bind-session.ts` does not mention `requestCompletion`
44. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "completion uses the session binder"`
45. Wire the scheduler to `requestCompletion`. Also send a `write` frame for that settled edit, so the next test fails. Re-run step 44 and verify PASS. Commit: `git commit -m "feat(terminal): request completion through the session binder"`
46. Write failing test for: a settled edit requests completion on the socket
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given `CompletionScheduler`, `bindSessionClient`, and the real web transport on a fake socket, When one edit settles, Then the socket sends one `command_editor_request_completion` and no `write` frame
    Exercise through: `bindSessionClient` and `CompletionScheduler`
    Test doubles: a fake socket and a fake clock. Do not mock the scheduler, `bindSessionClient`, or the transport
    Expected RED: the settled edit sends a `write` frame or does not send `command_editor_request_completion`
47. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "settled edit requests completion on the socket"`
48. Send one `command_editor_request_completion` and no `write` frame. After Enter's submit, send another completion request, so the next test fails. Do not re-run step 8 in this step. Re-run step 47 and verify PASS. Commit: `git commit -m "feat(terminal): request completion on the session socket"`
49. Write failing test for: enter does not leave a completion frame
    Test file: `src/renderer/lib/command-lifecycle/submit-integration.test.ts`
    Level: integration
    Test intent: Given the scheduler's completion request is in flight on the fake socket, When Enter runs through `bindSessionClient`, Then the socket sends cancel, then one submit, and no completion frame after that submit
    Exercise through: `bindSessionClient` and `CompletionScheduler`
    Test doubles: a fake socket. Do not mock the scheduler, `bindSessionClient`, or `submitDraft`
    Expected RED: a completion frame follows the submit
50. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/submit-integration.test.ts -t "enter does not leave a completion frame"`
51. Do not send a completion frame after the submit. The socket order stays cancel, then one submit. Re-run steps 8, 47, and 50 and verify PASS. Commit: `git commit -m "feat(terminal): stop completion when the draft is submitted"`
52. Write failing test for: a prompt frame is a lifecycle event
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given a PromptReady frame for session_id "s1", When `process_reader_chunk` runs, Then `TerminalEventHub` receives one `SessionLifecycle` event that carries "s1" and has no completion payload
    Exercise through: `process_reader_chunk` and `TerminalEventHub`
    Test doubles: an in-memory hub subscriber. Do not mock the filter or the hub
    Expected RED: PromptReady is only returned to the caller and no `SessionLifecycle` variant exists
53. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::prompt_frame_is_a_lifecycle_event`
54. Add `TerminalEvent::SessionLifecycle` in `terminal_events.rs` with session_id, kind, nonce, `prompt_suppressed`, `line_empty`, and an optional exit code. Update the `terminal_id()` match. Publish it from `process_reader_chunk` for PromptReady. Also publish it for a completion frame, so the next test fails. Do not add the web or Tauri client case in this step. Re-run step 53 and verify PASS. Commit: `git commit -m "feat(terminal): publish session lifecycle on the existing event hub"`
55. Write failing test for: a completion frame is not a lifecycle event
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given a completion frame, When `process_reader_chunk` runs, Then the gate waiter receives the result and the hub receives no `SessionLifecycle` event
    Exercise through: `process_reader_chunk` and `TerminalEventHub`
    Test doubles: an in-memory hub subscriber. Do not mock the filter, the hub, or `SessionGate`
    Expected RED: the completion frame publishes `SessionLifecycle`
56. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::completion_frame_is_not_a_lifecycle_event`
57. Return the completion frame only to the gate waiter. Re-run step 56 and verify PASS. Commit: `git commit -m "feat(pty): keep a completion frame off the event hub"`
58. Write failing test for: desync restores the prompt and not the draft
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given a `Desync` from the real filter and a draft "git status", When `process_reader_chunk` handles it, Then the gate is raw, the writer receives `RESTORE_PROMPT_SEQUENCE` through `SessionGate::restore_prompt`, the writer does not receive "git status", and the hub receives one `SessionLifecycle` Desync event whose payload does not contain "git status"
    Exercise through: `process_reader_chunk`
    Test doubles: an in-memory writer. Do not mock the filter
    Expected RED: Desync does not write the restore sequence
59. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::desync_restores_the_prompt`
60. Call `SessionGate::restore_prompt` and mark the gate raw. Emit `SessionLifecycle` Desync with no draft. Re-run steps 2, 53, 56, and 59. Verify PASS. Commit: `git commit -m "feat(pty): restore the shell prompt when the session desyncs"`
61. Write failing test for: the web adapter applies session lifecycle
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given a `session_lifecycle` payload, When the web client handles it, Then `web-terminal-protocol.types.ts` names `session_lifecycle` and `handleEvent` in `web-terminal-api.ts` calls `sessionStore.apply`. The case does not mention a completion payload
    Exercise through: a source read of those two files
    Test doubles: none
    Expected RED: `session_lifecycle` is absent from the payload union and `handleEvent` does not call `sessionStore.apply`
62. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "web adapter applies session lifecycle"`
63. Add the `session_lifecycle` variant on `WebTerminalEventPayload` and the one `handleEvent` case. Forward only `PromptReady` and `Desync` to `sessionStore.apply`, so a later kind is ignored. Do not add the Tauri listen, so the next test fails. Re-run step 62 and verify PASS. Commit: `git commit -m "feat(web): deliver session lifecycle to the session store"`
64. Write failing test for: the desktop adapter applies session lifecycle
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given a desktop session event, When the Tauri client listens, Then `tauri-terminal-api.ts` has one listen beside `terminal-cwd-changed` that calls `sessionStore.apply` and does not mention a completion payload
    Exercise through: a source read of `tauri-terminal-api.ts`
    Test doubles: none
    Expected RED: the listen is absent
65. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "desktop adapter applies session lifecycle"`
66. Add that listen and forward only `PromptReady` and `Desync`, so a later kind is ignored. Re-run step 65 and verify PASS. Commit: `git commit -m "feat(desktop): deliver session lifecycle to the session store"`
67. Write failing test for: desync leaves the editor and asks for the prompt
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given Editor holds draft "git status", When `sessionStore.apply` receives `SessionLifecycle` Desync, Then the stored mode is raw, the draft remains stored, `submitCommand` is not called, and `restorePrompt` is called once with no draft argument
    Exercise through: `sessionStore.apply`
    Test doubles: spies for `submitCommand` and `restorePrompt`. Do not mock `reduceSession` or `sessionStore`
    Expected RED: apply ignores Desync, so the mode stays Editor and `restorePrompt` is not called
68. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "desync leaves the editor and asks for the prompt"`
69. Call `reduceSession` for Desync and call `restorePrompt` once. Do not pass the draft and do not call `submitCommand`. A second restore sequence is idempotent: the shell binding sets the saved prompt again and does not run the draft. Re-run steps 2, 20, 53, 56, 59, 62, 65, and 68. Verify PASS. Commit: `git commit -m "feat(terminal): leave the editor when the session desyncs"`

70. Write failing test for: generator bytes do not reach the client
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given `SessionGate::generator_active` is true and the chunk contains "COMPGEN leak" plus a 9477 result, When `process_reader_chunk` runs, Then the pending client output does not contain "COMPGEN leak" and the gate waiter receives the result
    Exercise through: `process_reader_chunk` and the real `SessionGate`
    Test doubles: an in-memory pending buffer. Do not mock the filter or the gate
    Expected RED: "COMPGEN leak" is forwarded to the client
71. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::generator_bytes_do_not_reach_the_client`
72. While `generator_active` is true, drop non-9477 client bytes and still deliver the 9477 frame. Also keep dropping those bytes after the slot is free, so the next test fails. Re-run step 71 and verify PASS. Commit: `git commit -m "feat(pty): hide completion output from the terminal"`
73. Write failing test for: user output returns after the generator
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given the generator slot is free, When a later chunk contains "user output", Then that text is forwarded to the client
    Exercise through: `process_reader_chunk`
    Test doubles: an in-memory pending buffer. Do not mock the filter or the gate
    Expected RED: "user output" is dropped
74. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::user_output_returns_after_the_generator`
75. Forward bytes when the slot is free. Re-run steps 2, 53, 56, 59, 71, and 74. Verify PASS. Commit: `git commit -m "feat(pty): show shell output after completion finishes"`

76. Write failing test for: the hub publishes InitShell before the prompt
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given a reader chunk with InitShell for session_id "s1" and nonce "n1", then PromptReady for "s1", When `process_reader_chunk` runs, Then the hub receives those two `SessionLifecycle` events in that order and neither payload is a completion result
    Exercise through: `process_reader_chunk` and `TerminalEventHub`
    Test doubles: an in-memory hub subscriber. Do not mock the filter or the hub
    Expected RED: InitShell is not published
77. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::hub_publishes_initshell_before_the_prompt`
78. Publish InitShell from `process_reader_chunk`. Do not publish CommandStarted or CommandFinished in this step. Also make `sessionStore.apply` open Editor for a PromptReady that has no prior InitShell, so the next test fails. Re-run step 77 and verify PASS. Commit: `git commit -m "feat(pty): publish init shell on the session hub"`
79. Write failing test for: a prompt without init shell stays raw
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given no InitShell hub event for session_id "s2", When `sessionStore.apply` receives the PromptReady hub event for "s2", Then the mode stays raw
    Exercise through: `sessionStore.apply`
    Test doubles: none. Do not mock `reduceSession`
    Expected RED: PromptReady alone opens Editor
80. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "prompt without init shell stays raw"`
81. Require a prior InitShell for that session_id. Re-run steps 23 and 80 and verify PASS. Commit: `git commit -m "feat(terminal): wait for init shell before opening the editor"`
82. Write failing test for: the hub command start leaves the editor
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs` and `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given a CommandStarted frame for session_id "s1" while that session is in Editor after `bindSessionClient` returned `Accepted`, When `process_reader_chunk` publishes it and the web and Tauri adapters handle that hub event, Then the mode leaves Editor. The test does not set the committed-submit flag except through that `Accepted` result
    Exercise through: `process_reader_chunk`, `handleEvent` in `web-terminal-api.ts`, and the listen in `tauri-terminal-api.ts`
    Test doubles: an in-memory hub subscriber on the Rust side. Do not mock the filter, the hub, `sessionStore`, or `reduceSession`
    Expected RED: CommandStarted is not published, and both adapters ignore that kind
83. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::hub_command_started_leaves_the_editor` and `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "hub command start leaves the editor"`
84. Publish CommandStarted. Remove the PromptReady-and-Desync-only filter from the web case and the Tauri listen, so every `SessionLifecycle` kind reaches `sessionStore.apply`. Do not publish CommandFinished in this step. Also drop InitShell before it reaches `sessionStore.apply`, so the next test fails. Re-run step 83 and verify PASS. Commit: `git commit -m "feat(terminal): deliver a command start from the session hub"`
85. Write failing test for: init shell then prompt opens the editor through the adapters
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given InitShell for session_id "s1" and nonce "n1", then PromptReady for "s1" with that nonce, `line_empty`, and `prompt_suppressed`, When `handleEvent` and the Tauri listen each apply those hub events, Then the stored mode is Editor
    Exercise through: `handleEvent` in `web-terminal-api.ts` and the listen in `tauri-terminal-api.ts`
    Test doubles: none. Do not mock `sessionStore` or `reduceSession`
    Expected RED: PromptReady through the adapters stays raw because InitShell was dropped
86. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "init shell then prompt opens the editor through the adapters"`
87. Forward InitShell to `sessionStore.apply` from both adapters. Re-run steps 83 and 86 and verify PASS. Commit: `git commit -m "feat(terminal): open the editor after init shell reaches the adapters"`
88. Write failing test for: the hub records the finished exit code
    Test file: `src-tauri/src/pty/session_protocol/isolation_tests.rs`
    Level: integration
    Test intent: Given a CommandFinished frame for session_id "s1" with exit code 2, When `process_reader_chunk` runs, Then the hub receives one `SessionLifecycle` CommandFinished whose payload carries "s1" and exit code 2
    Exercise through: `process_reader_chunk` and `TerminalEventHub`
    Test doubles: an in-memory hub subscriber. Do not mock the filter or the hub
    Expected RED: CommandFinished is not published
89. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::isolation_tests::hub_records_the_finished_exit_code`
90. Publish CommandFinished with its exit code. Also make both adapters ignore CommandFinished, so the next test fails. Re-run steps 23, 77, 80, 83, and 89. Verify PASS. Commit: `git commit -m "feat(terminal): deliver a command exit code from the session hub"`

91. Write failing test for: the adapters record the finished exit code
    Test file: `src/renderer/lib/command-lifecycle/session-store.test.ts`
    Level: integration
    Test intent: Given the registered session is out of Editor, When `handleEvent` and the Tauri listen each apply CommandFinished for "s1" with exit code 2, Then the last exit code is 2 and the mode stays out of Editor
    Exercise through: `handleEvent` in `web-terminal-api.ts` and the listen in `tauri-terminal-api.ts`
    Test doubles: none. Do not mock `sessionStore` or `reduceSession`
    Expected RED: the adapters ignore CommandFinished, so the exit code stays empty
92. Run test — verify FAIL: `bun run test -- src/renderer/lib/command-lifecycle/session-store.test.ts -t "adapters record the finished exit code"`
93. Let both adapters apply CommandFinished. Re-run steps 89 and 92 and verify PASS. Refactor while green. Commit: `git commit -m "feat(terminal): record the exit code from both session adapters"`
## REFERENCES LOADED

Spec — exactly-once echo, protocol isolation, Enter during pending completion
T2 filter, T3 gate, T6 submitDraft, T9 transport
`@xterm/xterm@6.1.0-beta.287` — `term.write` is display-only and must not be used to duplicate the shell echo

## WHY THIS APPROACH

Complexity: standard
Justification: Each side can pass alone while the socket order or the forwarded bytes are still wrong. This task is the collaboration check.

## SANDWICH CONTEXT

[CRITICAL: Protocol bytes stay out of the client stream. Enter cancels completion before one submit.]
You are proving the reader-to-gate seam and the submit binder.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. The shell echoes the one host write.
Files in scope: deliver.rs, isolation_tests.rs, one call in spawn.rs `reader_loop`, bind-session.ts, submit-integration.test.ts, session-store.ts, session-store.test.ts, one `SessionLifecycle` variant in terminal_events.rs, one `session_lifecycle` variant in web-terminal-protocol.types.ts, one case or listen in each terminal adapter, one `sessionStore` read in ConnectedTerminal.tsx, and the Enter prop passed into CommandEditor.tsx. `useBindSessionClient` calls `useAddCommand()` once and passes the returned function into `submitDraft`. `submitDraft` is the only caller of that function. Do not edit the hook. Do not edit `transition.ts`.
Available after: T2, T3, T5, T6, T7, T8, T9, T11
Architecture rule: do not add an end-to-end browser harness in this task.
[RESTATE: Protocol bytes stay out of the client stream. Enter cancels completion before one submit.]

## DELIVERABLE

Given a reader chunk with terminal text and a 9477 completion frame, When `process_reader_chunk` runs from `reader_loop`, Then the pending output keeps the text, drops the 9477 payload, and the real gate waiter receives the result
Given one 9477 frame split across two reader calls on the same reader state, When `process_reader_chunk` runs, Then no protocol bytes are forwarded and the gate waiter receives that frame once, on the second call
Given a pending completion, When Enter is pressed through `bindSessionClient`, Then cancel precedes one submit of the draft and no keystroke write is sent
[must-not] Given `Partial`, When `bindSessionClient` finishes, Then history is not written and Editor stays unmounted
Given `PreWrite` for "make test" on a trusted prompt, When `bindSessionClient` finishes, Then the session store returns to Editor with "make test" and history is not called
Given a PromptReady `session_lifecycle` event, When `sessionStore.apply` handles it, Then the stored mode is still Editor after the caller returns
Given a PromptReady frame, When `process_reader_chunk` runs, Then the hub receives one `SessionLifecycle` event with that session_id and no completion payload
Given no InitShell for a session, When PromptReady is applied, Then the mode stays raw
Given `bindSessionClient` returns `Accepted`, When that result is stored, Then the session records a committed submit and the test does not set the flag itself
Given that `Accepted` result and a registered session, When CommandStarted is applied, Then the mode leaves Editor
Given CommandFinished carries code 2, When the store applies it, Then last exit code is 2 and the mode stays out of Editor
Given `ConnectedTerminal` mounts the surface, When its source is read, Then the mode comes from `sessionStore`
Given CommandEditor handles Enter, When its source is read, Then the handler is `useBindSessionClient` and `terminalApi.write` is not called
Given `Accepted` for "git sta" through `useBindSessionClient`, When Enter finishes, Then the history store contains exactly one matching entry
Given `Accepted` for "git remote -v" through the Enter path, When that path finishes, Then the xterm write spy receives zero calls containing that command
Given InitShell then PromptReady for the same session, When `process_reader_chunk` publishes both, Then the hub emits those two events in order
Given a PromptReady hub event and no InitShell for that session, When the store applies it, Then the mode stays raw
Given a committed submit, When the hub publishes CommandStarted and the adapters apply it, Then the mode leaves Editor
Given InitShell and then PromptReady for the same session, When both adapters handle those hub events, Then the stored mode is Editor
Given a CommandFinished frame carries code 2, When `process_reader_chunk` publishes it, Then the hub payload carries that session_id and code 2
Given CommandFinished carries code 2, When both adapters apply that hub event, Then the last exit code is 2 and the mode stays out of Editor
Given `bind-session.ts` wires completion, When its source is read, Then the call is `requestCompletion` and not `terminalApi.write`
Given one settled edit, When the scheduler runs through `bindSessionClient`, Then the socket sends one `command_editor_request_completion` and no `write` frame
Given that completion request is in flight, When Enter runs, Then the socket sends cancel, then one submit, and no completion frame after the submit
Given a generator is active, When its echo arrives in a reader chunk, Then the client output does not contain that echo and the waiter receives the 9477 result
Given the generator slot is free, When later user output arrives, Then that output is forwarded
[must-not] Given a completion frame, When `process_reader_chunk` runs, Then the hub receives no `SessionLifecycle` event
Given a `Desync` and draft "git status", When `process_reader_chunk` handles it, Then the gate is raw, the writer receives the restore sequence and not the draft, and the hub receives one Desync event
Given a `session_lifecycle` payload, When the web client handles it, Then `handleEvent` calls `sessionStore.apply`
Given a desktop session event, When the Tauri client listens, Then the listen beside `terminal-cwd-changed` calls `sessionStore.apply`
Given Editor holds "git status", When `sessionStore.apply` receives Desync, Then the mode is raw, the draft stays stored, submit is not called, and `restorePrompt` is called once with no draft argument

## QUALITY BAR

Must-have:
  - The Rust test uses the real filter and the real gate waiter
  - The TypeScript test uses the real `submitDraft`, the real transport, and `bindSessionClient`
  - The Enter path through `bindSessionClient` proves the client does not paint the command
  - Desync calls `SessionGate::restore_prompt`. The client calls `restorePrompt` once. Neither call includes the draft. A second sequence only resets the saved prompt

Must-not-have:
  - A Playwright suite
  - A mocked filter, a mocked `submitDraft`, or a mocked `bindSessionClient`
  - Output-block assertions
  - A second live bash process. The handshake stays in T4

Open question risks:
  - The live bash glyph check belongs to T4. This task does not respawn a shell.

Rollback note:
  - These tests can stay red only until the seam is fixed. They do not add a user-facing flag.

Red flags:
  - Mocking the unit that the scenario is meant to prove → STOP
  - A `write` frame on Enter → STOP

## STOP CONDITIONS

Done when: the cargo isolation filter and both vitest filters named in this task pass
Uncertain when: the pending-buffer handoff cannot be called without starting a real PTY thread
Escalate when: the proof requires splitting `spawn.rs` to expose the reader

---

## Plan Summary

| Task | Name | Depends | Complexity | Key Verification |
|------|------|---------|------------|------------------|
| T1 | Define the session protocol contract | prereq | standard | Ident 9477 and rejected frames |
| T2 | Strip session protocol bytes in the PTY reader | T1 | standard | Split frames never reach clients |
| T3 | Serialize submit and completion on the host | T1 | standard | One draft write, one generator, viewer refused |
| T4 | Inject bash and zsh integration | T1, T2 | standard | Login bash handshakes. Other programs stay raw |
| T5 | Reduce authenticated shell events to an input mode | T1 | standard | Nonce-matched suppressed prompt opens Editor |
| T6 | Schedule completion and record submit on the client | T5 | standard | Repeated Tab stays. Enter submits the draft once |
| T7 | Mount the command editor and keep its draft | T5 | standard | Composer only in Editor. Draft survives remount |
| T8 | Accept a suggestion without running it | T6, T7 | standard | Tab inserts. Enter submits the buffer |
| T9 | Send submit and completion on both transports | T1, T3 | standard | Closed socket does not queue submit |
| T10 | Add the automatic completion refresh preference | T6 | lightweight | Default on. Off waits for Tab |
| T11 | Route palette, assist, and mobile input through the editor | T7 | standard | Editor inserts do not call `terminalApi.write` |
| T12 | Prove the reader delivery and submit binder | T2, T3, T5, T6, T7, T8, T9, T11 | standard | Reader delivers one frame to the gate. Cancel then one submit. Desync restores the prompt |
