# Task T1 — Define the session protocol contract

**Phase:** 1
**Depends:** none
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
