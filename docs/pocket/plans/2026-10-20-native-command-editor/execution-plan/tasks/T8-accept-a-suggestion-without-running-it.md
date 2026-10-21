# Task T8 — Accept a suggestion without running it

**Phase:** 3
**Depends:** T6, T7
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
