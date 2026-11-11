# Checklists

## Day zero: recon

- [ ] Automation is allowed for this app (terms, ban risk)
- [ ] Device reachable; address kept in one place
- [ ] The app's own data found and verified against the screen (Android: APK assets; iPhone: tree, owner-set proxy)
- [ ] Shortest reliable action measured, with its miss rate
- [ ] Batching tested, where the platform has it
- [ ] Scroll, drag and pinch gestures found, with their timings or velocities
- [ ] Text input tested and read back byte for byte
- [ ] Side effects of long-press and of taps on the main area noted
- [ ] App frame measured inside the physical screen (letterboxing, rotation, mirrored window)
- [ ] List of screens, with one identifying feature each
- [ ] Never-tap list, by label where there is a tree
- [ ] What can draw over the app or steal focus, and what the bot does for each
- [ ] What expires and when: signing, ports, addresses, sessions, clipboard
- [ ] Who else drives this device
- [ ] Teardown list for anything set up on the owner's device

## Skeleton is done when

- [ ] One loop with phases, each capped by time
- [ ] Exits when the app is not focused; never relaunches it
- [ ] Refuses to start beside another instance (a lock file, not a process-name pattern)
- [ ] Stop flag honoured only at a cycle boundary
- [ ] State saved after every phase
- [ ] Log line per decision, with numbers
- [ ] Record line per event
- [ ] Heartbeat touched on every look at the screen or tree, and during waits
- [ ] OCR in a killable helper with a timeout
- [ ] Supervisor: restart on death, on stale heartbeat; give up after three quick exits; wait when the app is not in front
- [ ] Reload works: compile, flag, new code running

## Before an unattended run

- [ ] Phone on a charger
- [ ] Screen stays awake (Android: stay-awake while charging; iPhone: auto-lock off, set by the owner), and you know how to restore it
- [ ] Do Not Disturb or a Focus mode, if the owner agrees
- [ ] App open and in front
- [ ] Event stream armed, and its filter prints on crash, hang, fallback and the risky actions
- [ ] Outcome checks in place for the things the owner cares about
- [ ] A recorder ready for anything that happens once
- [ ] The owner knows how to stop it

## Every change

- [ ] Smallest change that does it
- [ ] Compiles
- [ ] Reloaded at a boundary, not by killing
- [ ] Evidence in the log that the new code is running
- [ ] Evidence that the behaviour changed
- [ ] The owner told what changed, in one or two lines
- [ ] If it was a switch: constant has a comment with who, when, why

## When the owner reports something odd

- [ ] Confirm it in the log or record before answering
- [ ] Find the cause, not just the instance
- [ ] Was it a question or an instruction?
- [ ] Say what was measured and what is inferred
- [ ] Add an outcome check so the next one is caught without them

## Before a risky in-app action

- [ ] It is identified robustly, not by its position among other things
- [ ] The gate is a named constant with its origin
- [ ] It is clear whether the gate is derived, calibrated or guessed
- [ ] The measure behind the gate is understood row by row
- [ ] Several consecutive readings, not one
- [ ] Recorder running
- [ ] The owner knows it may happen, or has ordered it

## Before anything leaves the device

- [ ] The complete final content has been shown
- [ ] The owner has said to send it, in words that mean that
- [ ] Nothing was inferred from an answer to a different question
- [ ] After sending: read back and report what the page shows

## When the tally and the screen disagree

- [ ] The screen is right
- [ ] Find a mark that states the truth and count it
- [ ] Reconcile the tally
- [ ] Redo any estimate that used the tally

## Wrapping up

- [ ] Bot and supervisor stopped; no stray processes
- [ ] Tabs, flags and temporary files you created are cleaned up
- [ ] Screenshots, crops and dumps deleted; nothing about other people left on disk
- [ ] Stay-awake, auto-lock and other settings restored, or left as they are with the owner's OK
- [ ] The owner told what state everything is in
- [ ] What was learned written down where the next build will find it
- [ ] Numbers in the write-up come from the log, not from memory

## Smells

| If you notice | It usually means |
|---|---|
| You are waiting in the foreground | A monitor or a supervisor rule is missing |
| The owner is telling you what the bot is doing | Outcome checks are missing |
| A detector has a magic rectangle | An overlay should be measured |
| A list is sorted and something at the end never happens | Budget should be reserved |
| An estimate keeps slipping | It is built on the bot's own tally |
| A threshold has a round number and no comment | It is a guess |
| The log says "skipped" with no numbers | You will not be able to explain it tomorrow |
| You are about to write "probably" about something that happened | It was not recorded |
| You wrote "I will do X once you answer" | You are about to treat an answer as permission |
| A process check by name pattern | It will miss a full-path start; use the lock file's pid |
| You are about to stop a proxy, runner or VPN you set up | The owner may now depend on it: ask first |
| Your own session just restarted | Your background watchers are gone: check before assuming the run is alive |
