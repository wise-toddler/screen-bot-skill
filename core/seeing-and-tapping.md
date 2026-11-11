# Seeing and tapping

The bot has one sense (a screenshot) and one hand (a tap). Most bugs are one of two beliefs: "I know what is on the screen" and "my tap did what I meant".

## Seeing

**1. Identify the screen first.**
Every phase starts with "which screen is this?", answered by one cheap fixed feature. If the answer is not the expected one, do nothing and navigate. A detector run on the wrong screen returns confident nonsense.

**2. The app's structure first; colour and shape next; OCR only for text.**
Where the platform exposes an accessibility or UI tree (iPhone through WebDriverAgent, Android uiautomator, desktop accessibility, the browser DOM), find elements by name and label, never by coordinates. Off-screen elements often stay in the tree with a position outside the screen, which tells you which way to scroll. Games and canvas apps usually expose no tree; there, blob detection on a colour mask is fast and almost never wrong. OCR is slow and was the source of most misreads. Use it for numbers and labels, nothing else.

| Want | Use |
|---|---|
| "Is this item available / locked / done?" | The marker's colour and fill ratio |
| "Where are the items?" | Connected components of the right size |
| "Is a panel open?" | Its background colour over a large area |
| "How much do I have?" | OCR on a small crop |

**3. Normalize the view, then prove it.**
If the app can zoom or scroll, bring it to one known view before detecting anything. Then measure a landmark to confirm: in one build, a recurring mark was exactly 11 px wide at minimum zoom and 12 px one notch in. Apps re-zoom on their own (for example onto a newly unlocked item), so check every time, not once.

**4. Do not hardcode a layout.**
A panel that had 7 rows grew to 9 and every slot moved. Find things by what they show (a label, a count, a colour), not by where they were yesterday.

**5. Measure overlays; never assume their zone.**
A fixed "ignore the area where the info panel appears" zone hid sixteen real items for twenty minutes, because early on the first items sit exactly there. Detect the panel itself and mask only what it covers.

**6. Clean the mask before finding blobs.**
A thin line connecting two markers merges them into one long shape, and the size filter drops both. A 3x3 morphological opening removes the line and keeps the markers.

**7. If you know where things should be, register instead of reading.**
When the app's own data gives positions, you can name every item on screen without touching it:

```python
def register(points, known):
    """Offset that lays known positions over detected points, and the share of points it explains."""
    d = (points[:, None, :] - known[None, :, :]).reshape(-1, 2)     # every pairing votes for an offset
    keys, counts = np.unique(np.floor(d / 4).astype(int), axis=0, return_counts=True)
    best = keys[counts.argmax()] * 4 + 2
    off = np.median(d[(np.abs(d - best) <= 6).all(axis=1)], axis=0)
    hit = sum(1 for p in points if (np.abs(known + off - p).max(axis=1) <= 4).any())
    return off, hit / len(points)
```

Always return the fit quality, and fall back to a slower method when it is poor. One zoom notch off gave a fit of 7%, the fallback ran for one round, and nothing wrong was tapped.

**8. Distrust every number.**

- Parse strictly. A regex that made a unit suffix optional turned "43.5M" into 43.5.
- Bound it with something you know. A balance below the price of an item the app marks as affordable is a misread.
- Never let one reading overwrite tracked state. Use it for this round only, or require agreement across frames.
- For anything that gates a risky action, require several consecutive readings.

**9. Keep the evidence.**
When a reading matters and looks odd, save the crop. A correct value was once thrown away as an OCR error; the saved crop is what showed it was right.

**10. Re-anchor on ground truth.**
Whatever the bot counts for itself will drift. Look for a mark on the screen that states the truth (a tick on finished items, a "max" label) and reconcile against it regularly. Counting those marks is also the right way to answer "how much is left?".

## Tapping

**1. Find the shortest action that works, then measure the miss rate.**
Where the platform allows fast batches (Android does), they are worth it, but in one build a quarter or more of batched taps did not land. Know your number. Where it does not (on iPhone one verified action costs 1–3 s), design for few actions, each checked.

**2. A tap is a request, not a result.**
Do not count it as done. Check the effect: the marker changed, the balance dropped, the screen moved. Where checking each tap is too slow, reconcile after each batch.

**3. Coordinates expire.**
Anything that scrolls, zooms, opens a panel or changes the screen invalidates every stored position. After such an action, detect again. Before a single important tap, confirm the target is visible right now, at that spot.

**4. Actions that change the screen go last, and one at a time.**
If buying one particular item throws the app to another screen, the remaining taps of the batch land on that other screen. Sort such actions to the end and send only one.

**5. Know which taps have side effects, and undo them.**
A tap on the play field switched a feature from auto to manual. The bot now restores it after every such tap.

**6. Mark the places that must never be tapped.**

| Danger | Defence |
|---|---|
| Reset, delete, purchase and social buttons | With a tree: a never-tap list checked against the target's label before every tap, and the script exits rather than taps. It survives layout changes. Without a tree: exclusion zones in every detector's output |
| A button sitting next to a safe one | Tap the far side of the safe one |
| Notification banners over the app's own buttons | Check the strip for a banner before tapping there; wait if one is present |
| Items partly hidden behind fixed buttons | Move the view so they are clear; never tap through |

A banner check can be very small: a banner is one dark slab, the app's top strip is not.

```python
dark = (strip.max(axis=2) < 110).mean()     # share of dark pixels in the top strip
banner = dark > 0.6
```

**7. Read once before an irreversible choice.**
Positions are good enough for repeatable actions. For a choice that cannot be undone, open the item's description and confirm it is the one you mean. If the check fails, do nothing this round.

**8. Check focus around every batch.**
Before a batch, between chunks of a long one, and after. If the app is no longer in front, stop tapping at once: those taps are now landing on something else.

**9. Hands off other people's things.**
The phone is someone's phone. Messages, calls and dialogs that appear are theirs. The bot waits or stops; it does not dismiss, accept or decline.

**10. Re-check the target before an outward action.**
Between a suggestion and the tap that sends something, the screen can change. Re-read the target's identity first (a name and one piece of content). If it differs, send nothing.

## Keep only what a decision needs

Screenshots, crops, recordings and network dumps can hold the owner's notifications and other people's content.

- Save evidence in the run folder only, and delete it in the wrap-up step.
- Keep content about other people in memory only, by default. Write one item to disk only when the owner asks, and wipe it after.
- On screens that show other people, the owner may ban screenshots entirely. Use the tree's text, or ask.
