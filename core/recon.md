# Recon: before writing the bot

Spend the first hour finding out what the app already knows, how it accepts input, what else can appear on the screen, what will expire under you, and who else uses the device. The commands for each are in `platforms/<yours>.md`. This file is the list of questions.

## 0. May you?

Read the app's terms and any ban-risk notes first. Some apps act on automation, even slow automation. If it is not allowed, stop here.

## 1. What does the app already know?

The bot's hardest job is learning the app's rules and reading its state. Before writing a line of screen-reading code, look for a better source.

| Source | What it gives | Where it exists |
|---|---|---|
| The app's own files | Tables of prices, stats, positions, text | Android (APK assets); not on iPhone without a jailbreak |
| The accessibility or UI tree | Exact labels, values, element names, positions, visibility | iPhone (WebDriverAgent), Android (uiautomator), desktop, browser DOM |
| The app's own network traffic, through a proxy the owner set up | Full data for what the app loaded | Any platform where the owner trusts the proxy's certificate; apps that pin certificates refuse it |
| The screen | Everything else | Everywhere |

Rules for network data: observe only what the app itself fetches. Never call the app's API or content servers yourself.

**Verify before you trust.** A table, a tree value or a decoded response is only useful once it matches the screen. In the Android build, a price formula was accepted only after it reproduced 167 prices seen on screen.

## 2. How does it accept input?

Test these by hand before building on them. The measured answers for each platform are in its note.

- What is the shortest action that registers, and how often does it miss?
- Can actions be batched, or does each one need its own round trip?
- Which gesture scrolls without starting a drag? Which velocity overshoots?
- What does a long-press do besides the obvious? Which taps silently switch a mode that must be undone?
- How does text get in: typing, or pasting? Does what lands match what you sent, byte for byte?
- Do all input paths agree on coordinates (rotation, scaling, a mirrored window)?

## 3. What else can be on the screen?

Things that drew over the app or took focus during real runs:

- notification banners across the top, exactly where the app's top buttons are;
- an incoming-call banner with Accept and Decline buttons;
- a low-battery dialog;
- the lock screen;
- the owner picking up the phone.

Decide now what the bot does for each: stop, wait, or carry on. Never "tap it away".

## 4. What expires, and when?

List everything with a lifetime before you depend on it:

| Kind | Examples seen |
|---|---|
| Signing and trust | A free-account developer signature lasts exactly 7 days |
| Addresses | Wireless debugging ports change on every toggle; DHCP gave the phone and the Mac new IPs within days |
| Sessions | Automation session ids go stale after an app or runner restart |
| Clipboards | A clipboard shared across devices expired after about 2 minutes |
| Screen | Auto-lock; the screen turns off |

Keep each one in one place in the code, and probe it at the start of every run.

## 5. Who else uses this device?

Another agent or session may drive the same phone or the same automation port. Start scripts that "kill whatever is running" will stop them. Check the processes, tell the other session, and take turns.

## 6. What will the owner come to depend on?

A proxy, a trusted certificate, a VPN or a debug setting becomes part of the owner's daily setup once it works. Write the teardown list at setup time (proxy off, certificate profile removed, runner stopped). Never stop any of it without asking.

## 7. Write down the screens

Before any detector, list the app's screens. For each one, note:

- one cheap feature that identifies it;
- how to get to it and back;
- which buttons must never be pressed: resets, purchases, anything social or paid;
- what on it is fixed and what moves or grows.

That list is the map your `goto()` function and your never-tap list are built from.
