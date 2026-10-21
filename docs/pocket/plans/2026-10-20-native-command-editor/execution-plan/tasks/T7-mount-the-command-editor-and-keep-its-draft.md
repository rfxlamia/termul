# Task T7 — Mount the command editor and keep its draft

**Phase:** 2
**Depends:** T5
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
