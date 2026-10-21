# Task T4 — Inject bash and zsh integration

**Phase:** 2
**Depends:** T1, T2
**Source plan:** ../../execution-plan.md

---

### Pocket Packet

### Task 4: Inject bash and zsh integration [depends: T1, T2] [test-risk]

## OBJECTIVE

Install original bash and zsh integration for a local interactive login shell. Every other program stays raw. The scripts speak the T1 frames and keep a generator out of the user prompt cycle.

Steps:
1. Write failing test for: bash and zsh are injected
   Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
   Level: unit
   Test intent: Given bash or zsh with a login arg and no custom program, When the injection plan is built, Then it includes the matching script and a fresh session id and nonce
   Exercise through: `injection_plan`
   Test doubles: none
   Expected RED: `injection_plan` does not exist
2. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::bash_and_zsh_are_injected`
3. Implement `injection_plan` so every program, including fish, dash, ssh, PowerShell, cmd, and a direct program, receives a script. Re-run step 2 and verify PASS. Commit: `git commit -m "feat(pty): choose bash and zsh for session integration"`
4. Write failing test for: other programs stay raw
   Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
   Level: unit
   Test intent: Given fish, dash, ssh, PowerShell, cmd, or a direct program, When the injection plan is built, Then it is raw and adds no script
   Exercise through: `injection_plan`
   Test doubles: none
   Expected RED: those programs still receive a script
5. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::other_programs_stay_raw`
6. Return raw for those programs. Re-run step 5 and verify PASS. Commit: `git commit -m "feat(pty): leave non-bash shells on the raw path"`
7. Write failing test for: a generator is not a user command
   Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
   Level: unit
   Test intent: Given the bash and zsh script sources, When they are scanned, Then a generator path sets an internal flag and does not emit user `CommandStarted`, and prompt frames carry the nonce and `prompt_suppressed`
   Exercise through: reading `src-tauri/resources/shell-integration/bash.sh` and `zsh.sh`
   Test doubles: none
   Expected RED: the script files do not exist
8. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::generator_is_not_a_user_command`
9. Write original scripts that flag the generator and emit the prompt fields. Do not add the clear-line binding or the bootstrap-failure path in this step. Re-run step 8 and verify PASS. Commit: `git commit -m "feat(pty): add bash and zsh session integration scripts"`
10. Write failing test for: bootstrap failure leaves the shell command unchanged
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the script sources, When the bootstrap-failure path is scanned, Then it does not replace the user's shell command
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts have no bootstrap-failure path that preserves the shell command
11. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::bootstrap_failure_keeps_the_shell`
12. Add that failure path. Re-run step 11 and verify PASS. Commit: `git commit -m "feat(pty): keep a normal shell when integration cannot start"`
13. Write failing test for: every keymap binds the clear-line sequence
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the bash and zsh scripts, When their keymap bindings are scanned, Then every keymap they declare binds the T1 `CLEAR_LINE_SEQUENCE` and no command text
    Exercise through: reading both script files and `CLEAR_LINE_SEQUENCE`
    Test doubles: none
    Expected RED: the scripts do not bind `\u001b[99;1~`
14. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::clear_line_sequence_is_bound`
15. Bind that sequence on every keymap the scripts declare. Re-run step 14 and verify PASS. Commit: `git commit -m "feat(pty): bind the clear-line sequence in bash and zsh"`
16. Write failing test for: a live bash handshake hides the protocol and reports suppression
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: integration
    Test intent: Given bash is on PATH, When an interactive login bash starts with the injection, Then the first session frame is InitShell for the issued nonce, the following PromptReady has `prompt_suppressed` and `line_empty`, and the bytes after the real T2 `SessionFilter` contain no 9477 payload. If bash is not on PATH, the test fails rather than passing
    Exercise through: a real bash process and `SessionFilter`
    Test doubles: none for bash. Do not mock the filter
    Expected RED: spawned bash emits no InitShell frame
17. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::bash_handshake_reports_suppression -- --nocapture`
18. Wire the plan into `spawn_pty` with a small call. Re-run steps 2, 5, 8, 11, 14, and 17. Verify PASS. If zsh is on PATH, run the same live assertion against zsh in this test module. If zsh is absent, that zsh process assertion is not run and the static zsh scan still is. Commit: `git commit -m "feat(pty): inject session integration into local login shells"`
19. Write failing test for: the visible prompt is zero-width
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the bash and zsh scripts, When they are scanned, Then they save the original prompt and install a zero-width visible prompt while integration is active
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts do not save and replace the visible prompt
20. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::visible_prompt_is_zero_width`
21. Save the original prompt and insert the zero-width assignment before the prompt hook's last statement, so a live shell still shows the control prompt. Re-run step 20 and verify PASS. Commit: `git commit -m "feat(pty): hide the shell prompt while integration is active"`
22. Write failing test for: the restore sequence brings the prompt back
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the scripts, When they are scanned, Then `RESTORE_PROMPT_SEQUENCE` restores the saved prompt and does not contain command text
    Exercise through: reading both script files and `RESTORE_PROMPT_SEQUENCE`
    Test doubles: none
    Expected RED: the scripts do not bind `\u001b[99;2~`
23. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::restore_sequence_brings_the_prompt_back`
24. Bind that sequence. Re-run step 23 and verify PASS. Commit: `git commit -m "feat(pty): restore the shell prompt from the saved value"`
25. Write failing test for: injected bash does not show the control prompt
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: integration
    Test intent: Given bash is on PATH, When one interactive bash starts without injection and another starts with injection, Then the filtered injected stream does not contain the prompt line captured from the control bash. If bash is not on PATH, the test fails rather than passing
    Exercise through: two real bash processes and `SessionFilter`
    Test doubles: none. Do not mock the filter
    Expected RED: the injected stream still contains the control prompt line
26. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::injected_bash_hides_the_control_prompt -- --nocapture`
27. Move the zero-width assignment to the end of the prompt hook. Re-run step 26 and verify PASS. Commit: `git commit -m "feat(pty): keep the injected prompt off the screen"`
28. Write failing test for: a user command emits lifecycle frames
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the bash and zsh scripts, When a user command path is scanned, Then it emits `CommandStarted` before the command and `CommandFinished` with the captured exit code after it, and the generator path emits neither
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts do not emit user `CommandStarted`
29. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::user_command_emits_lifecycle_frames`
30. Add those two emissions on the user path only. Also print completer stdout to the terminal, so the next test fails. Re-run step 29 and verify PASS. Commit: `git commit -m "feat(pty): emit command lifecycle from the user prompt hooks"`
31. Write failing test for: completer output stays off the terminal
    Test file: `src-tauri/src/pty/session_protocol/inject_tests.rs`
    Level: unit
    Test intent: Given the scripts, When the generator path is scanned, Then completer stdout and stderr are captured into the 9477 result frame and are not written to the terminal
    Exercise through: reading both script files
    Test doubles: none
    Expected RED: the scripts print completer output to the terminal
32. Run test — verify FAIL: `cargo test --manifest-path src-tauri/Cargo.toml --lib session_protocol::inject_tests::completer_output_stays_off_the_terminal`
33. Redirect that output into the result frame only. Re-run steps 26, 29, and 32. Verify PASS. Refactor while green. Commit: `git commit -m "feat(pty): keep completer output off the terminal"`

## REFERENCES LOADED

Spec — trusted prompt, v1 shells, prompt suppression, exactly-once echo steps 1 and 2
src-tauri/src/pty/env_refresh.rs — `shell_wants_login_arg`
src-tauri/src/pty/manager/spawn.rs — login arg around line 271
Warp was read only as behavior. No Warp source is copied.

## WHY THIS APPROACH

Complexity: standard
Justification: The shell is the only place that can hide its own prompt and skip Preexec for a generator. A static scan proves the contract markers. One live bash process proves the frame actually appears.

## SANDWICH CONTEXT

[CRITICAL: Inject only local interactive login bash and zsh. Do not copy Warp source.]
You are implementing shell integration for the native command editor.
Spec: docs/pocket/spec/2026-10-20-native-command-editor/native-command-editor.md
Design decision: Option A. The shell cooperates. A failed handshake stays raw.
Files in scope: inject.rs, inject_tests.rs, the two scripts, and the spawn call.
Available after: T1 and T2. The live bash test runs the real `SessionFilter`.
Architecture rule: ambient OSC 133 is not a mode signal.
[RESTATE: Inject only local interactive login bash and zsh. Do not copy Warp source.]

## DELIVERABLE

Given local interactive login bash or zsh, When a PTY is spawned, Then the integration script is installed with a new session id and nonce
Given fish, dash, ssh, PowerShell, cmd, or a direct program, When a PTY is spawned, Then no script is installed
Given the scripts, When a generator runs, Then it does not emit user `CommandStarted`
Given the scripts, When their keymaps are scanned, Then every declared keymap binds the T1 clear-line sequence
Given the scripts, When they are scanned, Then the original prompt is saved and the visible prompt is zero-width
Given the scripts, When they are scanned, Then the restore sequence puts the saved prompt back
Given bash on PATH, When it handshakes, Then PromptReady reports suppression and the filtered stream has no 9477 payload
Given bash on PATH, When an uninjected control prompt is compared with the filtered injected stream, Then the injected stream does not contain that prompt line
Given the scripts, When a user command runs, Then the user path emits `CommandStarted` and `CommandFinished` and the generator path emits neither
[must-not] Given a generator runs, When completer stdout or stderr is produced, Then those bytes are not written to the terminal
[must-not] Given bootstrap cannot prove suppression, When the shell starts, Then the spawn stays a normal shell

## QUALITY BAR

Must-have:
  - Scripts are original
  - The prompt hook re-applies suppression at the end
  - The clear-line binding matches the T1 sequence on every keymap the script can identify

Must-not-have:
  - fish `no-mark-prompt` injection in v1
  - A pretyping queue
  - A persisted force-raw setting

Open question risks:
  - zsh is absent in CI → the static zsh scan still runs. The live zsh assertion runs only when zsh is on PATH. Bash absence fails the live test rather than passing.
  - The wrapper is unproven → a provider that cannot restore state is not invoked, as required by T3.

Rollback note:
  - `injection_plan` returning raw for every shell returns startup to today's PTY.

Red flags:
  - A script copied from warpdotdev/warp → STOP
  - Injection into ssh or fish → STOP

## STOP CONDITIONS

Done when: the injector test, the script scan, and the live bash test pass
Uncertain when: bash exists but cannot be started as an interactive login shell in the test harness
Escalate when: spawn cannot take the plan without a rewrite of `spawn.rs`
