# Task T5 — Reduce authenticated shell events to an input mode

**Phase:** 1
**Depends:** T1
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
