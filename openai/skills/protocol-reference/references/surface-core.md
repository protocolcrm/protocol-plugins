# Protocol MCP surface - core: reading, and the replace grammar

The two read verbs every job starts with, the entity kinds they reach, and the shared write
grammar that array parameters follow across the whole surface. **Read this one first** - the
domain files assume it.

> **One of four.** The surface is split by the job you are doing, so you read the part you need
> rather than all 23 verbs:
>
> | File | Verbs |
> |---|---|
> | `surface-core.md` | `find` · `get` · `report` · the kind table · the replace grammar · `report_to_developers` · `guide` |
> | `surface-programming.md` | `build_program` · `build_workout` · `build_nutrition` · `assign_program` · `manage_library` |
> | `surface-clients.md` | `manage_client` · `record_progress` · `manage_forms` · `review_client` · `message` |
> | `surface-operations.md` | `manage_tasks` · `manage_media` · `manage_content` · `schedule` · `manage_automations` · `review_inbox` · `manage_support` · `manage_shop` |


The whole served surface is **exactly 23 intent verbs**. There are no other tools. Each verb
reshapes your input and forwards it to Protocol's internal layer, so **parameter names are exact** —
see `pitfalls.md` for why a wrong key is worse than an error.

Transport: MCP over remote HTTP, one route (`POST /mcp`), stateless — no session state survives
between calls. Auth is a per-coach key; every call runs tenant-scoped to that one coach.

Tiers exist (`read` < `write` < `send`, cumulative), and this connection is not exempt from them:
the coach chose one of them at consent when they installed, and `write` is the default. Enforcement
is two-layered: the tool list itself is tier-filtered (a `read` connection's `listTools` never
includes the write verbs at all - they are **absent**, not refused), and a call above the
connection's tier is also refused at call time, as defense-in-depth. See `guardrails.md` for the
full model and what to do about either. Every verb in this file needs at most `write` tier
(`find`/`get`/`review_client`/`report`/`message`/`guide` are `read`; `report_to_developers` is `write`) -
so if the coach chose `read` only, `report_to_developers` itself will simply be absent from your
tool list, not denied when you try to call it. (`message` lists at `read`, but its one write,
`action=draft`, needs `write`.)

## Verb index

| Verb | Tier | Purpose |
|---|---|---|
| `find` | read | List/search entities of one kind. |
| `get` | read | Fetch one entity by id, full detail. |
| `review_client` | read | One-call full picture of a single client. |
| `report` | read | Aggregate a window of history into a report. Six kinds: `training` gives verdicts, the other five give structured data with no judgement attached. Several report explicitly what they cannot compute. |
| `message` | read* | Read the inbox and conversations; `action=draft` leaves a draft for the coach (write). Never sends. |
| `guide` | read | These playbooks and reference files, served by the server itself. |
| `manage_client` | write | Create/update a client, its stage, trainer, access tier, and 4 profiles. |
| `build_program` | write | Create/edit a program's structure (metadata, phases, content). |
| `assign_program` | write | Assign (deep-copy) a program to a client, or flip its lifecycle. |
| `build_workout` | write | Create/edit a workout: metadata + full exercise tree. |
| `build_nutrition` | write | Create/edit a nutrition template: metadata + full item tree. |
| `record_progress` | write | Check-in entry, progress-report triage (incl. batch approve and unsend), or meeting note. |
| `manage_library` | write | Custom exercises; batch-resolve food names. |
| `manage_forms` | write | Create/update an intake / check-in / assessment form. |
| `manage_tasks` | write | The whole kanban surface (tasks, subtasks, boards, columns, labels). |
| `manage_media` | write | Media library: attach, edit, categorize, share. |
| `manage_content` | write | Articles: author from Markdown, edit, publish (claims-gated), attach to a program, duplicate, tag and file, share by public link, delete (confirm-guarded); save, edit and delete reusable audiences. |
| `review_inbox` | write | The coach's "what needs me" bundle + triage flips. |
| `manage_support` | write | Comment on / (Protocol only) move the status of a filed support ticket. |
| `report_to_developers` | write | Escalate a gap. Emails a fixed internal inbox, never a client — and now files a tracked ticket. |
| `schedule` | write* | Appointments, check-in reminders, booking config, send a reminder now. |
| `manage_automations` | write* | Build/operate automations; `run` dispatches an execution now. |
| `manage_shop` | write* | Shop bookkeeping: record a sale to a client (`create_purchase`, send) or money received (`record_payment`). Never charges a card. |

