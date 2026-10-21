# Task T3 — Serialize submit and completion on the host

**Phase:** 1
**Depends:** T1
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
