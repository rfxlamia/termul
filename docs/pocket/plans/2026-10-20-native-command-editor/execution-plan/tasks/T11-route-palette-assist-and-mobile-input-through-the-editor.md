# Task T11 — Route palette, assist, and mobile input through the editor

**Phase:** 3
**Depends:** T7
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
