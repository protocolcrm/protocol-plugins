---
name: protocol-reporting
description: Use when the operator asks how a month went, which clients are progressing or stalling, whether their programming is advancing, what a client is eating or tracking, how engaged they are, or where their purchases stand - so the answer comes from aggregated history with its real coverage stated, and the things this data cannot support are named rather than guessed at.
---

# protocol-reporting - say what the data supports, and no more

`report` aggregates history. It is the only surface that can answer "how did this month go". It is
also the easiest surface to narrate dishonestly, because sparse data and bad news look identical
once they are summarised.

Only `kind=training` produces verdicts. The other five hand you data: `checkin`, `body`,
`nutrition`, `engagement`, `business`. Four of them deliberately withhold a headline you would
expect, because the underlying data cannot support it, and each says so in its own `notes`. Those
refusals are the most important thing this skill has to teach, so they are listed first.

## Four things this data cannot tell you, however the question is phrased

A coach will ask for each of these in plain language. Say what you actually have, say why the rest
is missing, and stop. Do not estimate, infer, or reach for a proxy.

**"Is she hitting her macros? How is her adherence?"** There is no target. Not on the log, not on
the client's nutrition profile, not through a template link. `kind=nutrition` can tell you what was
logged and how many of the window's days carry a log at all, and that is the whole answer. What you
can offer instead: consistency. "She logged on 9 of the last 30 days, averaging about 1,900 kcal on
the days she logged" is true, useful, and does not pretend a target exists.

**"Did he actually turn up? What is his no-show rate?"** Appointment status is never transitioned
in this system, so almost every past appointment still reads as scheduled. Any attendance figure
computed from it would say nobody ever attends. `kind=engagement` gives you appointments **booked**,
with reminder records counted separately from real meetings. Say "three sessions were booked" and
never "three sessions happened".

**"What is this client worth per month? What is our MRR?"** No purchase carries subscription
linkage, and most do not record whether they recur. `kind=business` gives you purchases, their
status, their expiry dates and what has actually been invoiced and paid. Expiry is the real signal
and it is often the more useful one: "his package expires in 11 days" is what a coach can act on.

**"Are these numbers healthy? Is she in range?"** `kind=body` returns an `optimalRange` for most
metrics, and it is the same band the coach already sees in the app. It is reference data, not a
clinical judgement, and you are not the one to turn it into one. Watch `weight` in particular: its
band is computed from the client's **own** observed range, so a weight inside it means their weight
has been steady, not that it is healthy. Relay the band, relay the number, let the coach judge.

This one is also a compliance rule, not only an honesty rule. Protocol is a wellness and
optimization platform; it does not diagnose, treat, cure or prevent any disease, and nothing you
narrate may imply that it does. So: read every metric as a **trend and a position in a range**
relative to this person's own history, never call a value abnormal, critical, high-risk or out of
range against a clinical threshold, and never name a disease as something the numbers show, rule
out, or predict. Where a value could reasonably concern someone, say so plainly and suggest they
discuss it with their doctor - do not soften it into reassurance and do not inflate it into alarm.
Full rule: `../protocol-reference/references/guardrails.md`, under House style.

## Two ways `kind=body` and `kind=nutrition` will mislead you if you skim

**A metric can be history rather than tracking.** Most stored health data arrived through a one-off
bulk import, not a live device sync. Every metric carries `sources` and `backfilledOnly`. If
`backfilledOnly` is true, the client is not tracking that metric now, whatever the dates suggest,
and saying "she has been tracking her sleep all year" would be false. Check the flag before using
the word "tracking".

**A day with no nutrition log is a day nothing was logged.** It is not a day nothing was eaten.
`daysLogged` and `daysInWindow` are both on the row so the difference stays visible, and the means
are computed over logged days only. A client who logged two 2,000 kcal days in a month averaged
2,000 on the days they logged, and reporting that as a daily average across the month would put
them at 133 kcal a day and read as a medical emergency.

Also: `kind=engagement` sees in-app messages only. Coaches use WhatsApp too, so silence there is not
evidence of no contact, which is why response times are deliberately not computed. And every money
figure in `kind=business` is in **cents**, keyed by currency. Divide by 100 before saying an amount
out loud, and never add usd to cad.

## Scan the roster, then drill

