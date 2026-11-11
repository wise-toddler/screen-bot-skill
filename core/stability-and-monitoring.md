# Stability and monitoring

## Failure classes

Every row happened at least once in a real run (Android or iPhone).

| Class | What it looked like | Defence |
|---|---|---|
| A dependency hangs | OCR call never returns; bot silent | Run it in its own process with a timeout; kill and replace |
| The cure deadlocks | Terminating a stuck worker pool froze the parent for six minutes | Plain child process, plain kill. And a heartbeat, so any freeze is caught from outside. |
| A crash from a small edit | A variable name reused with another type | Compile before reload; supervisor restarts; three quick exits stop the loop |
| Two instances | Two bots tapping over each other | Start guard on the exact process pattern |
| The guard is too eager | It matched a shell that mentioned the script | Anchor the pattern |
| Stale coordinates | Taps on empty space | Re-detect after anything that moves the view; tap only what is visible now |
| A hardcoded zone | Real items hidden by an "ignore" rectangle | Measure the overlay |
| Merged detections | Two markers joined by a line read as none | Clean the mask |
| One bad reading poisons state | A misparsed number reset tracked levels | Sanity bounds; never overwrite state from one frame |
| A good reading rejected | A true value discarded as implausible | Compare with the app's data, keep the crop |
| Layout change | A panel grew; fixed positions missed | Find by content |
| The app moves the view itself | It zooms onto new items | Check the view every round |
| Focus stolen | Low-battery dialog; a call; the owner | Stop on loss of focus; supervisor waits; never relaunch |
| Overlays on the app's buttons | A call banner over a top button | Banner check; pause |
| Budget starvation | A cheap, vital action never bought | Reserve before ranking |
| Drifting bookkeeping | Tally ahead of reality | Reconcile with on-screen marks |
| Tests touching the live run | A test wrote into the live log | Tests in their own files and directory; never beside a live bot |
| Battery | Screen stays on; phone dies | Charger before a long run; log low battery |
| The screen locks | Everything stops; the automation server refuses to act until unlock | Owner turns off auto-lock for the run (Android: stay-awake while charging); a watcher that notices the lock and waits |
| Something expired | A 7-day signature lapsed mid-week; a session id went stale; an address changed | Probe each expiring thing at the start of a run; keep it in one place |
| A single-threaded driver hangs | One stuck request blocked every later command, while its status endpoint still answered | Timeout on every call; restart the driver |
| Another session drives the same device | Its start script killed ours | Check processes first; message the other session; take turns |
| The owner's infrastructure was stopped | The phone lost internet when its proxy was turned off | Never stop it unasked; keep a teardown list |
| The agent's own session restarted | Background watchers silently gone; a run dead for 3 days | After any restart, check processes and run files before assuming anything is alive |
| Wrong data shown for an item | A "most recent" fallback showed another item's data | Match by content the screen shows; else show nothing |

## Three layers of watching

**1. The supervisor** answers "is it alive?" It restarts on death and on a stale heartbeat. It knows nothing about whether the bot is doing the right thing.

**2. The event stream** answers "did something happen that someone should know about?" It is a filtered tail of the log:

```bash
tail -n 0 -F bot.log | grep -E --line-buffered "SUPERVISOR|EXIT|Traceback|Error|fallback|<the risky action>|<approaching a gate>"
```

Write the filter for failure first. If the bot crashed right now, would this print a line? If it hung? If it fell back to a slow path? Silence must not be able to mean both "fine" and "dead".

**3. Outcome checks** answer "is it doing the right thing?" This is the layer that was missing, and the owner filled it by watching the phone. Checks that would have caught what the owner caught:

| What the owner noticed | A check that would have noticed first |
|---|---|
| An important item stayed unbought | Same item marked available for N visits in a row |
| A kind of purchase had stopped | Zero purchases of a category in the last quarter hour while it is available |
| One category was taking over | Share of spend by category over a sliding window |
| The bot kept repeating something finished | Count of repeated actions on completed things |
| Things on one screen were not being bought | A buyable slot unchanged for N cycles |
| Progress had slowed | Income or progress rate falling between samples |

Each is a few lines over the record file. Write them as soon as the record exists.

## Waiting without blocking

A builder who waits on the bot in the foreground cannot be talked to, and may need rescuing by the owner.

| Need | How |
|---|---|
| "Tell me when X happens" | A background command that exits when X is true, with a time limit |
| "Tell me every time Y happens" | The event stream |
| "What is it doing now?" | A one-shot read of the log and state |
| "Did my change take effect?" | A background wait for the first log line only the new code prints |

Always give a wait a deadline, and always have it print what it saw. A wait that can only succeed tells you nothing when it does not.

## Proving a change took effect

After a reload, look for evidence the new code is running, not just that a process started:

- a log line with a value only the new code produces ("needs 20M" where it said "needs 12M");
- the first occurrence of the fixed behaviour (the starved item bought within one round);
- a count that should now be zero (no purchases of the paused category since the restart).

## Recording what cannot be repeated

For an action that happens once per run, start a recorder before it:

```bash
while [ $(date +%s) -lt $end ]; do
  adb -s $DEV exec-out screencap -p > fight/$(date +%H%M%S).png
  sleep 10
done
```

Twenty screenshots turned "we think we lost last time" into "it ended at this second, with this on screen". The earlier attempt had no recording, and a whole restart was planned on a guess.

## The log line

Every decision gets one line that can be understood alone:

```
05:25:06 stage 8 skipped: 407M health arriving over 388s needs 20M/s, have 6.97M/s
05:19:21 left the upgrade screen after <item>
05:14:22 income ~1,886,179/s over a cycle with a re-run
```

- the time;
- what was decided;
- the numbers it was decided on.

"Skipped" and "bought 3" are useless a day later.

## Reading back what you wrote

After typing into a form or a field through automation, read the value back before saying it is there. After changing a setting, read the setting. Report what the read-back showed, not what was sent.

## Honest numbers

Keep a few counters and look at them:

| Counter | Why |
|---|---|
| Restarts and their causes | Is stability improving? |
| Fallbacks to the slow path | Is the fast path as reliable as you think? |
| Misreads caught by sanity checks | How noisy is the reading? |
| View corrections (re-zooms) | How often does the app move things? |
| Cycle time | Did the last optimization do anything? |

In one build the median cycle went from 164 s to 91 s after long-press reading was replaced by registration. That number came from the log's own timestamps, not from an impression.
