# Skeleton: build this before any cleverness

The frame below is about 150 lines. It was added last in the build this came from, after the bot had already frozen once and crashed once. Add it first.

## Three processes

| Process | Job |
|---|---|
| Supervisor | Keeps exactly one bot alive. Restarts it after a crash, a hang, or a requested stop. |
| Bot | One loop. Screenshot, decide, tap, record. |
| OCR helper | Runs the OCR engine in its own process, so a hung OCR call can be killed without touching the bot. |

## Files as the control surface

| File | Written by | Meaning |
|---|---|---|
| `bot.log` | Bot and supervisor | One human-readable line per decision, with numbers |
| `data/<run>.jsonl` | Bot | One JSON line per event: the record |
| `state.json` | Bot | Memory across cycles and restarts |
| `heartbeat` | Bot, on every look at the screen or the tree, and during waits | "I am alive" |
| `bot.lock` | Bot, at start | One instance only; holds the pid the supervisor checks |
| `stop.flag` | You | Stop at the next safe point. Under the supervisor: reload the code. |
| `supervisor.stop` | You | End the supervisor |
| `<action>.flag` | You | A one-off order. The bot deletes it when it acts. |

Flags beat signals and restarts: they are visible with `ls`, they survive a crash, and anyone can set one.

## The loop

```python
def take_lock():
    """One bot at a time: an OS lock on bot.lock, holding our pid for the supervisor."""
    f = open("bot.lock", "a+")
    try:
        fcntl.flock(f, fcntl.LOCK_EX | fcntl.LOCK_NB)
    except BlockingIOError:
        return None
    f.seek(0); f.truncate(); f.write(str(os.getpid())); f.flush()
    return f                            # keep it open: the lock lives as long as the process

def main():
    lock = take_lock()
    if lock is None:                    # two bots tap over each other
        return log("EXIT: another bot is running")
    state = load_state()
    while time.time() < end:
        if Path("stop.flag").exists():  # only between cycles: never mid-purchase, never mid-fight
            Path("stop.flag").unlink()
            break
        if not in_app():                # the owner took the phone, or a dialog stole focus
            break
        do_screen_a(state)              # each phase: look, decide, act, record
        save_state(state)
        do_screen_b(state)
        save_state(state)
        idle(wait_for(state), state)    # waiting is part of playing; it also touches the heartbeat
    log("EXIT:", reason)
```

Rules that came from failures:

- **Stop only at a boundary.** A stop in the middle of a multi-tap action leaves the app in a state the next start does not expect.
- **Exit when the app is not in front. Never relaunch it.** If the owner took the phone, the bot has no business taking it back.
- **Save state after every phase**, not at exit. The bot will be killed.
- **Cap each phase by time.** One screen with endless work must not starve the other.

## Talking to the device

Each `platforms/<yours>.md` gives the same few helpers, so the loop reads the same everywhere:

| Helper | Does | Android | iPhone |
|---|---|---|---|
| `in_app()` | Is the app in front? | `dumpsys window` focus | WebDriverAgent active-app bundle id |
| `shot()` or `read()` | Look at the screen | `screencap` | the accessibility tree, or a screenshot |
| `press(x, y)` | One tap | `input swipe` with a hold | a WebDriverAgent tap on an element's rect |
| `taps(points)` | Many taps fast | 20 per `adb` call | none: verify each action |

Touch the heartbeat on every look at the screen or the tree. Where screenshots are not allowed (screens showing other people), the tree read is the heartbeat.

Put a timeout on every call to the device. A dropped Wi-Fi connection, or a single-threaded automation server stuck on one request, otherwise blocks forever.

## Log and record are different things

```python
def log(*a):
    line = time.strftime("%H:%M:%S ") + " ".join(str(x) for x in a)
    print(line, flush=True); LOG.write(line + "\n"); LOG.flush()

def record(state, kind, **data):
    run = state.setdefault("run_id", time.strftime("run_%Y%m%d_%H%M"))
    start = state.setdefault("run_start", time.time())
    with open(f"data/{run}.jsonl", "a") as f:
        f.write(json.dumps({"t": round(time.time() - start), "clock": time.strftime("%H:%M:%S"), "kind": kind, **data}) + "\n")
```

