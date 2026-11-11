---
name: screen-bot
description: >-
  How to build and run a bot that operates an app by looking at the screen and tapping (Android over
  ADB, iPhone over WebDriverAgent or iPhone Mirroring, and the same rules for desktop or browser
  automation), and how to keep it running unattended while its owner keeps changing what it should do.
  Use when building, debugging or supervising any screen-driven automation of an app on a device the
  person owns: game bots, app automation, UI test drivers, "tap this for me" helpers.
---

# Screen bot

How to build a bot that operates an app by looking at the screen and tapping, and how to keep it running unattended while its owner keeps changing what it should do.

This skill is distilled from real builds. An Android idle-game bot ran over wireless ADB for two days. An iPhone helper ran over WebDriverAgent and iPhone Mirroring for several weeks. The core rules are platform-neutral. Commands and measured numbers for each platform live in `platforms/`.

## The idea in ten lines

1. Find out what the app already knows about itself before teaching the bot to read it off the screen.
2. Build the boring frame first: one loop, one log, one record file, a supervisor, a heartbeat, a way to reload.
3. Read the app's structure where it exposes one (accessibility tree, DOM). Otherwise use colour and shape. Use OCR only for text.
4. Never assume a tap worked. Never assume a coordinate is still valid.
5. Rank what to do by value per cost, or, where a person decides, suggest and let them pick.
6. Put every risky or irreversible in-app action behind a gate, and say whether the gate is measured or guessed.
7. Record everything, with numbers, so "why did it do that?" has an answer.
8. Monitor outcomes, not just errors. A bot can run cleanly and be wrong for an hour.
9. Treat every instruction from the owner as a named switch with their words next to it.
10. Anything that leaves the device in the owner's name waits for their explicit word.

## Build order

| Step | What | Read |
|---|---|---|
| 0 | Is automation allowed for this app? Check its terms and ban risk | Scope, below |
| 1 | Recon: data, inputs, screen, what expires, who else uses the device | `core/recon.md`, then `platforms/<yours>.md` |
| 2 | Skeleton: loop, log, records, supervisor, heartbeat, flags | `core/skeleton.md` |
| 3 | Seeing and tapping | `core/seeing-and-tapping.md` |
| 4 | Deciding: ranking for app economies, human-in-the-loop for everything else | `core/deciding.md` |
| 5 | Stability and monitoring | `core/stability-and-monitoring.md` |
| 6 | Working with the owner and changing requirements | `core/working-with-the-owner.md` |
| 7 | Checklists | `core/checklists.md` |

In the Android build this came from, the order was nearly the reverse: decisions first, skeleton last. Most of the lost hours trace back to that.

## Platforms

| Platform | Read the screen | Act | Note |
|---|---|---|---|
| Android | `screencap`, the APK's own data files | `adb shell input`, about 90 ms per tap, batchable | `platforms/android.md` |
| iPhone | the WebDriverAgent accessibility tree, screenshots, the app's own traffic through a proxy the owner set up | WebDriverAgent gestures, 1–3 s per verified action, no batching | `platforms/iphone.md` |
| Browser, desktop | the DOM or the accessibility tree | the automation tool's own actions | The core rules apply as written |

## The costliest mistakes

| Mistake | What it cost | Rule |
|---|---|---|
| Reading everything off the screen when the app's data was available | Hours of slow OCR and a whole class of bugs | Recon before code |
| No supervisor or heartbeat until the bot had hung | A six-minute freeze nobody noticed, and waits that blocked the builder | Skeleton before features |
| Counting every tap as a success | Bookkeeping drifted, and estimates built on it were wrong | Verify effects; trust the screen |
| Monitoring only for errors | The owner caught seven wrong behaviours before the builder did | Monitor outcomes |
| Not recording an unrepeatable event | A whole restart was planned around an outcome nobody saw | Record what cannot be repeated |
| A safety threshold scaled from an unverified number | Needless waiting, then an override | Calibrate, or say it is a guess |
| Acting on inferred consent for an outward action | A submission that could not be taken back | Explicit go, every time |
| Stopping infrastructure the owner had come to depend on (a proxy) | The owner's phone lost internet at home | List it at setup; never stop it unasked |
| Assuming background watchers survived a restart of the agent's own session | A run was dead for three days unnoticed | After any restart, check processes and run files first |

## Scope

This is for automating an app on a device you own, for yourself.

- **Check first.** Online and competitive apps often forbid automation and act on it. Read their terms and the ban-risk notes before you start.
- **Human pace protects the owner's account; it is not evasion.** Rate-limiting, low volume and verifying each step serve that purpose. Nothing here is about hiding a bot or defeating detection.
- **Other people's content stays private.** Keep it in memory only, and keep nothing about third parties in notes or logs.

The code sketches are adapted from bots that ran. They are short on purpose: patterns to copy, not a framework.
