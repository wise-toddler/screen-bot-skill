# Deciding

Two kinds of app need two kinds of deciding:

- **An app with an economy** (most games): the bot decides alone. Ranking, budget, gates and payback, below.
- **An app where a person decides** (social, shopping, messaging): the bot suggests and the owner picks. See Human in the loop, at the end. Do not apply the economy rules to these apps.

## With an economy

Start with the dumbest rule that makes progress ("take whatever is available"). Add each refinement below when the need shows up. They are listed in the order they were needed.

### Ranking

**Value per cost.** For each available action: how much better does it make the thing it affects, relative to now, divided by its price.

```
score = weight(what it improves) x relative_gain(one more level) / price
```

- **Relative gain**, not absolute. "+20%" on something already at +400% is a 4% improvement.
- **Weights** say what matters most. Things that raise income come first because they pay for everything else.
- **Weight by real contribution.** For upgrades to one of several units, multiply by that unit's measured share of the total. An upgrade to something that does nothing is worth nothing.

Three corrections that each cost time to discover:

| Problem | Symptom | Fix |
|---|---|---|
| Unlocks have no "gain" to score | Things that open new content waited behind small upgrades | Rank unlocks first, outside the score |
| A new unit has zero share | Its upgrades scored twenty times too low and got only leftovers | Give new things a starting weight until they have a measured share |
| Endless upgrades always have a score | They absorbed a growing share of the budget (8% to 31% in twenty minutes) | Give them only what is left after finite ones, or require a payback time |

**Payback for bottomless sinks.** An upgrade with no cap is worth buying only while one level pays for itself quickly:

```
payback_seconds = price / (income_per_second x relative_gain)
```

A limit of ten minutes worked. This needs a measured income rate, so measure it: the balance at the end of one visit and at the start of the next.

### Budget

Ranking is not enough. The balance is spent in rank order, so whatever is ranked last gets nothing, however cheap and important.

1. **Set aside** the cost of unlocks and of anything that opens new content, before anything else.
2. **Spend down the ranked list** against the balance read on screen.
3. **Send multi-level purchases as one batch.**
4. **Re-check** before each single important tap that it is still affordable.

If the balance cannot be read this round, do not guess. Give each item its small default and let the next round correct.

The order of tapping and the order of budgeting are separate. An action that must be tapped last (because it changes the screen) can still be budgeted first.

### Gates for risky actions

Some in-app actions cannot be retried, or cost a lot when they fail. Put each behind a gate: a condition that must hold first.

**Write the gate as a named constant with its origin:**

```python
LAST_STAGE_MIN_POWER = 20_000_000   # set by the owner on <date>; the earlier attempt was at about 3M
```

**Say what kind of number it is.**

| Kind | Example | How much to trust it |
|---|---|---|
| Derived from the app's data | "Stage health from the table, times 1.3, over the time its waves take" | High, if the table is verified |
| Calibrated on similar past events | "Stages of known size took this long at this power" | Good |
| A multiple of a remembered number | "Four times what we had when we think we failed" | A guess. Say so. |

In the build this came from, the final gate was the third kind, and the remembered failure turned out never to have been observed. The attempt, when it finally happened below the gate, was not close.

**Check what your measure really measures.** The on-screen "total power" figure used for the gate turned out to be:

- net of a negative term (enemies healing each other), and
- limited by how many targets were present, so it fell when the bot was doing well.

Before gating on a number, read every row that adds up to it.

**Make the gate robust to one bad frame.** Require several consecutive readings. And make the identification of the risky action itself robust: in one case, a single undetected button would have made the bot treat the final stage as an ordinary one and apply the weak gate.

**"Nothing left to do" is harder than it looks.** "No affordable action right now" is true whenever the balance happens to be spent. If you mean "everything is finished", test for that: count the finished marks.

**Leave room for the owner to override.** A one-off flag file that the bot consumes is better than editing the constant: the gate stays in place for next time.

### Experiments

When you do not know whether a behaviour pays, measure it.

1. Put it behind a switch that defaults to off once the answer is in.
2. Alternate: some cycles with it, some without, and record the same measure for both.
3. Decide on a clear margin, with a few samples each.
4. **Tell the owner before it starts.** A bot that suddenly repeats something it has already done looks broken.

Example: re-doing a finished stage for its reward. Cycles with it earned about 2M per second, cycles without 4 to 5.6M. Two samples each were enough.

### Estimates

The owner will ask "how long until it is done?"

- **Count remaining work from the screen**, not from the bot's own tally.
- **Use measured rates** from the record, not the theoretical ones.
- **Give a range and say what it rests on.** "25 to 35 minutes, from the price table and the last ten minutes of income; the bot's level count may be high, which would lengthen it."
- **Say what the estimate leaves out** (bottomless upgrades, a gate that may not open).

A first estimate built on the bot's tally was wrong in its details; a tick count from screenshots a few minutes later was exact.

### Waiting is a decision too

In anything with income, doing nothing for a while is often the best move. Make the wait depend on what just happened:

| After | Wait |
|---|---|
| A long action during which income piled up | None |
| A round that bought a lot | Short |
| A round that bought little | Longer |
| Nothing left to buy | Longest, growing while nothing changes |

## Human in the loop

For apps where every action speaks for the owner:

1. **Suggest.** Read the screen (prefer the tree's text), and offer options with the reasons.
2. **The owner picks.** One item, the exact text.
3. **Re-verify the target** right before acting. Re-read its identity; if it changed, do nothing.
4. **Act exactly that.** One action at a time. Paste text instead of typing it fast, and read it back before submitting.
5. **Report** what the screen shows afterwards.

Paid or scarce actions happen only when the owner names that action for that item. Never auto-act, never bulk, never "skip" on the bot's own judgement. Upsell dialogs get their decline button and nothing more.
