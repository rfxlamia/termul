# Task T10 — Add the automatic completion refresh preference

**Phase:** 3
**Depends:** T6
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

### Task 10: Add the automatic completion refresh preference [depends: T6]

## OBJECTIVE

Default automatic completion refresh to true and let the existing preferences page turn it off. The scheduler reads that value.

Steps:
1. Write failing test for: the default is on
   Test file: `src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx`
   Level: unit
   Test intent: Given `DEFAULT_APP_SETTINGS`, When `automaticCompletionRefresh` is read, Then it is true
   Exercise through: `DEFAULT_APP_SETTINGS`
   Test doubles: none
   Expected RED: the settings field does not exist
2. Run test — verify FAIL: `bun run test -- src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx -t "refresh defaults on"`
3. Add the field to `AppSettings` and `DEFAULT_APP_SETTINGS` next to `terminalScreenReaderMode`. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(settings): default automatic completion refresh to on"`
4. Write failing test for: the appearance switch renders on
   Test file: `src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx`
   Level: unit
   Test intent: Given the appearance section receives `automaticCompletionRefresh` true, When it renders, Then the switch is checked. When the user turns it off, Then the change handler is called with false
   Exercise through: rendering the appearance section
   Test doubles: a change handler. Do not mock the switch
   Expected RED: the section has no switch for this setting
5. Run test — verify FAIL: `bun run test -- src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx -t "appearance switch renders on"`
6. Add the switch beside the screen-reader row and pass the value through `AppPreferences.tsx`. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(settings): show the completion refresh switch"`
7. Write failing test for: the scheduler reads the setting
   Test file: `src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx`
   Level: unit
   Test intent: Given `automaticCompletionRefresh` is false on the settings object, When `CompletionScheduler` is constructed from that object and the draft changes with no generator active, Then no replacement is scheduled
   Exercise through: `CompletionScheduler`
   Test doubles: a fake host. Do not mock the settings object
   Expected RED: the scheduler still schedules from its own flag and ignores the settings field
8. Run test — verify FAIL: `bun run test -- src/renderer/pages/app-preferences/automatic-completion-refresh.test.tsx -t "scheduler reads the setting"`
9. Read `automaticCompletionRefresh` from settings. Re-run step 8 and the T6 refresh-off filter. Verify PASS. Refactor while green. Commit: `git commit -m "feat(settings): prefer automatic command completion"`

## REFERENCES LOADED

Spec — `automaticCompletionRefresh` defaults to true
src/renderer/types/settings.ts — `AppSettings` line 62 and the default near line 282
src/renderer/pages/app-preferences/appearance-section.tsx — screen-reader switch at lines 109–123
src/renderer/pages/AppPreferences.tsx — the prop passed near line 305

## WHY THIS APPROACH

Complexity: lightweight
Justification: The off branch is a real acceptance rule. One boolean on the existing settings object is enough.

## SANDWICH CONTEXT

[CRITICAL: The default is on. Do not create a new settings architecture.]
You are implementing the completion refresh preference.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: refresh is on unless the user turns it off.
Files in scope: settings.ts, appearance-section.tsx, AppPreferences.tsx, and the new test.
Available after: T6
Architecture rule: follow the existing settings row pattern.
[RESTATE: The default is on. Do not create a new settings architecture.]

## DELIVERABLE

[derived] Given a new profile, When settings load, Then automatic completion refresh is on
Given the appearance section, When it renders with the default, Then the refresh switch is on
Given the settings value is false, When the scheduler sees a draft change, Then no replacement starts until the next explicit request

## QUALITY BAR

Must-have:
  - The scheduler reads this setting rather than a second flag

Must-not-have:
  - A force-raw preference
  - A rewrite of AppPreferences

Open question risks:
  - The assumed default is wrong → the field stays, and only the default literal changes.

Rollback note:
  - Removing the field and treating a missing value as true matches the spec default.

Red flags:
  - Default false → STOP

## STOP CONDITIONS

Done when: every vitest filter named in this task passes and the T6 refresh-off test still passes
Uncertain when: old persisted settings without the field crash the store
Escalate when: the preference cannot be added without splitting `AppPreferences.tsx`
