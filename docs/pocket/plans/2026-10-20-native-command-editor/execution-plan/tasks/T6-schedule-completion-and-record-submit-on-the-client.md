# Task T6 — Schedule completion and record submit on the client

**Phase:** 2
**Depends:** T5
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

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