\* **The tier split on these verbs is per-ACTION, not per-verb.** `schedule`, `manage_automations`
and `manage_shop` list at `write` because most of their actions are ordinary internal writes:
booking an appointment, reading the booking config, authoring an automation, recording a payment.
Four actions are held to `send` because they reach a client: `schedule action=send_reminder` (fires
now), `schedule action=reminder` (arms a recurring push), `manage_automations action=run`
(dispatches an execution whose post-actions can email or WhatsApp), and `manage_shop
action=create_purchase` (bills the client: it shows in their app and arms payment and expiry
reminders). `message` works the other way round: it lists at `read` so a read-only connection keeps
the inbox, and its one write, `action=draft`, needs `write`. If this connection is below the tier
an action needs, the call is refused by the platform with a `PermissionDeniedError` naming the tier
- tell the coach plainly and stop, per `guardrails.md`. Never treat that refusal as a reason to fall back to a direct API key.

---

## Read tier

---

### `find` — list/search one kind

Required: `kind`.

| Param | Type | Notes |
|---|---|---|
| `kind` | string | **Required.** One of the 34 kinds below. |
| `query` | string | Free-text search (where the kind supports it). |
| `clientId` | string | Filter to one client (where supported). |
| `formId` | string | `kind=submission`. |
| `isTemplate` | boolean | Templates vs client-assigned (program / workout / nutrition). |
| `scope` | string | **Whose library.** `mine` (yours, unshared), `team` (shared with the team), `all` (default). Applies to `kind=workout` / `nutrition` / `program` / `exercise` / `media`. |
| `status` | string | Status filter (where supported). |
| `limit` | number | Result cap. |
| `muscleGroup` | string | `kind=exercise` — primary muscle group (e.g. CHEST, BACK, THIGHS). |
| `exerciseType` | string | `kind=exercise` — STRENGTH, CARDIO, FLEXIBILITY, … |
| `difficultyLevel` | string | `kind=exercise` — BEGINNER … ELITE. |
| `isCompound` | boolean | `kind=exercise` — compound (true) vs isolation (false). |
| `automationId` | string | `kind=automation_run` — **required for that kind**. |
| `specialPurpose` | string | `kind=submission` — CHECK_IN, INITIAL_QUESTIONNAIRE, SURVEY, OTHER. |
| `labelNames` | string[] | `kind=client` — clients carrying these labels, **by name**. Enumerate them with `find kind=client_label`. A name that matches no label returns **nobody**, not everybody. |
| `labelMatch` | string | `kind=client` — `any` (default) carries at least one of `labelNames`; `all` carries every one. Use `all` for a segment like VIP *and* marathon-prep. |
| `from` / `to` | string | `kind=appointment` / `health_metric` / `client_history` / `workout_session` — the window, `YYYY-MM-DD` or ISO-8601; a bare `to` date includes that whole day. |
| `types` | string[] | `kind=client_history` — only these event types (`stage.changed`, `label.added`, `label.removed`, `reminder.assignee_changed`, `program.assigned`, `program.status_changed`, `checkin.submitted`, `form.submitted`, `purchase.created`, `purchase.status_changed`, `purchase.renewed`, `invoice.paid`, `email.sent`, `email.failed`, …). An unknown type is refused with the valid list. |
| `order` | string | `kind=client_history` — `desc` (default, newest first) or `asc`. |
| `cursor` | string | `kind=client_history` — the `nextCursor` of the previous page. This kind pages by cursor, **not** `offset`. |
| `includeSets` | boolean | `kind=workout_session` — every exercise and set (see *What the client actually did* in `surface-clients.md`). |
| `folder` | string | `kind=client_media` — a folder key from the tree (`checkins`, `checkins/2026-09`, `forms/<formId>`, `chat/2026-09`, `nutrition/2026-09`, `profile`); omit for the tree. |
| `favorites` | boolean | `kind=exercise` — only the exercises this coach starred. Every exercise row carries `isFavorite`. |
| `tags` | string[] | `kind=exercise` — tenant exercise tags as `key:value`; an exercise must carry **all** of them. |
| `variants` | string | `kind=exercise` — `all` lists every gender/version variant of a public exercise (default: one row each). |
| `includeArchived` | boolean | `kind=habit` — also the coach's retired custom habits. |
| `installmentState` | string | `kind=purchase` — installment plans that are `OVERDUE`, `DUE_SOON`, `ON_TRACK` or `FULLY_PAID`. |
| `expiresBefore` / `expiresWithinDays` | string / number | `kind=purchase` — access ends on or before a date, or within the next N days: the renewal list. |
| `unread` / `hasDraft` / `reminderAssigneeId` | boolean / boolean / string | `kind=conversation` — the inbox filters (also on `message action=list`, with `labelNames` and `lifecycleStageId`). |

