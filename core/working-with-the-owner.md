# Changing requirements, and working with the owner

A bot like this is never specified up front. The owner watches the phone and tells you what is wrong, one sentence at a time. In one build nearly every feature began as such a sentence. Treat that as the normal way requirements arrive, and build so that each one is cheap to absorb.

## The kinds of request you will get

| Kind | Sounds like | What it needs |
|---|---|---|
| A correction from watching | "it is tapping in thin air", "it stopped buying the farms" | Confirm in the log first, then fix the cause, not the instance |
| A hint about the app | "zoom out", "there is another area up north", "that tap switched it to manual" | The owner knows the app better than you do. Take it at face value and build it in. |
| A policy | "everything by return on cost", "only buy the bottomless ones if they pay" | A rule in the ranking, stated in their words in a comment |
| Speed | "the taps feel slow" | Measure, change, measure again, report the two numbers |
| Data | "keep everything for next time", "can't you get the data out of the app?" | Recording; recon |
| Risk appetite | "don't risk it this time", "raise the bar", later "forget it, go" | A named gate; a one-off override |
| A pause | "give me my phone back" | Stop cleanly, at once. Resume only on their word. |
| A switch | "turn that off for a while", "turn it back on" | A constant, a reload |
| A question | "why did it do that?", "how many are left?", "how long?" | An answer from the record and the screen. Not a code change. |
| A check | "confirm this, read-only" | Screenshots and log reads only |

## Make each kind cheap

- **Switches at the top of the file, with provenance.**

  ```python
  BUY_BOTTOMLESS = True   # owner's switch: paused 05:45-05:53 on <date> while the last finite ones were bought
  REPEAT_FOR_REWARD = False   # off at the owner's request: measured 2M/s with it, 4-5.6M/s without
  ```

  A year later the comment answers "why is this off?"

- **Reload at a boundary.** Compile, set the stop flag, let the supervisor bring up the new code. Five changes in an hour is routine this way.
- **One-off orders as flag files** the bot consumes, so a single exception does not rewrite the rule.
- **Remembered choices in state**, so the owner decides once ("you pick, I don't care" is also a decision: record it).
- **The record**, so a question is one small script away.

## Question or instruction?

The same sentence can be either. "It seems to be buying only the bottomless ones now" is an observation and a question. It is not yet "turn them off".

| The owner | Do |
|---|---|
| Asks why, or whether, or how many | Investigate and answer. Recommend. Do not change. |
| Points out something wrong | Confirm it, explain the cause, and say what you would change. Change it if it is small, reversible, inside the bot, and plainly serves what they already asked for; otherwise ask. Either way, tell them. |
| Tells you to do something | Do it, verify it, report what the verification showed |
| Says "read-only" | Look only |

When a sentence is cut off or could mean two opposite things, take the reading that costs nothing if wrong, and ask.

## Actions that leave the device

Posting, submitting, sending, buying, reviewing: anything that speaks in the owner's name or cannot be taken back.

1. Show the complete final content.
2. Wait for words that plainly mean "send it".
3. An answer to a question you asked is not that. "The ratings look fine" is not that. Your own earlier sentence "I will submit once you answer" does not turn their answer into consent.
4. Filling a form without submitting is fine. Say that it is filled and waiting.
5. Best of all: draft in place and let the owner press the button.

In the build this came from, one form was submitted on inferred consent and could not be withdrawn. The next one was drafted in place and the owner pressed Submit. The second way took one more message and no apology.

## Things on the device that are not yours

- Notifications, calls, messages: do not open, dismiss, accept or decline. Wait or stop.
- If one affects the run, tell the owner what appeared and what you did (ideally: paused, touched nothing).
- Open the app only when the owner starts a session. If focus is lost mid-run, stop; never take the phone back.
- Tell the owner before taking the phone, and send a progress line every few minutes while you have it.
- Buttons that reset progress, spend money or involve other people are off limits unless the owner names that exact action, each time.

## Settings and infrastructure are the owner's

- **System and security settings** (trusting a certificate, proxy settings, local-network permission, auto-lock, Do Not Disturb or Focus): open the settings pane, and let the owner tap.
- **Infrastructure the owner now depends on** (a proxy, a trusted certificate, a VPN, an automation runner): list it when you set it up, keep a teardown list, and never stop any of it without asking.
- **A shared device:** if another agent or session also drives the phone, check before starting, tell it, and say when you are done.

## Measured, inferred, guessed

The most damaging sentences in the build were confident and unverified:

- "that reward was not credited" (it had been, and spent);
- "we lost that attempt" (nobody saw it end);
- "that stage lasts six minutes" (it took under three).

Before stating something, know which it is:

| It is | Say |
|---|---|
| Measured | The number, and where it came from |
| Inferred | "This looks like X, because Y" |
| Guessed | "A guess" |

And when the owner corrects a fact about their own history with the app, they are right. Update and move on.

## Announce what will look strange

If the bot is about to do something an onlooker would read as a malfunction (repeat a finished stage, stop buying for a while, sit idle), say so first, with the reason and how long.

## Report shape

When asked for status, lead with whether it is fine, then the few numbers that matter, then anything the owner has to do. When something went wrong, lead with that. Keep working state (what you tried, the scaffolding) out of it.

## Keep the owner out of process work

If the owner has to background your waits, ask why it is stuck, or poll you for status, the tooling is missing something. Each of those is a prompt to add a supervisor rule, a monitor, or a notification, so it does not happen twice.
