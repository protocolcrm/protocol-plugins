# Protocol MCP surface - clients and their progress

The client record, their check-ins and reports, and the forms that feed both. Assumes
`surface-core.md`.

> **One of four.** The surface is split by the job you are doing, so you read the part you need
> rather than all 23 verbs:
>
> | File | Verbs |
> |---|---|
> | `surface-core.md` | `find` · `get` · `report` · the kind table · the replace grammar · `report_to_developers` |
> | `surface-programming.md` | `build_program` · `build_workout` · `build_nutrition` · `assign_program` · `manage_library` |
> | `surface-clients.md` | `manage_client` · `record_progress` · `manage_forms` · `review_client` · `message` |
> | `surface-operations.md` | `manage_tasks` · `manage_media` · `manage_content` · `schedule` · `manage_automations` · `review_inbox` · `manage_support` · `manage_shop` |


---

### `review_client` — the client bundle

Required: `clientId`.

| Param | Type | Notes |
|---|---|---|
| `clientId` | string | **Required.** |

Returns one object: `client`, `profiles`, `programs`, `nutrition`, `recentProgress`,
`upcomingAppointments`, `openTasks`, `insights`, `trackingSummary` and `recentHistory`. Each section
is null-safe — a section that fails comes back `null` rather than failing the whole call. Prefer
this over ten separate `find` calls.

- **`client.client`** carries `lifecycleStage` as `{ id, name }` and the client's current `labels`
  (`[{ id, name }]`), so neither needs a second lookup.
- **`programs`** rows carry `startDate`, `endDate`, `currentWeek` and `totalWeeks`.
- **`nutrition`** is the plans **assigned to this client** (their own copies), never the tenant
  library. An empty list means nothing is assigned, not that the read failed.
- **`upcomingAppointments`** is the next five from now, soonest first, up to 60 days ahead, the
  client's check-in reminders (`type: CLIENT_REMINDER`) included. For more, or further out, use
  `find kind=appointment clientId=… from=<now> order=asc` (and `to`).

- **`trackingSummary`** is the client record's tracking rail: `checkins`, `workouts`,
  `nutritionLogs`, `labs` and `formSubmissions` over the last **90 days**, `biometricsDaysSince`
  (days since the latest wearable/health reading, `null` = never), and `habitCompletionPct` over
  the last **30 days** (`null`, not 0, when nothing was logged). The windows ride along in
  `windows`; say them when you quote a number, or "38 workouts" reads as all-time.
- **`recentHistory`** is the latest 10 events of the client history timeline, newest
  first, as `{ items, hasMore, nextCursor }`: stage changes, labels
  added and removed, reminder-assignee changes, programs assigned or changed, check-ins, form
  submissions, purchases, invoices paid, emails sent or failed - each with who did it (`channel`
  `session` / `apiKey` / `agent` / `system`). For more, `find kind=client_history`.

#### Reading the client record

Three `find` kinds read what the dashboard's client record shows, each needing `clientId`:

| kind | Answers | Notes |
|---|---|---|
| `client_history` | "What changed, and who did it?" | Filter with `types`, `from`/`to`; cursor-paged (`cursor` = the previous `nextCursor`). An event's `data` carries the names at the time (stage name, label name), so it reads right after a rename. |
| `workout_session` | "What did they actually do?" | One row per logged session: status, duration, ratings, notes. `includeSets: true` adds every exercise and set (below). |
| `client_media` | "What have they sent me?" | Without `folder`: the folders (check-ins, forms, chat, nutrition, profile) with counts. With `folder`: that folder's files, newest first, each with where it came from (`origin`). |

**What the client actually did (`workout_session` with `includeSets`).** Sets are **positional**.
Each exercise carries `units` (in practice almost always `["kg","reps"]`), and each set carries
`values` and `results` **by the same index**: `values: [60, 10]` under `units: ["kg","reps"]` is
60 kg for 10 reps. There is no `set.kg` or `set.reps`. `values` holds the number in each column
(`null` when nothing numeric was logged); `results` is what the client typed; `rangeHints` is the
prescription shown under an empty box. `completed: false` means the set was opened and never
confirmed - not done. `confirmedVia` says how it was confirmed (`tap`, `edit`, `finish_prompt`,
`inferred`, `backfill`); `null` means unknown, which is every set logged before September 2026, not
"the client tapped it". `rpe` is per set when the client logged it. RIR is not recorded per set:
`rirHint` on the exercise is the coach's prescription. `setCounts` gives confirmed vs total.