Filters that a given kind's underlying list doesn't support are **ignored silently** — you get the
unfiltered list, not an error.

**Whose shelf.** The library has two: what this coach owns and hasn't shared, and what the team
has shared. Rows come back carrying their own `visibility` (`PRIVATE` / `TEAM`), so you can always
tell the coach where something came from — say so when you build a plan out of a colleague's work.

Ask with the one word. Do **not** try to compose it out of `ownerId` and `visibility` yourself:
"mine" is an owner *and* a visibility, and half of it is a different question rather than a
narrower one — a visibility with no owner matches every private row in the tenant, which is other
coaches' unshared work. The translation lives on the server.

---

### `get` — fetch one entity by id

Required: `kind`, `id`.

| Param | Type | Notes |
|---|---|---|
| `kind` | string | **Required.** One of the 19 `get` kinds below. |
| `id` | string (uuid) | **Required.** The entity id. `get` maps it onto the right id param for you. |

---

### `report` — aggregate a window of history

Required: `kind`.

| Param | Type | Notes |
|---|---|---|
| `kind` | string | **Required.** One of the report kinds below. |
| `subject` | string | `roster` \| `client`. Defaults to `client` when `clientId` is set, else `roster`. |
| `clientId` | string | Required when `subject=client`. |
| `from` / `to` | string | Window bounds, `YYYY-MM-DD`. `to` defaults to today; `from` defaults to 30 days before `to`. |
| `compareToPrevious` | boolean | `kind=training` only: echoes a `previousWindow` block of reference dates. Default `true`. **No delta against it is computed** anywhere in the response, for ANY kind, so never narrate one. Every other kind reports `previousWindow` as null. `kind=checkin` does compute `deltaFromBaseline`, but that is measured against the client's first entry on record, not bounded by the window and not a preceding window; `kind=body` computes a delta across the window it was given. |
| `limit` / `offset` | number | Roster paging (max ~200 rows per page). |
| `entriesCount` | number | `kind=checkin`: how many entries to return. Default 6, max 20. |
| `messagesCount` | number | `kind=checkin`: how many recent messages. Default 20, max 50, `0` to omit. |
| `priorReportsCount` | number | `kind=checkin`: how many of the coach's previous reports. Default 2, max 5, `0` to omit. |
| `includeIntake` | boolean | `kind=checkin`: include the initial questionnaire. Default `true`. |
| `includeProfile` | boolean | `kind=checkin`: include the client profile snapshot. Default `true`. |
| `seriesPoints` | number | `kind=body`: daily points per metric. Default 30, max 90, `0` to omit the series. |
| `seriesDays` | number | `kind=nutrition`: days of daily series. Default 60, max 180, `0` to omit. |
| `timelineDays` | number | `kind=engagement`: days of contact timeline. Default 60, max 180, `0` to omit. |
| `purchasesCount` | number | `kind=business`: purchases returned, newest first. Default 20, max 100, `0` to omit. |

