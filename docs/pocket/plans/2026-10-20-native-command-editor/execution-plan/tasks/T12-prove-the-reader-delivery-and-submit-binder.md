# Task T12 — Prove the reader delivery and submit binder

**Phase:** 3
**Depends:** T2, T3, T5, T6, T7, T8, T9, T11
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
