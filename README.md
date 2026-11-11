# screen-bot

An agent skill for building and running bots that operate an app by looking at the screen and tapping. It covers Android over ADB and iPhone over WebDriverAgent or iPhone Mirroring, and its rules also apply to desktop and browser automation. It also covers keeping such a bot running unattended while its owner keeps changing what it should do.

It is distilled from real builds: an Android idle-game bot that ran over wireless ADB for two days, and an iPhone helper that ran over WebDriverAgent for several weeks. Every rule comes from a failure that happened, with the numbers that were measured.

## Install

```bash
npx skills add wise-toddler/screen-bot-skill -g
```

Or copy this folder to `~/.claude/skills/screen-bot/`.

## What's inside

| File | Covers |
|---|---|
| `SKILL.md` | The idea in ten lines, build order, the costliest mistakes, scope |
| `core/recon.md` | What to find out before writing the bot: data, input, overlays, what expires, who else uses the device |
| `core/skeleton.md` | Supervisor, loop, control files, lock, heartbeat, a killable OCR helper, live reload |
| `core/seeing-and-tapping.md` | Rules for perception and action: tree first, verify every tap, never-tap lists, evidence retention |
| `core/deciding.md` | Ranking and budgets for app economies; human-in-the-loop for apps where a person decides |
| `core/stability-and-monitoring.md` | Failure classes and their defences; three layers of watching |
| `core/working-with-the-owner.md` | Changing requirements; question vs instruction; outward actions; the owner's settings and infrastructure |
| `core/checklists.md` | Checklists for each stage, and smells |
| `platforms/android.md` | ADB connect, APK data, input measurements, helpers, run prep |
| `platforms/iphone.md` | Capability matrix (Mirroring, pymobiledevice3, WebDriverAgent), setup traps, gestures, typing, failures, safety |

## Scope

Use this to automate an app on a device you own, for yourself. Check the app's terms first: online and competitive apps often forbid automation. Human pace here protects the owner's account; nothing in this skill is about hiding a bot or defeating detection.

## License

MIT