---

### `message` — the inbox, drafts, and sending as the coach

No required params. 6 actions; the verb lists at `read`, `action=draft` and `action=clear_draft`
need `write`, and `action=send` and `action=poll` need `send`.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | `list` · `read` · `draft` · `clear_draft` · `send` · `poll`. Omit it to list, or to read when `conversationId` is given. |
| `conversationId` | string | `read` / `draft` / `clear_draft` / `send` / `poll`: the conversation. |
| `clientId` | string | `list`: only conversations with this client. |
| `unread` | boolean | `list`: only conversations with unread messages. |
| `hasDraft` | boolean | `list`: only conversations holding an unsent draft. |
| `labelNames` | string[] | `list`: 1:1 conversations whose client carries **any** of these client labels, by name, ignoring case, accents and spacing (`ceka odgovor` matches `Čeka odgovor`; `find kind=client_label` lists them). A name that is no label is an **error** naming it, with close spellings in `unknownLabels[].didYouMean` (`cekaa odgovor` suggests `Čeka odgovor`). A label that exists but nobody carries returns an empty list. |
| `lifecycleStageId` | string | `list`: clients at this pipeline stage. |
| `reminderAssigneeId` | string | `list`: clients with a live reminder assigned to this coach (a user id). |
| `activeSince` | string | `list`: only conversations whose last activity is strictly after this ISO 8601 date or date-time (`2026-10-20`, `2026-10-20T08:00:00+02:00`). An unreadable value is an error, never "no filter". |
| `offset` | number | `list`: rows to skip; pass the previous page's `nextOffset`. |
| `page` | number | `read`: which page of messages, counted back from the newest: `1` (default) is the newest `limit`, `2` the ones before them. Pass the previous read's `nextPage`. |
| `content` | string | `draft` / `send`: the message text, as the coach would send it. |
| `overwrite` | boolean | `draft`: replace an existing, different draft. |
| `question` | string | `poll`: the question, 1-300 characters. |
| `options` | string[] | `poll`: 2-12 answers, each 1-100 characters, unique ignoring case. |
| `allowMultiple` | boolean | `poll`: let each person pick more than one answer. Default `false`. |
| `closesAt` | string | `poll`: ISO 8601 time the poll closes, with an explicit offset or Z (`2026-10-20T18:00:00+02:00`, `2026-10-20T16:00:00Z`), 1 minute to 365 days ahead. A time with no offset, or a non-string, is refused. Omit to leave it open until the coach closes it. |
| `limit` | number | `list`: conversations per page (default 25, **max 100**; a larger ask returns `limitCappedTo: 100`). `read`: messages per page (default 25, max 100). |

`list` returns `conversations`, `total` (every match, not just this page), `hasMore`, and
`nextOffset` while there is more. Each row carries:

- `type`: `DIRECT` is the 1:1 with one client; `GROUP` (members post) and `INFO` (announcement
  group, staff post) reach many clients. `isDirect` is `type === "DIRECT"`. Groups have a `title`.
- `participantCount`, and `participantIds` only for conversations of up to 10 people. The
  all-clients broadcast group lists its count, never its members.
- `lastActivityAt`, and `lastMessage` (`preview`, `sentAt`, `unreadCount`, `senderId`,
  `fromClient`: true when a client wrote it, so "did the client write last" is one field; null
  when the sender is unknown).
- `draft` (`preview`, `updatedByName`, `updatedAt`) if one is open. The draft is shared by the
  owner and admins, so `updatedByName` says who last typed it.
- on a 1:1, `client`: `id` and `name`, plus their `lifecycleStage` and `labels` (coach-internal, never shown
  to the client). A group never has one, and the label, stage and assignee filters only ever
  return 1:1s.

To find a client's 1:1, list with `clientId` and take the row with `isDirect: true`.

`read` returns `messages` (one page of `limit`, oldest first; page 1 is the newest), each with `senderId`, `senderName`,
`senderRole` (`CLIENT`, `TRAINER`, `ADMIN`) and `fromClient`, and `authorId` when an admin replied
as the owner; plus `unreadCount` for the coach's inbox and `draft`, the text already waiting in the
composer (`null` if none). Check `draft` before you write one. Reading marks nothing as read.
The page is told by `page`, `total` (every message in the conversation) and `hasMore`; while older
messages remain, `nextPage` is the page to pass for the next ones back, so a long thread is walked
`page=1, 2, 3...` until `hasMore` is false.

