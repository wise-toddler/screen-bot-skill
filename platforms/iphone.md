# iPhone

The iPhone side of `core/recon.md` and `core/skeleton.md`. The numbers come from several weeks of driving apps on one iPhone (iOS 26) from a Mac. Tags: **[M]** measured, **[I]** inferred, **[G]** guessed or untested.

## Capability matrix

| Path | Read | Act | Speed | Reliability | Use it for |
|---|---|---|---|---|---|
| iPhone Mirroring + CGEvents | Screenshots of the mirrored window; Vision OCR reads a page in 0.65 s [M] | Taps, long-press and drags through `kCGHIDEventTap` [M]; text by clipboard paste only (unicode key events are ignored) [M] | Fast per event | About 70% of operations succeeded in a home-screen reorganisation [M, rough count] | Quick jobs while the owner sits at the Mac |
| pymobiledevice3 alone | Screenshot about 1.5 s [M]; the accessibility list of third-party apps (labels, roles, selected state) [M] | None: `perform_press` is silently dropped, and HID touch fails with error 9021 on iOS 26 [M] | — | Read-only | Reading state with no runner installed |
| WebDriverAgent (Appium WDA) over USB or Wi-Fi | Screenshot 0.2–0.5 s; `/source` (the accessibility tree) 3.8 s on the home screen [M] | Taps about 1 s; gestures; byte-exact text including emoji [M] | 1–3 s per verified action [M] | No silent misses seen on named elements; failures were a wrong target or a blocked main thread [M, no formal count] | Real work |

There is no Android-style batching. Design for few actions, each one checked.

## Connect and set up WebDriverAgent

All [M]:

