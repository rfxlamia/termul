# Task T2 — Strip session protocol bytes in the PTY reader

**Phase:** 1
**Depends:** T1
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