**Whose clients.** The client set is decided by the account's permissions, the same rule `find`,
`get` and `review_client` use: an owner or admin reaches every client in the team, a coach reaches
the clients assigned to them. There is no parameter that widens or narrows it. A `clientId` outside
that set returns empty with a note saying the client is **not on your roster**, which is not the
same as the client not existing; check the id with `find kind=client`.

**Unknown parameters are refused, not ignored.** A key that is not in the table above (for example
`team: true`) fails the call with an error naming the key and listing the valid parameters, so a
report can never quietly answer a different question than the one you asked.

`report` is a different job from `find kind=report`: `find`/`get kind=report` read the coach's
saved **progress-report** documents (one per check-in); `report` computes a fresh **aggregate**
over raw history on demand — it does not read or write any saved report row.

**Pick the subject to match the question.** `subject=roster` gives you one thin row per client, so
use it first to see who needs attention across the whole book: for `kind=training`, who is
progressing, flat, regressing or an anomaly; for `kind=checkin`, cadence, the headline measurement
deltas and flags like `no_checkins_in_window`, `overdue_checkin` and `never_checked_in`. Once you
know who, switch to `subject=client` with `clientId` set for the detail: per-exercise progression
verdicts, volume load and the prescribed audit on `kind=training`, or the entries, grouped answers
and intake on `kind=checkin`. Do not fetch every client's full detail to answer a roster-level
question; that is what `subject=roster` is for. A `checkin` roster deliberately carries no answers,
entries or messages, because a 40-client roster would otherwise return thousands of
question-answer pairs.

Report kinds:

| kind | legacy tool | what it aggregates |
|---|---|---|
| `training` | `report_training` | Session counts and volume load, a prescribed-progression audit, and per-exercise progression verdicts (`progressing` / `flat` / `regressing` / `data_anomaly`). |
| `checkin` | `report_checkin` | The client's check-in history as structured data: entries, measurement series against their first entry on record, questions grouped into dated answer series, cadence, and (on `subject=client`) intake, profile, recent messages and the coach's prior reports. **No verdicts.** Nothing scores the answers, because they are free text a coach wrote, often not in English. Read them and draw the conclusion out loud yourself. |
| `body` | `report_body` | Device and lab health metrics: one series per metric with `first`/`last`/`delta`/`min`/`max`/`mean`, the canonical `unit` and `category`, the contributing `sources`, and an `optimalRange`. **No verdicts, no clinical assessment.** |
| `nutrition` | `report_nutrition` | What the client logged eating: a daily series, means over the days they logged, a macro split, and how many of the window's days carry a log at all. **No adherence.** |
| `engagement` | `report_engagement` | In-app messages by direction, contact recency, longest silence, active conversations, and appointments **booked**, with reminder records counted separately. **No attendance.** |
| `business` | `report_business` | Purchases with status and expiry, active/expired counts, next expiry, and paid-invoice totals for the window and for all time. All money in **cents**, keyed **by currency**. **No recurring revenue.** |
| `habits` | `report_habits` | Per habit, built-in and custom alike: `daysLogged`, `daysCompleted`, `completionPct` over **logged** days, the mean of any values entered with its `unit`, and `lastLoggedOn`. The roster gives one rate per client across all habits. Default window 30 days. **No targets**, so no expected-days denominator. |

The kinds differ in kind, not just in subject matter. `kind=training` hands you conclusions and
echoes the `thresholds` they were computed with. Every other kind hands you data and computes
nothing about it, so none of them carries `thresholds` and all report a null `previousWindow`. Do
not paraphrase a check-in answer into a verdict and attribute it to the client.

#### What these kinds refuse to tell you, and why

Four of them are missing a headline you would reasonably expect, because the data cannot support
it. Each says so in its own `notes` on every response. Relay the refusal if the coach asks for the
missing thing; do not fill the gap by inferring it.

