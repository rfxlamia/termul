# Task T9 — Send submit and completion on both transports

**Phase:** 2
**Depends:** T1, T3
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