**`action=draft` saves the text into that conversation's composer and sends nothing.** The coach
sees "Draft: ..." in their inbox and sends it, edits it, or deletes it. It never reaches the client.
A conversation holds one draft for the coach's messaging identity (shared by the owner and admins),
so if one is already there with different text the call is refused and returns `existingDraft`:
fold it into yours, or ask the coach, before you pass `overwrite: true`. An empty `content` is
refused rather than clearing the draft: **`action=clear_draft`** is how you remove one (a draft you
left on the wrong client, or one the coach asks you to drop). It deletes nothing the client ever
saw, returns `cleared` and the removed text as `removedDraft` so the text can be put back with
`action=draft`, and is a no-op (`cleared: false`) when there was none. Attachments on the draft are
**lost** (`action=draft` takes text only; `removedDraft.attachmentCount` says how many there were),
so clear a draft that has attachments only when the coach agrees. The claims rule binds a draft
exactly as it binds anything else you write (`guardrails.md`).

**`action=send` and `action=poll` reach the client immediately, as the coach's own message.** The
message lands in the conversation and pushes to the client's phone the moment you call it; there is
no undo from this surface. It carries no AI label: to the client it is a message from the coach,
exactly as if they had typed it, so only send text the coach asked for or confirmed, and when the
coach wants to review it first, `draft` instead. Both need a `send` connection (a `write` connection
is refused with a `PermissionDeniedError`), and both are claims-checked before anything is written:
flagged text (the message, or a poll's question or any option) is refused with `flagged`, the
phrases and why, and nothing is sent. Reword and send again; never work around it. A successful call
returns `sent: true`, `messageId` and `message` (`id`, `conversationId`, `sentAt`), plus `poll` (the
live card, with counts) for a poll. A conversation you are not in, or an announcement group you
cannot post to, comes back as an error. Never say a message was sent unless the call returned
`sent: true` (see wall 1 in `guardrails.md`).

---

## Write tier

---

**Listing clients.** `find kind=client` takes `isActive` — and it matters more than it sounds. A
roster holds everyone who ever signed up: on a live account, 168 clients of whom **31 are active**.
Answering "how many clients do you have?" without it is wrong by five times. Also takes
`lifecycleStageId` for a pipeline stage, and a free-text `query` over name and email.

### `manage_client`

No schema-level required params, but you must pass either `clientId` **or** `create`.

| Param | Type | Notes |
|---|---|---|
| `clientId` | string | The client to edit; omit when passing `create`. |
| `create` | object | `{ firstName, lastName, email, phoneNumber?, accessTier?, sendAccessInstructions? }`. Any other key is refused and nothing is created. `sendAccessInstructions: true` emails the client the app link and a login code, so it needs a **send**-tier connection; without it nothing is sent. |
| `accessTier` | string enum | `LEAD` · `LOW_TICKET` · `CLIENT`. On an existing client, moves them between tiers; alongside `create`, the new client's tier. See *Access tiers* below. |
| `lifecycleStageId` | string \| null | Move **this client** to a stage; `null` clears it. |
| `assignTrainerId` | string | Assign this trainer. The response's `assignment` object carries the assignment `id`. |
| `unassignAssignmentId` | string | Remove an assignment **by assignment id** (not trainer id). |
| `healthProfile` | object | Partial patch: `dateOfBirth` (YYYY-MM-DD), `gender`, `heightCm`, `allergies`, `previousInjuries`, `riskAssessment`, `notes`, `coachNotes`, the three `clientDenies*` flags. No weight, body fat or age (see below). |
| `fitnessProfile` | object | Partial patch: `experienceLevel`, `primaryGoals`, `preferredTrainingStyles`, `availableDays`, `maxSessionsPerWeek`, `sessionDurationPreferenceMinutes`, `equipmentAccessLevel`, `notes`, `coachNotes`. |
| `nutritionProfile` | object | Partial patch: `primaryDietType`, `nutritionGoals`, `preferredMealFrequency`, `foodPreferences`, `cuisinePreferences`, `dietaryRestrictions`, `previousDietSuccess`, `dietChallenges`, `nutritionRelatedHealthConcerns`, `notes`, `coachNotes`. |
| `behavioralProfile` | object | Partial patch: `motivationFactors`, `adherenceRate` (1-5), `consistencyScore` (1-5), the `preferredCommunication*` fields, `preferredFeedbackStyles`, `responsesToChallenges`, `behavioralTriggers`, `notes`, `coachNotes`. |
| `lifecycleStage` | object | Manages the tenant's **stage list itself**: `{ action: "create"\|"update"\|"reorder", ... }`. |
| `addLabels` | string[] | Label **names** to attach. A name matching an existing label ignoring case, accents and spacing reuses it (`ceka odgovor` attaches `Čeka odgovor`); any other name is created, so there is no need to look one up first. |
| `removeLabels` | string[] | Label **names** to detach, matched the same way. Unknown names are ignored. |

`lifecycleStageId` moves one client. `lifecycleStage` edits the pipeline columns for the whole
tenant. They are not the same thing.

**Labels are addressed by name here, never by id** — unlike almost everything else on this surface.
**A profile key the profile does not have is refused, and the whole call changes nothing.** The
error names each key and where it belongs. The usual one is a measurement: weight, body fat and
girths are not profile fields, they are a dated check-in, `record_progress action=entry` with
`measurements {"weightKg":84,"bodyFatPercentage":16}`, which is what the charts and reports read.
Age is not stored either; set `healthProfile.dateOfBirth`, and ask the coach for it when you only
know the age. Never fall back to writing these into `notes` or `coachNotes`. The response carries
the client with all four profiles, so you can check what landed. A profile field you leave out is
kept; one you send as `null` is cleared (`[]` also empties a list), so `dietaryRestrictions: null`
removes every restriction. This is the same rule the REST profile PUTs follow, so never send a field
as `null` to mean "unchanged": leave it out. (Medical conditions are not editable through
`manage_client`; on the REST API `medicalConditions: null` removes every condition row.) Each restriction is `{ food, isAllergy, severity, alternatives? }`, and
`severity` is the coach form's whole number **1-5** (1 a mild preference, 5 must never appear in a
plan; the form starts a new one at 3). Any other severity is refused and the whole call writes
nothing.