| You may be asked for | What you actually get | Why |
|---|---|---|
| Nutrition adherence, "is she hitting her macros" | Intake and logging consistency only | No target exists anywhere: no log carries a template link, and the client nutrition profile holds only free-text preferences. There is nothing to compare against. |
| Attendance, no-shows, "did he turn up" | Appointments **booked** | Appointment status is never transitioned in this system, so almost every past appointment still reads `SCHEDULED`. A computed attendance rate would say nobody ever attends. |
| MRR, churn, renewal rate | Purchases, statuses and expiry dates | No subscription linkage exists on any purchase, and most do not record whether they recur. |
| "Is this client healthy", ranges | Metric series plus an `optimalRange` band | `optimalRange` is the platform's own display band, the same one the coach sees in the app. It is reference data, not a diagnosis, and turning it into one is a claims violation, not just a stretch: see "Claims and intended purpose" in `guardrails.md`. `weight`'s band in particular is computed from the client's **own** observed range, so inside it means their weight has been stable, not that it is healthy. |

Two more traps specific to these kinds:

- **`kind=body`: a metric with `backfilledOnly: true` is history, not tracking.** Most stored health
  data arrived through a one-off bulk import rather than a live device sync. Check `sources` and
  `backfilledOnly` before saying a client "has been tracking" anything.
- **`kind=nutrition`: a day with no log is a day nothing was LOGGED, not a day nothing was eaten.**
  `daysLogged` and `daysInWindow` are both reported so you can see the difference, and the means are
  over logged days only.
- **`kind=engagement` sees in-app messages only.** Coaches also use WhatsApp, so silence here is not
  evidence of no contact, and response times are deliberately not computed.
- **`kind=business` money is in cents and keyed by currency.** Divide by 100 before saying an amount
  to a person, and never add figures across currencies.
- **`kind=habits`: `completionPct` is over the days a habit was LOGGED.** A day with no log is a day
  nothing was logged, not a missed habit, so read `daysLogged` against `daysInWindow` before saying
  anything about consistency. `derived: true` (SESSION_COMPLETED) is completed by Protocol from a
  logged workout, not ticked by the client. Habits are wellness routines: describe consistency,
  never adherence to a treatment.

**Read `coverage`, `notes`, and (when present) `limits` before narrating anything.** Every response
carries a `coverage` block (e.g. `sessionsLogged`/`exercisesReported`/`degradedSections` for a `training` client report,
`entriesInWindow`/`entriesReturned`/`entriesTotal`/`intakePresent`/`degradedSections` for a `checkin`
client report, `clientsRequested`/`clientsReturned`/`clientsWithData` for any roster) so you can
say exactly what the numbers are built from, not just what they say. `notes` is plain-language
context worth relaying verbatim or near-verbatim: how many roster clients logged nothing this
window, that a client has too few sessions for a real verdict, what period the prescribed audit
actually covers, or that a section failed to load and its emptiness means nothing about the client
(on `kind=training`, the prescribed-programs comparison can fail on its own; the sessions and
verdicts still come back, and the note says the `prescribed` blocks are defaults, not findings).
`limits` only appears when something was capped or collapsed (a roster page truncated to the hard
cap, a verdict collapsed for too few sessions, an entries page shorter than the window holds), so
treat its presence as "this is not the whole picture," the same way `hasMore` works for `find`. An
unknown `kind` is rejected with the valid kinds named in the error; do not guess a kind that is not
in the table above.

Two coverage keys are assertions of absence rather than counts, and they are on every response of
their kind: `attendanceTracked: false` on `kind=engagement` and `recurringRevenueTracked: false` on
`kind=business`. They are there so the gap is a stated fact you can relay, not a silence you fill in.

---

### Paging, and knowing what you did not read

Every `find` carries `limit` (default 20-25, max 50-100 by kind) and, on the paged kinds, `offset`.
A paged response tells you where you stand:

```
{ kind, count, total, hasMore, nextOffset, items: [...] }
```

**Loop while `nextOffset` comes back**, passing it as the next `offset`. `total` is the real count,
so "86 check-ins, I read all 86" is a statement you can make honestly. Paged today: `progress`,
`task`, `appointment`, `workout_session`, `client_media` (files in a folder), plus the kinds that
already had it (`client`, `form`, `automation`, `media`, `report`, `submission`, `automation_run`).
`habit`, `product` and `recent_exercise` come back whole, with a `total`.