- Xcode needs the iOS simulator runtime (about 8 GB) installed before a real device shows up as a destination.
- **A free Apple ID signature lasts exactly 7 days** (the profile's `ExpirationDate`). When it runs out, the only symptom is "Failed to launch process with bundle identifier …xctrunner". Check the expiry before a run. After re-signing, the owner sometimes has to trust the developer again.
- **The phone must be unlocked** to build, launch and activate apps.
  - `/wda/locked` returned False on the lock screen on iOS 26. Don't trust it.
  - The real test: `/wda/apps/activate` fails with "device was not, or could not be, unlocked". Retry it in the background every 5 s, so the owner never has to tell you it's unlocked.
- **After a phone restart**, the developer disk image is not mounted. Run `mounter auto-mount` first.
- **Over Wi-Fi**, every `dvt` subcommand needs `--native`, and the runner logs its LAN URL.
  - Addresses change: the phone moved .8 → .4 within days, and the Mac .6 → .7 → .5.
  - Keep the base URL in one state file and re-probe `/status`.
- **Stopping the Mac-side `dvt xcuitest` does not stop the runner on the phone.** Port 8100 keeps answering anyone on the LAN, unauthenticated. Stop it with `dvt pkill --bundle <runner>`, and confirm that `/status` fails.
- **macOS Local Network privacy:** after a Homebrew Python upgrade, Python sockets to the LAN failed with errno 65 "No route to host" while Apple's `curl` still worked. Keep a `curl` fallback in the HTTP client.

## What the app already knows

App Store binaries are encrypted and the app sandbox is unreadable without a jailbreak [I, not attempted]. There is no APK equivalent. Use these instead:

1. **The accessibility tree** (`/source`): exact labels, values, element names, positions, and whether each element is visible [M].
2. **The app's own HTTPS traffic**, through a proxy the owner set up and whose CA certificate the owner trusts on the phone. It gives full JSON and images for what the app loaded [M on one app].
   - Apps that pin certificates won't work [G].
   - The app prefetches and caches, so for cached items the proxy saw nothing. Fall back to the tree's text [M].
   - Observe only what the app itself fetches. Never call its API or content servers yourself.

## Seeing

- **On normal UIKit/SwiftUI apps, the tree beats colour and OCR.** Find rows by element name or label, never by coordinates. An info banner that reappears shifts the layout [M].
- **Off-screen elements stay in the tree** with `visible=false` and a y outside the screen. That tells you which way to scroll [M].
- **Games and canvas apps** probably expose no tree, so use colour and shape as in `core/` [G].
- **WDA coordinates are points** (up to about 440 × 956 on the test phone) [M]. Screenshots are pixels at 3× [I].
- **iPhone Mirroring:** work from app names, never cached coordinates, because the grid reflows after every operation [M]. Map coordinates from the window's content rect to phone points, and re-read the window rect each time [I].

## Tapping and gestures

All [M]:

- **Tapping a StaticText by its label did nothing.** Tap the cell's rect.
- **Duplicate labels:** the keyboard's return key is also named "Done". Pick the navigation-bar one by its rect (y < 120).
- **The plain swipe (`dragfromtoforduration`) holds long enough to start a drag-reorder.** Use `pressAndDragWithVelocity` instead:

| Gesture | Settings |
|---|---|
| Slow scroll | press 0.01 s, hold 0.3 s, velocity 350–500, in the right gutter (x ≈ 432–434) |
| Flick | hold 0.05 s, velocity 900–1200 |
| Go home | 220,953 → 220,450, press 0.01 s, velocity 2500. A slow swipe opens the App Switcher, and `/wda/pressButton home` was silently dropped |
| Pan a photo | press 0.05 s, velocity 150–200 |
| Reorder | long-press 1.2 s |
| Photo picker | ignores the plain swipe; flick at velocity 1500 |

## Typing

Paste, don't type. Fast typing drops a whole sentence at once, and doesn't look like a person.

Recipe [M]:
1. Tap the text twice, about 1.2 s apart, to open the edit menu.
2. Select All, then Paste.
3. Read the field back before pressing Done.

Clipboard traps:
- **Universal Clipboard:** `setPasteboard` alone once pasted the Mac's clipboard instead [M].
  - Fix: `pbcopy` the same text on the Mac and also `setPasteboard`, wait about 4 s, then restore the Mac clipboard.
  - Universal Clipboard needs Bluetooth on at copy time, and expires after about 2 minutes [M].
- **`setPasteboard` can return OK and leave the phone's clipboard empty** (the read-back was empty) when the runner isn't the foreground app [M for the empty read-back, I for the cause]. Always read back.

## Failures and their defences

All [M]:

| Failure | Defence |
|---|---|
| Signature expired mid-week | Check the profile's expiry before a run; keep a re-sign script |
| "Not authorized for performing UI testing actions" (XCTDaemon code 41) | Restart the runner |
| Stale session: "Application 'local.pid.0' is not running" | Drop the session id and create a new one |
| One hung call (activate while locked, an app stuck on its splash) blocks every session command, while `/status` still answers | Timeout on every HTTP call; restart the runner or `dvt pkill` the stuck app |
| The phone re-locked during a ~1 minute runner restart | The owner sets Auto-Lock to Never for the run; the unlock watcher keeps polling |
| The item on screen changed between the suggestion and the send | Re-read the target's identity (name and one content field) before every outward action; if it differs, send nothing |
| A button wasn't found because the view had scrolled past it | Scroll toward the element using its off-screen y |
| A sheet was slow to appear | Wait up to 8 s for its text field before pasting |
| A "most recent images" fallback showed another item's data | Removed the fallback: match by what the screen shows, else show nothing |
| Guessed labels for coded values printed wrong facts | Label only codes verified against known data; otherwise print "code N" |
| The proxy was stopped and the owner's phone lost internet at home | Never stop infrastructure the owner depends on without asking |
| The Mac's IP changed and broke the manual proxy. A PAC file using the Mac's `.local` name kept the internet up but sent no traffic to the proxy in 6 minutes [I: the name didn't resolve] | PAC by IP is not yet confirmed |
| Background watchers died when the agent's own session restarted; one run was dead for 3 days | After any restart, check processes and run files first |
| Another session drove the same WDA on :8100, and start scripts killed whoever was running | Check processes and message the other session before starting; say when done |

## Safety on the owner's accounts

These rules came from the owner and are enforced in code where possible:

- **Human pace, for the account's safety, not evasion.**
  - Paste, never instant typing.
  - Jittered gaps: 1.5–5 s between actions, 5–15 s "reading" a screen, 20–40 s between groups [M, values in use].
  - Short sessions; stop the runner afterwards.
- **No blind or random taps** inside social or shopping apps: they can land on like or buy. Where an idle action is wanted, use a harmless one.
- **A never-tap list is checked against the target's label before every tap.** Paid and scarce actions, undo and skip are on it. The script exits rather than tap.
- **Outward actions** (like, comment, send):
  - only the exact text the owner picked, one at a time;
  - re-verify the target first, and read back before submitting.
  - Paid or scarce actions happen only when the owner names that action for that item.
  - Upsell dialogs get their decline button.
- **Never auto-act and never bulk.** Skip only on the owner's word.
- **Other people's data:**
  - kept in the proxy's memory only;
  - one on-screen item is written to disk only on request, and wiped after;
  - no screenshots on those screens;
  - nothing about third parties in notes or memory.
- **System and security settings are the owner's:** trusting the certificate, proxy settings, the Local Network toggle, Auto-Lock. Open the pane and let the owner tap.
- **Tell the owner before taking the phone**, and send a progress line every few minutes.

## Where `core/` differs on iPhone

| Core rule | On iPhone |
|---|---|
| `in_app()` via `dumpsys` | `/wda/activeAppInfo` bundle id [M] |
| Heartbeat on every screenshot | Heartbeat on every tree read, because screenshots may be banned for privacy |
| Fast batches, reconcile after each batch | No batching; verify each action |
| "Never relaunch the app" | Open it when the owner starts a session; if focus is lost mid-run, stop and don't take it back |
| Colour and shape first | Tree first; colour, shape and OCR only when the tree is empty |
| Battery: the screen stays on and drains the phone | The opposite risk: the screen locks and everything stops. Auto-Lock Never (set by the owner) plus a lock watcher |
| Do Not Disturb | A Focus mode, set by the owner |
| Battery level via `dumpsys battery` | pymobiledevice3 diagnostics or WDA battery info [G] |
| Pinch via a monkey script | WDA has a pinch endpoint [G, untested] |