`addLabels` creates on demand, which is convenient and also means a typo silently becomes a new
label rather than failing (a missing accent does not: label names ignore case, accents and
spacing everywhere, so `Ceka` can never sit beside `Čeka`). Enumerate the existing vocabulary with `find kind=client_label` before
inventing a name, and filter clients by label with `find kind=client labelNames=[…]`.

Unlike the array parameters elsewhere in this reference, `addLabels` / `removeLabels` are **deltas,
not replacements**: they add and remove exactly what you list and leave the client's other labels
alone. Do not read-merge-write them.

#### Access tiers

`get kind=client` shows each client's `accessTier`.

| Tier | What they get | Billed |
|---|---|---|
| `CLIENT` | Full coaching. Moving someone here links you as their trainer and enrolls them in the coach's broadcast groups. | yes |
| `LOW_TICKET` | The reduced app (no coach chat, no AI meal analysis unless the coach switched them back on), out of coach workload: moving someone here ends their trainer link. | **yes**, the same as a CLIENT |
| `LEAD` | Nothing delivered, no stored history. | no |

Moving a client **out of `LEAD`** adds a billable person and is refused when the plan is at its
client cap. Confirm a tier change with the coach before making it: it changes what the client can
open in the app and, from `LEAD`, the bill. Nothing is sent to the client either way. Asking for the
tier a client already has is a no-op. The response carries `accessTier: { accessTier,
previousAccessTier, changed }`.

---

### `record_progress`

Required: `action`. 3 actions; `report` has 5 sub-actions in `reportAction`.