Start at `report kind=training subject=roster`. Scan for rows carrying flags: `no_sessions_in_window`,
`has_regressions`, `has_data_anomalies`. Then call `report kind=training subject=client
clientId=...` for the two or three that look wrong.

Never loop `subject=client` across a roster. That is dozens of calls for something one call answers.

## Sparsity is normal, and it is not bad news

Coaches rotate exercises. A typical client logs about 10 sessions and 24 distinct exercises in 30
days, and only about a third of those exercises clear the 3-session minimum a verdict needs. Below
that minimum, an exercise never becomes a row at all: it is dropped before scoring and folded into
a single count, surfaced (for `subject=client` only) as `limits.collapsedBelowMinSessions` plus a
matching line in `notes` that says this plainly: exercise rotation, not a stall, and widening the
window will not fix it. A roster row's per-exercise counts carry no such number, so a roster looks
thinner than the client's real training was; that gap is exactly what drilling into `subject=client`
is for.

Three rules follow, and they are correctness rules, not style:

- **A collapsed exercise is not "flat", "not progressing", or "insufficient_data" written on a row.**
  It is simply absent from `rows`. Read `limits.collapsedBelowMinSessions` and the note before you
  describe a client's exercises, and say plainly that N exercises did not log enough sessions to
  trend. Never let that count silently disappear into the verdict tally, and never claim it as a
  stall: saying a client stalled when they merely rotated exercises is a false claim about a real
  person's training.
- **Never widen the window to chase coverage.** Density plateaus near a third at 30, 60 and 90 days,
  because a wider window adds newly rotated exercises as fast as it deepens existing ones. You will
  burn a call and get the same sparsity.
- **Lead with what is solid.** Session count, total volume load, and the prescribed audit hold for
  every client. Per-exercise verdicts are the garnish, not the meal.

## The prescribed audit has fixed wording

When `prescribed.changed` is false, you may say:

> "The same set and rep scheme every time this exercise appeared in the program."

You may **not** say the coach never progressed it. Across the platform, a prescribed weight is
recorded on only 4.7% of prescribed exercises, and 0% in the single largest tenant: for the large
majority of clients, `prescribed.comparable` will read false every time, because Protocol simply
never captured a load figure to compare. A coach who progresses load in person, or who lets the
client autoregulate, is indistinguishable in this data from a coach doing nothing. Asserting the
latter is both wrong and insulting to the person reading it. Check `prescribed.comparable` before
you say anything about load at all; if it is false, talk about sets and reps only.

Two more things this block does not say, and you must not say for it:

- **Check `prescribed.appearances` before quoting `changed` at all.** At one appearance there was
  nothing to change, so `changed: false` is arithmetic, not a fact about the coach. The 96.4%
  agreement figure behind this audit was measured on exercises appearing three or more times. At
  one or two, say how many times it appeared and stop.
- **The audit is not scoped to your window.** It covers the client's current programming, which is
  the set of non-expired programs overlapping the window, counted across the whole plan rather
  than these dates. `changed: true` means the prescription differs somewhere in that plan, never
  that the coach changed it during the month you asked about. The `notes` line says the same
  thing; relay it rather than tightening it into a date claim.

## `data_anomaly` means ask, not report

A verdict of `data_anomaly` fires on a swing bigger than 50% between the first and last logged
value in the window, too large to be physiological within that time, almost always a logging
correction, like an empty bar recorded before a real load. Show the coach the number and say it
looks like a logging artifact. Do not present it as progress.

## Narrate by basis

Check `basis` before choosing a verb. `e1rm` and `volume` are load-based, so "lifted more" is fair.
Anything else is not: never say a client "lifted more" on a plank or a carry.

## `previousWindow` is echoed for reference only, and no delta is computed against it

For `kind=training` the envelope carries a `previousWindow` block with real dates on it. Every other
kind reports `previousWindow` as null. Nothing is measured over that window, for any kind. History is read exactly once, for the window in `window`, and no
figure anywhere in the response is a comparison against the preceding period: `volumeDeltaPct` on
a roster row is always `null`, and there is no previous-period session count, volume figure or
verdict in the payload at all. `compareToPrevious` only decides whether those reference dates are
echoed.

So never narrate a change against last month. A sentence putting this window's session count
against a named earlier month, or claiming volume rose or fell relative to the previous window,
is a statement this surface cannot support, and inventing one attributes a trend to a real person
that nobody measured. If the coach asks how this month
compares to last, say the report covers one window at a time and offer to run the earlier window
as a second call, so both numbers are ones you actually read.

