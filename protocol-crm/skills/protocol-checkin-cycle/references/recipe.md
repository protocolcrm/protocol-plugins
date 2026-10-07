# Recipe: weekly check-in and progress report

Turning a client's check-in data into a report the client will actually see, without
overstepping into the coach's own approval decision.

Verb + param source: **[surface-clients.md](../../protocol-reference/references/surface-clients.md)**.

## Sequence

1. **Gather this client's recent context.** Either or both:
   ```
   review_client clientId=<clientId>
   ```
   for the full bundle (profiles, programs, progress, appointments, tasks, insights in one call),
   and/or:
   ```
   find kind=progress clientId=<clientId>
   ```
   to list recent check-in entries directly. If there's already an AI-drafted report waiting on
   this client, locate its id with:
   ```
   find kind=report clientId=<clientId>
   ```

2. **If the client hasn't logged a check-in yet and the coach wants one recorded on their behalf**,
   create or update the entry:
   ```
   record_progress action=entry clientId=<clientId> entryDate=<date> measurements={...} userNotes=<text> trainerNotes=<text>
   ```
   `action` is required (`entry`|`report`|`note`); for the entry sub-surface, `progressEntryId` is
   only needed when updating an existing entry rather than creating a new one.

3. **Draft or refine the client-facing report**, using round, realistic numbers throughout
   (see Gotchas):
   ```
   record_progress action=report reportAction=update reportId=<reportId> clientFacingSummary=<text> sections=[...]
   ```
   `reportAction` here is `update` — this is the safe, edit-only path. Write like a coach: prefer
   clean numbers ("down about 1.5 lb this week", "averaged ~7 hrs sleep") over exact-looking
   decimals that just make up a target.

4. **Stop here.** Do not call `reportAction=approve` yourself unless the coach has explicitly told
   you to send this report to the client. Approving is a plain write that takes effect the moment
   you call it: Protocol flips the report to APPROVED and it is visible in the client's app
   immediately, with no push notification and no confirmation step in between. Nothing on the
   platform holds it back for a coach's sign-off; that pause exists only because you wait for one.
   Treat it as the coach's decision, not the agent's default. If instructed to send it:
   ```
   record_progress action=report reportAction=approve reportId=<reportId>
   ```
   Several the coach reviewed at once (say, over the weekend) go in one call, and the response
   lists each report approved or skipped with its reason - read the skips back to the coach:
   ```
   record_progress action=report reportAction=approve_many reportIds=[<id>, <id>, ...]
   ```
   Both are silent unless the coach also wants the client pushed: `notifyClient: true`, which needs
   a `send`-tier connection.
5. **A report that went out wrong** is taken back with `reportAction=unsend reportId=<reportId>`:
   it returns to DRAFT and leaves the client's app at once. Correct it with `update`, then approve
   it again; the client is never pushed twice for the same report.

## Gotchas

- **`reportAction: approve` or `discard` refuses any edit fields sent in the same call.** If you
  need to change `clientFacingSummary`/`sections`/etc. *and* approve or discard, that's two calls:
  `reportAction=update` first, then `reportAction=approve`/`discard` as a separate call. Bundling
  them is refused and nothing is approved.
- **Realistic round numbers, always** — the server's own connection-time instructions (restated in
  `../../protocol-reference/references/guardrails.md`) prefer a tidy number slightly off target
  over an exact one built from awkward fractions; this applies directly to anything you write into
  `clientFacingSummary` or `measurements`.
- **Approval is the coach's call, not the agent's default.** Draft and refine freely with
  `reportAction=update`; never chain straight into `reportAction=approve` on your own initiative.
- **`clientFacingSummary` is read by the client, so the claims rule binds it hardest.** Trends and
  ranges against this person's own history, wellness framing, no "abnormal"/"out of range" verdict,
  no named disease, and "worth raising with your doctor" wherever a result could concern them. See
  "Claims and intended purpose" in `../../protocol-reference/references/guardrails.md`.

## See also

- `../../protocol-reference/references/surface-clients.md` — `record_progress`'s full param list.
- `../../protocol-reference/references/pitfalls.md`: the report-editing pitfall (see
  "Action-specific fields belong to their action").
- `../../protocol-reference/references/guardrails.md` — house style (round numbers, mirror the
  coach) and the coach-approval pattern for client-facing outputs.
- [find and review a client recipe](../../protocol-client-review/references/recipe.md) — the
  read-tier lookups this recipe builds on.
- [build and assign a program recipe](../../protocol-build-program/references/recipe.md) — the
  program this progress is usually measured against.