| Param | Type | Action | Notes |
|---|---|---|---|
| `action` | string enum | — | **Required.** `entry` · `report` · `note` |
| `progressEntryId` | string | entry | Update this check-in; omit to create. |
| `clientId` | string | entry, note | |
| `entryDate` | string | entry (create) | `YYYY-MM-DD` |
| `measurements` | object | entry (**create and update**) | The numbers. See *Measurements* below. |
| `userNotes` | string | entry (create) | |
| `trainerNotes` | string | entry | |
| `internalNotes` | string | entry, report | |
| `status` | string | entry (update) | Free-form tenant status name. |
| `labels` | object[] | entry (update) | |
| `reportAction` | string enum | report | `update` · `approve` · `approve_many` · `discard` · `unsend` |
| `reportId` | string | report | Every sub-action but `approve_many`. |
| `reportIds` | string[] | report (`approve_many`) | Up to 50 DRAFT reports. |
| `notifyClient` | boolean | report (`approve`, `approve_many`) | Also push the client. Needs a **send**-tier connection. |
| `clientFacingSummary` | string | report | `update`: refused when the report already holds text a person wrote or edited, unless `overwrite`. |
| `internalNotes` | string | report | `update`: refused when the report already holds a note a person wrote or edited (anything but the AI original), unless `overwrite`. |
| `overwrite` | boolean | report (`update`) | Replace a client-facing text or internal note a person wrote or edited. Default false. |
| `priority` | string | report | |
| `sections` | object | report | Its `client_facing_summary` / `priority` keys always follow those fields; `sections.internal_notes` is the AI original, so read `internalNotes`. A client only ever receives `sections.client_facing_summary`. |
| `title` | string | note | |
| `content` | string | note | |
| `appointmentId` | string | note | |

#### Measurements - the numbers everything else is computed from

Keys, all optional, all **numbers** (a string `"80"` is discarded, not parsed):

`weightKg` · `bodyFatPercentage` · `muscleMassPercentage` · `leanMassPercentage` · `chestCm` ·
`waistCm` · `hipsCm` · `leftArmCm` · `rightArmCm` · `leftThighCm` · `rightThighCm` · `squat1rm` ·
`deadlift1rm` · `benchPress1rm` · `pullupsMax` · `kmTimeSeconds` · `vo2Max` · `sleepAverageHours`

`energy` · `mood` · `adherence` · `sleepQualityScore` are a **1-10 scale, not a percentage** - pass
8, not 80. Out of range is rejected outright.

- **Create** merges by client + date: a second check-in for the same day updates the first rather
  than duplicating. So re-recording a day is safe.
- **Update** (with `progressEntryId`) merges the keys you send over the existing ones - send only
  what you are correcting. Body-composition weights are recomputed from a new `weightKg`.
- Both paths echo the resulting `measurements`. Read it back; that is how you know the correction
  landed rather than assuming it.

`entry` create honors `clientId`/`entryDate`/`measurements`/`userNotes`/`trainerNotes`/
`internalNotes`; `entry` update honors `progressEntryId`/`status`/`trainerNotes`/`internalNotes`/
`labels`. A field belonging to the other path is refused, and nothing is written.

#### Report triage

| `reportAction` | Reads | Does |
|---|---|---|
| `update` | `reportId` + `clientFacingSummary` / `internalNotes` / `priority` / `sections` (+ `overwrite`) | Edits a DRAFT. Will not replace a client-facing text or an internal note a person wrote or edited without `overwrite: true`; the refusal returns `existingClientFacingSummary` or `existingInternalNotes`. |
| `approve` | `reportId` (+ `notifyClient`) | Publishes one DRAFT to the client's app **at once**. Refused without a client-facing summary. |
| `approve_many` | `reportIds` (+ `notifyClient`) | Approves a batch. Each report is approved or **skipped with its reason** (already sent, no summary, not found); the response lists `approved` and `skipped`. Nothing approved at all is an error. |
| `discard` | `reportId` | Discards a DRAFT. |
| `unsend` | `reportId` | Takes a sent (APPROVED) report back to DRAFT: it leaves the client's app at once, you correct it with `update`, and `approve` sends it again. The client is never pushed twice for the same report. |

**A person's wording is not overwritten silently.** `get kind=report` carries
`clientSummaryEdited` on an AI-drafted report: `false` while the client-facing text is still the
automation's draft, `true` once someone (the coach, or an agent) has rewritten it; it is absent on a
hand-written report. `update` replaces an untouched draft or an empty summary freely, and refuses
any other different text unless `overwrite: true`. Fold the coach's wording in, or ask, before you
pass it. `internalNotes` follows the same rule: the automation's original note (still in
`sections.internal_notes`) or an empty one is replaced freely, a note the coach wrote or edited
needs `overwrite: true`. `priority` edits never need it. Edits carry no author, so a text **you**
rewrote earlier in the session also reads as edited: check it is yours, then pass `overwrite: true`.

