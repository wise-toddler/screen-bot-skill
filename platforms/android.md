# Android

The Android side of `core/recon.md` and `core/skeleton.md`: how to connect, where the app's own data is, how input behaves, and the helpers the skeleton calls. The numbers are from one Android phone over wireless ADB.

## 1. Connect

Wireless ADB, no cable:

```bash
dns-sd -B _adb-tls-connect._tcp local.            # service name (macOS)
dns-sd -L <name> _adb-tls-connect._tcp local.     # host and port
adb connect <ip>:<port>
```

- The port changes every time wireless debugging is toggled. Keep the address in one place in your code.
- More than one transport can show up after connecting. Always pass `-s <ip:port>`.
- Some apps refuse to run while wireless debugging is on. Do not work around that.

## 2. What does the app already know?

The bot's hardest job is learning the app's rules. Often the rules are sitting in the install.

```bash
adb shell pm path <package>          # where the APKs are
adb pull <path>/base.apk             # and any split_config.*.apk
unzip base.apk -d apk/
```

Then look, in this order:

| Where | What you may find | How |
|---|---|---|
| `assets/` | Tables as CSV, JSON, INI, XML, SQLite: prices, stats, level caps, text in every language | `ls`, open them |
| The main data file | URLs, keys, format names | `strings -n 8 <file> \| grep -iE "https?://"` |
| `lib/<abi>/*.so` | Which engine built it, and whether function names survived | `nm -D`, or pyelftools |
| Compiled code with symbols | Constants built in code, not in tables | Disassemble the named function (capstone) and read the literals |

Engine hints from library names: `libyoyo.so` is GameMaker, `libunity.so` and `libil2cpp.so` are Unity, `libgodot_android.so` is Godot, `libUE4.so` or `libUnreal.so` is Unreal.

What you usually cannot get without root: the save file. `run-as` only works on debuggable apps. Plan to read live state off the screen and static rules out of the files.

**Verify before you trust.** A table is only useful once it matches the screen. Take every on-screen value you have already read and fit it against the table. In the build this came from, a price formula was accepted only after it reproduced 167 prices seen on screen, and a decoded table only after it matched every value the bot had measured.

What good data buys you:

- prices and effects without opening a single info panel;
- the position of things, if the table has coordinates;
- which choices exclude each other;
- how hard the next stage is, before attempting it.

## 3. How does it accept input?

Test these by hand before building on them.

| Question | Why it matters | What was found on one phone |
|---|---|---|
| Shortest hold that registers as a tap | Sets your speed limit | About 90 ms for a single tap; 40 ms in a fast batch mostly works, not always |
| Do some buttons need longer? | A button that ignores short taps looks like a bug in your code | One button needed about 200 ms |
| Can inputs be chained in one `adb shell` call? | Four times faster per tap than one call each | Yes: `input swipe x y x y 40 ; input swipe ...` |
| What does a long-press do, besides the obvious? | Side effects move things | It opened a panel, and under some conditions scrolled the view |
| Pinch and drag | You will need to normalize zoom | `input` cannot pinch. A monkey script with `PinchZoom(...)` can. Raw `sendevent` was denied. |
| Do all input paths agree on coordinates? | Rotation can break one of them | Monkey `Tap` landed in the wrong place on a rotated display; `input` was correct |
| Which taps change a mode? | You must undo it | Any tap on the play field switched an auto feature to manual |

## 4. What is on the screen besides the app?

| Check | Command or method |
|---|---|
| Logical size and rotation | `adb shell wm size`, `dumpsys window displays` |
| Is the app letterboxed? | Screenshot: where does the app's picture start and end? Your coordinates live inside that frame. |
| Focused app | `adb shell "dumpsys window \| grep mCurrentFocus"` |
| Battery and charging | `adb shell dumpsys battery` |

Things that will draw over the app or take focus, all of which happened in one night:

- notification banners across the top, exactly where the app's top buttons are;
- an incoming call banner with Accept and Decline buttons;
- the low-battery dialog;
- the owner picking up the phone.

Decide now what the bot does for each: stop, wait, or carry on. Never "tap it away".

## 5. Helpers the skeleton calls

```python
def adb(*args, binary=False):
    out = subprocess.run(["adb", "-s", DEVICE, *args], capture_output=True, timeout=40).stdout
    return out if binary else out.decode(errors="replace")

def in_app():
    return PKG in adb("shell", "dumpsys window | grep mCurrentFocus")

def shot():
    Path("heartbeat").touch()           # every look at the screen proves the bot is alive
    return Image.open(BytesIO(adb("exec-out", "screencap", "-p", binary=True))).convert("RGB")

def press(x, y, ms=90):
    adb("shell", "input", "swipe", str(x), str(y), str(x), str(y), str(ms))

def taps(points, hold=40):
    """Many taps fast: 20 per adb call. False if the app loses focus on the way."""
    for i in range(0, len(points), 20):
        adb("shell", " ; ".join(f"input swipe {x} {y} {x} {y} {hold}" for x, y in points[i:i + 20]))
        if i + 20 < len(points) and not in_app():
            return False
    return True
```

The `timeout=40` on every `adb` call matters: a dropped Wi-Fi connection otherwise blocks forever.

## 6. Before a long run

- **Keep the screen on** while plugged in: `adb shell svc power stayon usb` (or `true`). Note the old setting first, and restore it when the run ends.
- **Charger.** A screen that stays on drains the battery. Log `adb shell dumpsys battery` now and then, and stop cleanly below a floor you choose.
- **Do Not Disturb**, if the owner agrees, keeps banners off the app's top buttons.