**`client_history` pages by cursor instead.** The response carries `hasMore` and `nextCursor`; pass
`nextCursor` back as `cursor` and keep going while it comes back. There is no `offset` for it.

On kinds without paging there is no `total`, and the response falls back to a warning instead:
`defaultLimitApplied` (you set no limit, so a default cap applied) or `truncated` (you got exactly
the number you asked for, so there are probably more). Both mean *you are not looking at
everything* — raise the limit, or say what you covered.

Never describe a trend, a count, or "all of X" from a response carrying `hasMore: true`,
`truncated`, or `defaultLimitApplied`.

### `find` / `get` kind table

34 `find` kinds; `get` covers a 19-kind subset. The 15 list-only kinds have **no by-id fetch**.

| kind | `find` | `get` | id param used internally |
|---|---|---|---|
| `client` | ✓ | ✓ | `clientId` |
| `program` | ✓ | ✓ | `programId` |
| `workout` | ✓ | ✓ | `workoutId` |
| `nutrition` | ✓ | ✓ | `templateId` |
| `exercise` | ✓ | ✓ | `exerciseId` |
| `food` | ✓ | ✓ | `foodId` |
| `appointment` | ✓ | ✓ | `appointmentId` |
| `form` | ✓ | ✓ | `formId` |
| `task` | ✓ | ✓ | `taskId` |
| `board` | ✓ | ✓ | `boardId` |
| `automation` | ✓ | ✓ | `automationId` |
| `progress` | ✓ | ✓ | `progressEntryId` |
| `purchase` | ✓ | ✓ | `purchaseId` |
| `media` | ✓ | ✓ | `mediaId` |
| `report` | ✓ | ✓ | `reportId` |
| `submission` | ✓ | ✓ | `submissionId` |
| `transcript` | ✓ | ✓ | `transcriptId` |
| `support_ticket` | ✓ | ✓ | `ticketId` — see `manage_support` in `surface-operations.md` |
| `article` | ✓ | ✓ | `articleId` — `get` returns the body as **Markdown**; `find` reads `status` (DRAFT/PUBLISHED), `scope` (mine/team), `query`, `category`. See `manage_content` in `surface-operations.md` |
| `audience` | ✓ | — | list-only — the saved client segments articles are aimed at, with conditions and a live `memberCount`; `query` filters by name |
| `conversation` | ✓ | — | list-only |
| `lifecycle_stage` | ✓ | — | list-only |
| `lab` | ✓ | — | list-only |
| `health_metric` | ✓ | — | list-only |
| `automation_run` | ✓ | — | list-only; requires `automationId` |
| `automation_kind` | ✓ | — | list-only; the kind catalog, takes no params |
| `task_label` | ✓ | — | list-only; the task label vocabulary |
| `client_label` | ✓ | — | list-only; the client label vocabulary, separate from `task_label` |
| `client_history` | ✓ | — | list-only, cursor-paged; the client history timeline: stage, label, reminder-assignee, program, check-in, form, purchase, invoice and email events, who did each, newest first |
| `workout_session` | ✓ | — | list-only; requires `clientId`; what the client actually did per session, sets positional with `includeSets` |
| `client_media` | ✓ | — | list-only; requires `clientId`; files the client sent (check-ins, forms, chat, nutrition logs, profile photo) as folders |
| `habit` | ✓ | — | list-only; what a `HABIT_TRACKING` form can offer: the standard habits (`mapTo`) and the coach's own (`customHabitId`) |
| `recent_exercise` | ✓ | — | list-only; the exercises this coach used most recently in their own workouts |
| `product` | ✓ | — | list-only; the shop's products (price in minor units, access duration, programs granted) — what `manage_shop action=create_purchase` sells |

Remember: on `get` you always pass the id as **`id`**, never as `clientId`/`programId`/etc. The
right-hand column is what happens internally, not what you send.

---

### The replace grammar (`phases`, `items`, `groups`)

