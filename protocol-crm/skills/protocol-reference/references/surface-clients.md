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

### `message` — the inbox, and drafts for the coach to send

No required params. 3 actions; the verb lists at `read`, and `action=draft` needs `write`.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | `list` · `read` · `draft`. Omit it to list, or to read when `conversationId` is given. |
| `conversationId` | string | `read` / `draft`: the conversation. |
| `clientId` | string | `list`: only conversations with this client. |
| `unread` | boolean | `list`: only conversations with unread messages. |
| `hasDraft` | boolean | `list`: only conversations holding an unsent draft. |
| `labelNames` | string[] | `list`: clients carrying **any** of these client labels, by name (`find kind=client_label`). A name that matches no label returns nobody. |
| `lifecycleStageId` | string | `list`: clients at this pipeline stage. |
| `reminderAssigneeId` | string | `list`: clients with a live reminder assigned to this coach (a user id). |
| `content` | string | `draft`: the message text, as the coach would send it. |
| `overwrite` | boolean | `draft`: replace an existing, different draft. |
| `limit` | number | Result cap. |

`list` rows carry the last-message preview, the coach's own `draft` preview if one is open, and
`client` (the client's stage and labels - coach-internal, never shown to the client).

**`action=draft` saves the text into that conversation's composer and sends nothing.** The coach
sees "Draft: ..." in their inbox and sends it, edits it, or deletes it. It never reaches the client.
A conversation holds one draft for the coach's messaging identity (shared by the owner and admins),
so if one is already there with different text the call is refused and returns `existingDraft`:
fold it into yours, or ask the coach, before you pass `overwrite: true`. An empty `content` is
refused rather than clearing the draft. The claims rule binds a draft exactly as it binds anything
else you write (`guardrails.md`).

**There is still no send path on this verb, or anywhere on the surface.** Sending a client message
is the coach's act (see `guardrails.md`).

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
| `healthProfile` | object | Partial patch. |
| `fitnessProfile` | object | Partial patch. |
| `nutritionProfile` | object | Partial patch. |
| `behavioralProfile` | object | Partial patch. |
| `lifecycleStage` | object | Manages the tenant's **stage list itself**: `{ action: "create"\|"update"\|"reorder", ... }`. |
| `addLabels` | string[] | Label **names** to attach. A name that does not exist yet is created, so there is no need to look one up first. |
| `removeLabels` | string[] | Label **names** to detach. Unknown names are ignored. |

`lifecycleStageId` moves one client. `lifecycleStage` edits the pipeline columns for the whole
tenant. They are not the same thing.

**Labels are addressed by name here, never by id** — unlike almost everything else on this surface.
`addLabels` creates on demand, which is convenient and also means a typo silently becomes a new
label rather than failing. Enumerate the existing vocabulary with `find kind=client_label` before
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
| `clientFacingSummary` | string | report | |
| `priority` | string | report | |
| `sections` | object | report | |
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
| `update` | `reportId` + `clientFacingSummary` / `internalNotes` / `priority` / `sections` | Edits a DRAFT. |
| `approve` | `reportId` (+ `notifyClient`) | Publishes one DRAFT to the client's app **at once**. Refused without a client-facing summary. |
| `approve_many` | `reportIds` (+ `notifyClient`) | Approves a batch. Each report is approved or **skipped with its reason** (already sent, no summary, not found); the response lists `approved` and `skipped`. Nothing approved at all is an error. |
| `discard` | `reportId` | Discards a DRAFT. |
| `unsend` | `reportId` | Takes a sent (APPROVED) report back to DRAFT: it leaves the client's app at once, you correct it with `update`, and `approve` sends it again. The client is never pushed twice for the same report. |

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