- The **log** is for reading: what was decided and why, in one line, with the numbers that drove it. "Skipped: needs 20M/s, have 9.3M/s" answers a question a week later. "Skipped" does not.
- The **record** is for computing: every purchase, reading and outcome as data. It is what lets you answer "what did the bot spend the last ten minutes on?" in one script.

## OCR in a process you can kill

An OCR engine can hang inside native code. A thread cannot be killed, and terminating a worker pool whose worker is stuck can deadlock the parent. A plain child process and a plain kill cannot.

```python
# ocr_worker.py: reads image paths on stdin, writes one JSON line per image
for line in sys.stdin:
    try:
        print(json.dumps(run_ocr(line.strip())), flush=True)
    except Exception as e:              # one bad image must not stop the helper
        print(json.dumps({"error": str(e)}), flush=True)
```

```python
def vision(im):
    global OCR_PROC
    im.save("data/ocr_tmp.bmp")
    for _ in range(3):
        if OCR_PROC is None or OCR_PROC.poll() is not None:
            OCR_PROC = subprocess.Popen([sys.executable, "ocr_worker.py"], stdin=PIPE, stdout=PIPE, text=True)
        try:
            OCR_PROC.stdin.write("data/ocr_tmp.bmp\n"); OCR_PROC.stdin.flush()
            if select.select([OCR_PROC.stdout], [], [], 25)[0]:     # 25 s of silence means it is stuck
                out = json.loads(OCR_PROC.stdout.readline() or "null")
                if isinstance(out, list):
                    return out
        except (OSError, ValueError):
            pass
        log("OCR helper gave no answer, replacing it")
        OCR_PROC.kill(); OCR_PROC = None
    return []
```

## The supervisor

```sh
#!/bin/zsh
say() { echo "$(date +%H:%M:%S) SUPERVISOR: $1" >> bot.log; }
alive() { [ -f bot.lock ] && kill -0 "$(cat bot.lock 2>/dev/null)" 2>/dev/null; }
quick=0; started=0
while true; do
  [ -f supervisor.stop ] && { rm -f supervisor.stop; say "stopped on request"; exit 0; }
  if ! alive; then
    if (( started )) && (( $(date +%s) - started < 45 )); then quick=$((quick + 1)); else quick=0; fi
    (( quick >= 3 )) && { say "the bot stopped within 45s three times: needs a look"; exit 1; }
    app_in_front || { sleep 15; continue; }       # the platform's focus check (platforms/<yours>.md)
    pkill -9 -f 'ocr_worker.py'; touch heartbeat; say "starting the bot"
    (python3 bot.py > bot_run.out 2>&1 &); started=$(date +%s); sleep 25; continue
  fi
  age=$(( $(date +%s) - $(stat -f %m heartbeat) ))          # macOS; Linux: stat -c %Y
  (( age > 150 )) && { say "no sign of life for ${age}s: restarting"; kill -9 "$(cat bot.lock)"; pkill -9 -f 'ocr_worker.py'; started=0; }
  sleep 10
done
```

What each part is for:

| Part | Failure it covers |
|---|---|
| Restart when no process | Crashes, finished time budgets, requested reloads |
| Heartbeat age | Hangs: the process exists but does nothing |
| Three quick exits | A bug at start-up; restarting forever would hide it |
| Focus check before starting | The owner has the phone; wait, do not fight for it |
| Kill stray helpers | A helper left over from the killed bot |
| PID from the lock file, not a name pattern | A pattern misses a bot started by full path or `uv run`, and matches shells and editors that mention the file |

## Changing code while it runs

```bash
python3 -m py_compile bot.py && touch stop.flag
```

The bot finishes its cycle and exits; the supervisor starts the new code. No process is touched by hand, and a syntax error never reaches the phone. In the build this came from, five changes went live this way in one hour without incident.

## What goes in `state.json`

| Kind | Examples | On a fresh run |
|---|---|---|
| The owner's choices | Which of two exclusive options to take; switches | Keep |
| What the bot has learned about this run | Levels bought, stages cleared, measured rates | Clear |
| Where the view was left | Scroll position, zoom | Clear |

Keep the two apart from the start. Mixing them means a restart of the app's progress either forgets the owner's choices or carries stale progress forward.