Three verbs take a list that **replaces** an existing tree: `build_program.phases`,
`build_nutrition.items`, `build_workout.groups`. They share one grammar, and getting it wrong used
to delete the operator's work silently.

**Every row needs exactly one control key:**

| Key | Means |
|---|---|
| `add` | Create this row. **`build_program`: `add: true` (boolean).** `build_nutrition`: `add: "MEAL"` / `"LABEL"` / `"GROUP"` / `"SUPPLEMENT"` (the row type). |
| `ref` | Keep this existing row, and its children, exactly as they are. Takes the row's id. |
| `modify` | Patch this existing row. Takes the row's id. |

A row with none of the three is **rejected**. The list you send becomes the entire tree, so:

> **Editing something that already exists? `get` it first, and send `{ ref: <id> }` for every row
> you are keeping.** Omitting a row deletes it.

If every row is rejected, the write is now refused and nothing is saved — the server returns an
error naming the missing control key, and the existing content survives. Read the error and resend
with control keys; do not create a second record.

**Deleting existing rows needs `confirmDelete: true`.** A well-formed list that leaves out a week, a
group, an exercise or a meal row deletes it, so all three verbs now **refuse** that write and list
what would go (`weeksToDelete` / `rowsToDelete`). Show the coach; only on their yes, repeat the call
with `confirmDelete: true`. Agent edits also leave a version in the entity's history, so a coach can
restore one from the dashboard.

A *partial* failure still applies, and the response carries `failCount` and `entries`. Check it: a
plan that came back with `failCount: 2` is missing two rows you thought you wrote.

---

### `report_to_developers`

Required: `summary`.

| Param | Type | Notes |
|---|---|---|
| `summary` | string | **Required.** What you could not do, and why. |
| `goal` | string | What the coach was trying to achieve. |
| `toolOrArea` | string | The verb or product area involved. |
| `error` | string | Exact error text, if any. |

Emails a fixed internal inbox — never a client. Only call it after the coach agrees.

**Now also files a tracked support ticket** (`source: AGENT`), not just an email — the escalation
is a real row on Protocol's support board that a human (or an agent, via `manage_support`) can
comment on and move through `OPEN` → `IN_PROGRESS` → `DONE`. `toolOrArea` becomes the ticket's
`area` when it happens to match Protocol's closed area vocabulary (clients / programs / nutrition /
chat / billing / mobile / other); otherwise the ticket is filed with a null area rather than
rejected. List/read what you filed with `find`/`get` `kind=support_ticket`; add a follow-up
comment or (Protocol only) change its status with `manage_support` — see `surface-operations.md`.

**The ticket write and the email are independent — either can fail without the other.** The
response carries `ticketFiled` (boolean), `ticketId` (null if the ticket write failed), and
`emailSent` (boolean) so you know exactly which channel(s) actually succeeded; `message` is
human-readable prose already worded to match those three fields, but if you are relaying anything
more specific than `message` verbatim, check the booleans rather than assuming both channels went
through — a coach whose ticket write failed still gets a truthful "the ticket did not get filed"
in `message`, not a false "filed" claim.

---

### `guide`

Optional: `topic`.

| Param | Type | Notes |
|---|---|---|
| `topic` | string | A playbook (`protocol-build-program`) or a reference file (`protocol-reference/surface-core.md`, the relative link a playbook uses, or the bare file name when it is unique). Omit to list every topic. |

Serves the same playbooks and reference files a plugin install bundles, so a connection made
without the plugin (a custom connector, a custom MCP server in ChatGPT) can still read them. With
no `topic` it returns `playbooks` (name + when to use it) and `references`; with one it returns
that file as `markdown`. Read-only. Call it before multi-step work you have no playbook loaded for.

---

## Send tier

`schedule`, `manage_automations` and `manage_shop` are the three verbs with an action gated to this
tier; their full parameter and action detail lives in `surface-operations.md`, not here. `send` is
opt-in: the coach must specifically choose it at consent, and the default is `write`, so do not
assume this connection has it. See `guardrails.md` for exactly which four action calls on these
verbs need `send`, and what to do if a call to one of them is refused.
