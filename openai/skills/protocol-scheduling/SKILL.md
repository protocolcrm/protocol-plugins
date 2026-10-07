---
name: protocol-scheduling
description: Use when the operator wants to book, move, or cancel an appointment, set up check-in reminders, or configure booking in Protocol - so routine calendar work runs freely while anything that pings a real client waits for a human tap.
---

# protocol-scheduling - the calendar, and the one part that reaches a client

Almost everything in `schedule` is internal calendar work. Two actions are not: they push a
notification to the client's phone. Know which is which before you call it.

## When to use

- "Book Sarah for Thursday", "move my 3pm", "cancel Friday"
- "Set up weekly check-in reminders", "change my booking hours"
- "Remind him about tomorrow" - this one reaches the client.

## Steps

1. Read the client first if the appointment concerns one (see `../protocol-client-review`).
2. Find the existing appointment with `find` (`kind: "appointment"`) before updating or cancelling -
   never act on a guessed id.
3. Call `schedule` with the right `action`:
   - **Internal, runs autonomously, works at the `write` tier:** `create`, `update`, `cancel`,
     `booking_config`, `gcal_disconnect`
   - **Reaches the client, waits for approval, needs the `send` tier:** `send_reminder` (pushes
     now), `reminder` (arms a scheduled push - it fires within minutes if the start time is now or
     past). If this connection is at `write` (the default), expect a refusal, not a silent no-op.
4. When an action needs approval, tell the operator plainly what will be sent and to whom, and wait
   for their explicit yes before making the call. There is no built-in confirmation step on this
   path, so that pause has to come from you. There is a real tier check, though: `send_reminder`
   and `reminder` both need the `send` tier (see
   `../protocol-reference/references/guardrails.md`), and if this connection is at the default
   `write`, the call is refused outright with a `PermissionDeniedError`. That refusal is normal, not
   a bug - tell the operator plainly which access level is needed and stop. Do not treat it as a
   missing feature, and do not look for another route to send it anyway; that would override the operator's own choice at consent.
5. Confirm what changed.

## Rules

- `cancel` does **not** tell the client - it only changes the appointment's status. If the operator
  expects the client to be informed, say that it will not happen automatically.
- `reminder` is not a harmless setting. It arms a real push; treat it as outward, because it is.
- Never batch outward reminders across many clients without saying how many will be sent.
- **A weekly check-in is `reminder` with an RRULE**: `clientId` + `formId` + `startTime` (the first
  occurrence) + `recurrenceRule` like `FREQ=WEEKLY;BYDAY=MO`. Use the client's existing check-in
  form; the client has 48 hours to answer by default.
- **`booking_config` is the coach's public page. Read it before you write it** (`read: true`).
  `globalSettings` merges, so send only what changes - but `sharedAvailabilities` and
  `eventConfigurations` are whole-array replaces, so leave them out unless you mean to rewrite them.
  A real coach has around 66 availability slots; a short list deletes the rest.