## Read `coverage`, `notes` and `limits` before writing a word

`coverage` says how much of what you asked for had data. `notes` explains the gaps in plain
language. `limits` appears only when the answer is degraded. If any of them are non-empty, the
coach hears about it in your first two sentences, not in a footnote.

## When NOT to reach for REST

Sparse data is never a reason to escape to the REST API. The rows are not hiding; they do
not exist. Escaping to REST to re-derive the same aggregate by hand will produce the same sparsity,
slower, with no verdict rules and no tenant guardrails.

## Worked example

`report kind=training subject=client clientId=... from=2026-07-01 to=2026-07-31` returns 9 exercise
rows and `limits.collapsedBelowMinSessions: 7`. Of the 9: four are `progressing`, three `flat`, one
`regressing`, one `data_anomaly`.

A good answer:

> Across July she logged 11 sessions, over nine exercises with enough sessions to trend.
> Bench press and leg extension are both moving well, up 20% and 37% on estimated top-end strength.
> Machine row and incline dumbbell press have not moved in five sessions each, so those are worth a
> program change. Deadlift is down 6% over four sessions on both strength and volume, worth asking
> about before assuming it is a program problem. One flag: hyperextension shows a 100% jump in three
> sessions, almost certainly a load logged against an empty bar rather than real progress, worth a
> look before anyone calls it a win. Seven other exercises appeared once or twice each this month,
> too few sessions to say anything about, which is normal given how much she rotates.

Note what it does not do. It does not fold the seven collapsed exercises into the verdict counts or
call them stalled, it does not present the anomaly as a win, it asks about the regression instead
of declaring it a failure, and every percentage in it is a within-window first-versus-last figure
the response actually carried. There is no "up from last month" anywhere, because no such number
was computed.

## `kind=checkin` is data, not a verdict

`report kind=training` hands you conclusions. `report kind=checkin` hands you the client's own
words and computes nothing about them. That difference is deliberate: most check-in questions are
free text a coach wrote themselves, often not in English, and any server-side scoring of that text
would throw away the signal. Read it and draw your own conclusion, out loud, so the coach can
disagree with it.

Three rules follow, and they are correctness rules, not style:

- **Never paraphrase an answer into a verdict and attribute it to the client.** "She said her sleep
  is bad" when the answer was "malo bolje nego prosli put" is a false statement about a real person.
  Quote, or say what you actually read.
- **`internalNotesPrivate`, on entries and on prior reports, is the coach's private note.** It exists
  so the coach can see their own thinking. It must never appear in anything drafted for the client.
- **Check `intake.source` before calling anything a starting goal.** `tagged` is a real intake
  questionnaire. `earliest_fallback` means no intake was ever tagged and you are looking at the
  client's oldest form of any kind, which may be a contact form with nothing but a name in it.

### Read the cadence before you call anyone overdue

`overdue` compares this client's gap against their own median, not a global schedule, because
check-in rhythm is a per-coach convention. The median counts only `REMINDER` entries: a client who
logs an ad-hoc weigh-in has not checked in, and a coach-created entry is the coach's own record.
`cadence.unknownSourceEntries` counts rows predating that distinction; when it is high, the median
is thin and the `overdue` flag is weak evidence.

### Prior reports are what you already said

`priorReports` carries the coach's own previous write-ups, newest first, with `dispatchedToClient`
and `openedAt`. Use them. A check-in summary that repeats last month's advice without noticing it
was already given reads as though nobody is paying attention. Ask instead whether what was
recommended actually happened.

### Scan, then drill

`report kind=checkin subject=roster` gives one thin row per client: cadence, the headline
measurement deltas, and flags (`no_checkins_in_window`, `overdue_checkin`, `never_checked_in`).
It deliberately carries no answers, no entries and no messages, because a 40-client roster would
otherwise return roughly 2,000 question-answer pairs. Drill with `subject=client` for the two or
three that look wrong.

### A degraded field is not an empty field

`coverage.degradedSections` on `subject=client`, and `data_incomplete` on a roster row, both mean
the same thing: a read failed, so that section came back empty for a reason that has nothing to do
with the client. Do not describe a degraded section as "nothing to report" or "no data": say the
read failed and that the emptiness is not evidence of anything about the client.