Approving is **silent by default**: the report appears in the client's app with no notification.
`notifyClient: true` also sends a "new progress report" push, once per report, and is held at the
**send** tier because it reaches the client's phone. Either way, approve only what the coach said to
send (`guardrails.md`).

---

### `manage_forms`

Required: `action`. 2 actions.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `create` · `update` |
| `formId` | string | Required for `update`. |
| `title` | string | |
| `description` | string | |
| `presentationType` | string | `SINGLE_PAGE` · `MULTI_PAGE` · `HABIT_TRACKING` · `PROGRESS_TRACKING` · `INTAKE` — **not** schema-validated; a bad value passes straight through. |
| `questions` | object[] | **Replaces the whole question array.** `get` the form first. See *Question rows* below. |
| `sections` | object[] | `INTAKE` only. Ordered `[{ id, title, role, description? }]`; `role` is one of `ABOUT_YOU` · `GOALS` · `HEALTH` · `BODY_METRICS` · `PHOTOS` · `TRAINING_HISTORY` · `CONSENT` · `NUTRITION` · `AVAILABILITY` · `INJURIES` · `CUSTOM`. Every question of an intake form carries the `sectionId` of its section. |
| `theme` | object | |
| `settings` | object | |

#### Question rows - and `mapTo`, which decides whether a check-in is analysable

**`mapTo` is the whole answers-to-measurements pipeline.** On submission, every answer carrying a
`mapTo` is written into the check-in's `measurements` (`WEIGHT` → `weightKg`, `WAIST` → `waistCm`,
…) - which is what every chart, trend and progress report reads. A question with **no** `mapTo` is
stored as text and is invisible to all of it. Build a weekly check-in without it and the form looks
perfect, collects diligently, and produces nothing you can plot. Set it on every question that
records a number; use `CUSTOM` for a qualitative question, `INFO` for a screen that asks nothing.

Both write paths return the resulting `questions` (id, type, title, mapTo) plus
`unmappedQuestionCount` - check it, it is how you catch this.

Row fields: `id` · `type` · `title` · `description` · `placeholder` · `required` · `purpose` ·
`options[{id,label}]` · `allowMultiple` · `maxRating` · `mapTo` · `customHabitId` · `pinned`.

**Habit forms take habits, and only habits.** Every question on a `HABIT_TRACKING` form must be one
the client can log: a standard habit (`{ mapTo: "STEPS" }`, `SLEEP_HOURS`, `WATER_INTAKE`, ...) or
one of the coach's own custom habits, written as
`{ mapTo: "CUSTOM", customHabitId: "<id>" }`. List both with `find kind=habit`. `INFO`, a `CUSTOM`
row without a `customHabitId`, and a retired custom habit are all refused on save. The habit's name
and unit come from its definition, so a later rename keeps every logged day. Two standard habits
behave differently: `SESSION_COMPLETED` is derived from the workout log and cannot be ticked by
hand, and `BODY_WEIGHT` also lands in the client's health metrics.

`type` is one of `welcome` · `multiple_choice` · `picture_choice` · `yes_no` · `dropdown` ·
`short_text` · `long_text` · `legal` · `rating` · `upload_media` · `end_screen`. **There is no
`number` type** - a numeric field is `short_text` with `purpose: "number"`.

**Three families, two row shapes.** Questionnaires (`SINGLE_PAGE`, `MULTI_PAGE`) take full rows and
get welcome/end screens added automatically. Trackers (`PROGRESS_TRACKING` for weekly measurement
check-ins, `HABIT_TRACKING` for daily habits) take minimal rows - often just `{ mapTo, pinned }`,
no type or title - and get no bookends. The intake questionnaire (`INTAKE`, the onboarding form a
new client is sent) takes full rows grouped by `sections`, gets no bookends (the app steps through
its sections), and its purpose is always `INITIAL_QUESTIONNAIRE` whatever `settings` say. Real
accounts use all of these; read the form you are editing before assuming which.

**Keep each surviving question's `id`.** Submitted answers are stored against it, so replacing the
array with fresh ids orphans every past answer and loses that question's history.

Form reads go through `find` / `get`, never this verb.
