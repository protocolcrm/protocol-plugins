# Protocol MCP surface - running the practice

The calendar, the kanban, the media library, articles, the inbox, automations and the shop's
bookkeeping - the operational half that is not about one client's programming. Assumes
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

### `manage_tasks`

Required: `action`. 16 actions.

`create_task` · `update_task` · `complete_task` · `move_task` · `archive_completed` ·
`create_subtask` · `update_subtask` · `toggle_subtask` · `reorder_subtasks` · `create_board` ·
`update_board` · `create_column` · `update_column` · `reorder_columns` · `create_label` ·
`update_label`

Pass the fields that action needs. The per-action list is published on the `action` enum, and a
field the action does not read is **refused** with that list, not ignored.

| Param | Type | Typically used by |
|---|---|---|
| `taskId` | string | task + subtask actions |
| `subtaskId` | string | update_subtask, toggle_subtask |
| `subtaskIds` | string[] | reorder_subtasks |
| `boardId` | string | board + column actions |
| `columnId` | string | move_task, update_column |
| `columnIds` | string[] | reorder_columns |
| `labelId` | string | update_label |
| `title` | string | tasks / subtasks |
| `name` | string | boards / columns / labels |
| `description` | string | tasks |
| `dueAt` | string | tasks — `YYYY-MM-DD` |
| `recurrenceInterval` | integer | create_task, update_task — repeat every N periods |
| `recurrencePeriod` | string enum | create_task, update_task — `DAY` · `WEEK` · `MONTH` · `YEAR` |
| `isPrivate` | boolean | create_task, update_task — visible only to the coach who created it |
| `clientId` | string | tasks |
| `assigneeId` | string | tasks / subtasks |
| `isDone` | boolean | toggle_subtask |
| `position` | integer | move_task, columns |
| `color` | string | labels / columns / boards |
| `icon` | string | boards |
| `wipLimit` | integer | columns |
| `isDoneColumn` | boolean | columns |
| `isArchived` | boolean | boards |
| `defaultColumns` | string[] | create_board |

Task/board reads go through `find` (`kind=task`, `kind=board`) and `get`.

**Labels.** `create_label` makes one (name + color, both required); `find kind=task_label` lists
them; `labelIds` on create_task/update_task puts them ON a task, replacing whatever was there. Ids
that do not resolve are skipped silently by the service, so the write reports back
`unresolvedLabelIds` — check it. Filter by them with `find kind=task labelIds=[...]`.

**Assignees.** A task can have several: `assigneeIds` replaces the set, `assigneeId` is a
one-owner convenience. Unresolved ids come back as `unresolvedAssigneeIds`. Subtasks take a single
`assigneeId`.

**The filters that answer the usual questions.** `find kind=task` takes `isOverdue` ("what's
late?"), `assigneeId` ("what's on Ana's plate?"), `labelIds`, `dueAfter`/`dueBefore`, `priority`,
`columnId`, `boardId`, `isArchived` and a free-text `searchTerm` that also matches label names.

**`title` names a task or subtask; `name` names a board, column or label.** Send the wrong one and
the call is refused, naming the right one. `create_task` also needs a `boardId` or a `columnId`; a
title alone is refused. `reorder_subtasks` / `reorder_columns` take the COMPLETE id list.

**Recurring tasks.** `recurrenceInterval` + `recurrencePeriod` (8 + `WEEK` = every eight weeks) make
completing the task create the next one, in the first column of its board, due N periods after this
one's due date (or after today when it has none). Send both on create; on update either half may
change alone once the task has a recurrence. Half a recurrence on a task with none is refused, since
it would never fire. The write echoes `recurrenceInterval`, `recurrencePeriod` and `isPrivate`; check
them. This surface cannot clear a recurrence once set; the coach does that in the dashboard.

---

### `manage_media`

Required: `action`. 6 actions.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `attach` · `update` · `create_category` · `update_category` · `share` · `update_share` |
| `mediaId` | string | |
| `categoryId` | string | |
| `shareId` | string | |
| `url` | string | `attach`: the hosted asset URL. There is no raw upload path. |
| `name` | string | Also the free-text filter key on `find kind=media`. |
| `type` | string | `IMAGE` · `VIDEO` · `AUDIO` · `FILE` · `DOCUMENT` · … |
| `thumbnailUrl` | string | |
| `userId` | string | Assign media / scope a category to this client. |
| `categoryIds` | string[] | |
| `color` | string | |
| `iconEmoji` | string | |
| `order` | integer | |
| `parentId` | string | Nested category. |
| `description` | string | |
| `shareType` | string enum | `PUBLIC` · `USER_SPECIFIC` — **create only**; `update_share` cannot re-type a share (recreate it). |
| `sharedWithUserIds` | string[] | |
| `permission` | string enum | `READ` · `WRITE` |
| `expiresAt` | string | ISO datetime. |
| `isActive` | boolean | |

Shares never send an email.

`categoryIds` **replaces** the media's category set. To add one more folder, `get` the media, send
all its existing ids plus the new one - otherwise you quietly unfile it from the rest.

---

### `manage_content`

Required: `action`. 12 actions. Articles are the coach's rich content pieces - a title, category,
cover, and a block body - that clients read in the app's Learn tab and inside programs. You write
and read the body as **Markdown**; the platform stores structured content and renders it itself.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `create_article` · `update_article` · `publish_article` · `unpublish_article` · `attach_to_program` · `duplicate_article` · `set_tags` · `share_article` · `delete_article` · `create_audience` · `update_audience` · `delete_audience` |
| `articleId` | string | Every article action but `create_article`. |
| `audienceId` | string | `update_audience`, `delete_audience`. |
| `addTags` · `removeTags` | string[] | `set_tags`: tag **names**. Unknown names in `addTags` are created. The article vocabulary is separate from client labels. |
| `folder` | string \| null | `set_tags`: a folder by name or id; `null` takes it out of its folder. An unknown folder is refused with the list of folders that exist (the coach creates folders in the dashboard). |
| `revoke` | boolean | `share_article`: `true` takes the public link down; the old url stops working. |
| `expiresAt` | string \| null | `share_article`: ISO date the link stops working; omit for no expiry. |
| `embedEnabled` | boolean | `share_article`: allow the embed snippet on another site (default true). |
| `confirm` | boolean | `delete_article`, `delete_audience`: must be `true` to delete. |
| `title` | string | Required on create. The slug is derived from it once and does not follow later renames. |
| `markdown` | string | The body. On `update_article` it **replaces the whole body** - read with `get kind=article` first. |
| `category` | string | Free text. Reuse what `find kind=article` already shows rather than inventing a near-duplicate. |
| `excerpt` | string | One or two sentences for the card. Falls back to the first 200 characters of the text when empty. |
| `coverMediaId` | string | A media id (image or video). |
| `slug` | string | Optional override; made unique in the tenant. |
| `audience` | object | `{ all: true }` or `{ audiences: [saved audience names or ids] }`; reaching any listed audience is enough. Default on create: everyone. |
| `name` · `description` · `conditions` | | `create_audience` / `update_audience`, see below. |
| `programId` | string | `attach_to_program`. |
| `collectionName` | string | `attach_to_program`: the program content collection to file it under; created if missing, default "Articles". |

Body syntax, beyond ordinary Markdown (headings 1-3, lists, tables, links, bold/italic/strike/code):

| You write | It becomes |
|---|---|
| `![The gym floor](media:<mediaId>)` | An image from the media library, referenced by id and resolved fresh on every read. |
| `![Demo](media:<mediaId>?kind=video)` | A video. Add `&align=wide` on either for full-bleed. |
| `> [!tip]` / `> [!info]` / `> [!warning]` on the first line of a blockquote | A tinted callout; the rest of the quote is its body. |
| `https://www.youtube.com/watch?v=<id>` alone on a line | An embedded YouTube player. |

Anything outside that set degrades to plain text rather than failing; the stored body always fits
the app's renderer.

`create_article` makes a **DRAFT**. Nothing reaches a client until `publish_article`, which first
runs the claims guardrail over the title, excerpt and text. A hit is an error carrying `signals`
(each flagged phrase and the field it sits in) and the article stays a draft - reword in wellness
language and publish again; there is no override. `unpublish_article` hides it again and keeps its
place in the feed for a re-publish.

`attach_to_program` adds an ARTICLE item to the program's content section (idempotent). Being on
the program is its own audience: a client on that program can open the article even when the
audience rules do not match them - once it is published. `build_program content` also accepts
`{ type: 'ARTICLE', articleId }` items when you rebuild a whole content section.

**Audiences are saved and reused.** An audience is a named segment of clients: `create_audience`
with `name` and `conditions`, where conditions **AND** together and each condition matches **any**
of its `values` (names or ids): `[{ type: "lifecycleStages", values: ["Onboarding"] }, { type:
"labels", values: ["At risk"] }]` is "Onboarding + At risk"; two `labels` conditions is "Overweight
+ Churn risk". Enumerate the vocabulary with `find kind=lifecycle_stage` and `find kind=client_label`,
and the saved audiences with `find kind=audience` (each with a live `memberCount`). Then aim an
article with `audience: { audiences: ["Onboarding + At risk"] }`.

The response echoes `audience` in the friendly shape, `audienceAll` and `audienceIds` as stored,
`folderId`, `tags`, `publicShare`, the body as `markdown`, and `unresolvedAudience` when a name did
not match. An audience list that resolves to **nothing** is refused outright.

`update_audience` renames it, edits its description, or replaces its conditions (same grammar as
create); every article aimed at it follows at once.

**Organising the library.** `duplicate_article` copies an article into a new DRAFT of yours (same
body, cover, folder, audience and tags; the title says "(copy)"); nothing is published. `set_tags`
files and tags it. Tags and folders are the coach's organisation and clients never see them.

**The public link.** `share_article` gives a **published** article a public url and an iframe
`embedSnippet` for the coach's own website, returned under `publicShare`. A draft is refused:
publishing is where the claims guardrail runs. Anyone holding the link can read the article -
**the audience does not apply** - so say that to the coach. Nothing is sent to anyone: you hand the
url to the coach and they decide where it goes. `unpublish_article` and `revoke: true` both take the
link down. It is a `write`-tier action for that reason, the same as a public `manage_media` share.

**Deleting is permanent and confirm-guarded.** `delete_article` and `delete_audience` remove the row
with no trash and no undo. Without `confirm: true` they delete **nothing** and return an error
carrying `confirmRequired: true` and `wouldDelete` (the article's title and status and its public
link; or the audience and every article aimed at it, which then reach nobody through it). Show that
to the coach and call again with `confirm: true` only on their explicit go-ahead (wall 3 in
`guardrails.md`). To hide an article, `unpublish_article` it instead: that keeps it.

---

### `review_inbox`

No required param — `action` defaults to `overview`. 5 actions.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | `overview` (default) · `mark_read` · `mark_all_read` · `dismiss_insight` · `mark_insight_read` |
| `notificationId` | string | `mark_read` |
| `insightId` | string | `dismiss_insight`, `mark_insight_read` |
| `insightLimit` | number | `overview`: how many insights (default 25, max 200), highest priority first. |

`overview` returns `{ dashboard, notifications, unreadCount, insights, insightsTotal,
insightBreakdown }`, each section null-safe. **`insights` is the top 25, not all of them** - a real
account had 518. `insightsTotal` and `insightBreakdown` (counts by type and severity) describe the
whole set, so describe the pile honestly - "518 active, mostly plateau warnings, here are the 25
that matter" - rather than reporting 25 as if it were everything. Raise `insightLimit` only when
you are genuinely working the whole list; it is the most expensive read on the surface.

---

### `schedule`

Required: `action`. 7 actions.

| Param | Type | Action | Notes |
|---|---|---|---|
| `action` | string enum | — | **Required.** `create` · `update` · `cancel` · `reminder` · `booking_config` · `gcal_disconnect` · `send_reminder` |
| `appointmentId` | string | update, cancel, send_reminder | |
| `title` | string | create, update, reminder | |
| `startTime` | string | create, update, reminder | ISO-8601 instant. |
| `endTime` | string | create, update, reminder | ISO-8601 instant. On `reminder` it is the end of the first response window: the same thing as `responseWindowHours`, so send either; a disagreement is refused. |
| `clientId` | string | create, reminder | |
| `type` | string | create, update | |
| `modality` | string | create, update | |
| `location` | string | create, update | |
| `description` | string | create, update, reminder | |
| `status` | string | update | |
| `notifyParticipants` | boolean | create, update, cancel | Email every guest an invitation, update or cancellation with a calendar file. Needs a **send**-tier connection. |
| `formId` | string | reminder | The check-in form the reminder asks for. |
| `recurrenceRule` | string | reminder | |
| `responseWindowHours` | number | reminder | |
| `reminderHoursBefore` | number | reminder | |
| `reminderType` | string | reminder | |
| `read` | boolean | booking_config | `true` reads the config instead of writing it. |
| `trainerId` | string | booking_config | |
| `bookingUrlSlug` | string | booking_config | |
| `sharedAvailabilities` | object[] | booking_config | `{dayOfWeek 1-7, startTime "HH:MM:SS", endTime, timezoneId, isActive}`. Full replace — **omit to preserve**. |
| `globalSettings` | object | booking_config | |
| `eventConfigurations` | object[] | booking_config | |

`booking_config` write: pass `sharedAvailabilities`, `globalSettings`, **and**
`eventConfigurations` together — the write path expects all three even though the verb's own schema
does not mark them required. `globalSettings.maximumAdvanceDays` is restricted to
`7` · `14` · `30` · `60` · `90` to stay in lock-step with the coach dashboard's own preset picker.

`send_reminder` is outward — it reaches the client. Confirm before firing it.

#### Recurring check-ins, and the booking page

`action: "reminder"` is how a weekly check-in gets scheduled: `clientId` + `formId` + `startTime`
(the FIRST occurrence) + `recurrenceRule`, an iCal RRULE - `FREQ=WEEKLY`, or
`FREQ=WEEKLY;INTERVAL=2` for fortnightly. The weekday and the time both come from
`startTime`, so `BYDAY` is not used and is ignored if sent. Defaults: `responseWindowHours` 48,
`reminderHoursBefore` 0 (fires at the occurrence), `reminderType` PROGRESS_CHECK_IN.

**This call is outward, not configuration.** Despite reading like a scheduling setting, `reminder`
arms a real client-facing push, and it fires within minutes if `startTime` is now or in the past
(a `startTime` in the future just delays it to that time, it does not make the call itself safe to
skip approval for). Confirm with the coach what will be sent and to whom before calling it, the
same as `send_reminder` above.

`action: "booking_config"` writes the coach's **public** booking page. **Read it first**
(`read: true`). `globalSettings` is merged, so send only the keys you are changing;
`sharedAvailabilities` and `eventConfigurations` are whole-array replaces, so **omit them unless
you mean to rewrite them** - a real coach has ~66 availability slots and a short list deletes the
rest. `maximumAdvanceDays` must be one of 7 / 14 / 30 / 60 / 90.

`modality` is closed: `IN_PERSON` · `VIRTUAL` · `HYBRID` · `ASYNCHRONOUS`.

**Two actions on this verb are outward, not one.** `send_reminder` fires a reminder now.
`reminder` is not the plain write it looks like either: despite the name, it arms a recurring
client push that fires within minutes if `startTime` is now or in the past, so treat it exactly
like `send_reminder`, confirm with the coach before calling it. `create`, `update`, `cancel`,
`booking_config`, and `gcal_disconnect` are internal writes that reach nobody - **unless**
`create` / `update` / `cancel` carry `notifyParticipants: true`.

#### Telling the guests

`notifyParticipants: true` on `create`, `update` or `cancel` emails every guest with an email
address - the client included, never the coach - an invitation, an update or a cancellation, each
with a calendar file keyed on the appointment, so the guest's own calendar adds, moves or removes
the same event. On `update`, a guest hears only when something they see changed (time, title,
location, link). Off by default: without it nobody is told. Because it reaches people outside the
account, a call carrying it is held at the **send** tier: at `write` it is refused naming
`notifyParticipants`, while the same call without the flag books, moves or cancels fine. Confirm
with the coach before sending it. A client reminder never emails, whatever the flag says.

---

### `manage_automations`

Required: `action`. 6 actions.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `create` · `update` · `activate` · `pause` · `archive` · `run` |
| `automationId` | string | Required for everything except `create`. |
| `name` | string | `create` |
| `kind` | string | `create` — enumerate valid kinds via `find kind=automation_kind`. |
| `config` | object | Kind-specific config. |
| `triggerConfig` | object | Kind-specific trigger config. |
| `triggerData` | object | `run`: kind-specific trigger payload. |
| `schedule` | object \| null | `create`, `update`: run on a clock. See *Scheduled automations*. |

`create` lands the automation in DRAFT. `run` **dispatches an execution now**: outward. Read a
run's outcome with `find kind=automation_run` + `automationId`.
There are three kinds of automation. Two can be created here: `PROGRESS_REPORT` (triggers
`PROGRESS_ENTRY_CREATED` and `MANUAL`) and `FORM_REPORT` (triggers `FORM_SUBMITTED` and `MANUAL`).
The third, **CUSTOM** (`CUSTOM_WORKFLOW` / `CUSTOM_AI_STEP`), is a workflow Protocol built in n8n for
one coach; it carries `readOnly: true`. You can read it and its runs with `find` / `get`, and every
write to it (`update`, `activate`, `pause`, `archive`, `run`) is refused, as is creating one; changes
go through Protocol (`report_to_developers`). Confirm with `find kind=automation_kind` before
assuming another kind exists. `run` needs `triggerData` - for
PROGRESS_REPORT that is `{ entryId: "<progress entry uuid>" }`, for FORM_REPORT that is
`{ submissionId: "<form submission uuid>" }`. **`run` is the
only outward action here, and it is the one held to the `send` tier** (see `guardrails.md`): if
this connection is at `write` (the default), calling `run` is refused with a
`PermissionDeniedError` naming the tier needed - that is expected, not a bug, so tell the coach
plainly and stop rather than escalating or falling back to `protocol-rest-escape`. If the
connection is at `send`, confirm with the coach before calling it anyway: getting the payload
wrong still dispatches a real, client-facing execution, so it costs more than a wasted call.
Authoring (create/update/activate/pause/archive) is a plain write, at `write` tier. Read what an
execution actually did with `find kind=automation_run`.

#### Scheduled automations

`schedule: { startAt, timezone, rrule }` makes an automation run on a clock instead of an event.
`startAt` is a **wall clock** in `timezone`, written `2026-11-02T09:00` with no `Z` and no offset
(an instant is refused, because the two read the same and mean different things an hour apart).
`timezone` is an IANA zone (`Europe/Belgrade`); `rrule` is an iCal rule (`FREQ=WEEKLY;BYDAY=MO`), or
`null` for a one-off. It is stored as `triggerConfig.schedule`; sending `schedule` alone on `update`
keeps the rest of the stored `triggerConfig` (sending `triggerConfig` replaces it whole). `null`
removes the schedule. A bad zone or rule is refused with the field named. The response echoes
`schedule`.

---

### `manage_support`

Required: `action`. 2 actions.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `comment` · `set_status` |
| `ticketId` | string | Required by both actions. |
| `body` | string | `comment`: the comment text. |
| `status` | string | `set_status`: `OPEN` · `IN_PROGRESS` · `DONE`. |
| `priority` | string | `set_status`: `LOW` · `NORMAL` · `HIGH` · `URGENT` — Protocol team only, same as `status`. |

This works a support ticket that **already exists** — list and read them with `find` / `get`
`kind=support_ticket`. Filing a NEW ticket is not an action here: use `report_to_developers` (see
`surface-core.md`), which now files a tracked ticket (`source: AGENT`) in addition to its email.

`comment` appends to the ticket's thread. `authorSide` (`COACH` vs `PROTOCOL`) is derived from the
caller's own role, never from input, so a coach's comment can never be posted as Protocol.

**`set_status` is Protocol team only.** A coach's agent can read and comment on a ticket it filed,
but calling `set_status` is refused with a permission error reading "Only the Protocol team can
change a ticket status" — it names no tier, because this is not a tier restriction at all: it is
the service enforcing role, not the connection's `write`/`send` tier, so no tier upgrade fixes it
and no coach role can move a ticket's status or priority, ever. Passing neither `status` nor
`priority` is a harmless no-op that leaves the ticket untouched.

`find kind=support_ticket` is scoped by **team permission inside one tenant**: an OWNER or ADMIN of
a tenant sees every ticket filed in that tenant, and every other member sees only the tickets they
personally filed. Nobody but Protocol crosses a tenant boundary. The `status` / `area` / `tenantId` /
`query` filters on `find` only do anything for a Protocol (ADMIN/SYSTEM) caller — a coach's list
ignores them rather than erroring.

---

### `manage_shop`

Required: `action`. 2 actions, **tiered per action**: `record_payment` is `write`,
`create_purchase` is `send`.

| Param | Type | Action | Notes |
|---|---|---|---|
| `action` | string enum | — | **Required.** `create_purchase` · `record_payment` |
| `clientId` | string | create_purchase | **Required.** The client buying. Must be one this coach may reach. |
| `productId` | string | create_purchase | The product sold (`find kind=product`). Price, currency, programs granted and access duration default from it. |
| `name` | string | create_purchase | Required when there is no product (a one-off). |
| `amount` | number | both | **Minor units** (15000 = 150.00). `create_purchase`: defaults to the product price. `record_payment`: defaults to the invoice's whole outstanding balance; less is a partial payment. |
| `currency` | string | create_purchase | e.g. `eur`. Defaults to the product's, then the shop's. |
| `status` | string | create_purchase | `ACTIVE` (default: paid or started) or `PENDING` (awaiting payment). |
| `purchasedAt` / `expiresAt` | string | create_purchase | ISO dates. `expiresAt` defaults to the product's access duration. |
| `paymentSchedule` | string | create_purchase | `PAID_IN_FULL` (default) or `INSTALLMENTS`. |
| `installments` | object[] | create_purchase | `INSTALLMENTS` only: `[{ dueDate, amount, paid? }]`, at least two, summing to the total (including any exclusive tax). `paid: true` marks one already received, e.g. the first, taken up front. |
| `purchaseId` | string | record_payment | The purchase; the earliest invoice still owed is used. |
| `invoiceId` | string | record_payment | One specific installment invoice instead. |
| `date` / `note` | string | record_payment | When it was received (default now), and e.g. "bank transfer, ref 1234". |

This is bookkeeping, the dashboard's **Add purchase** and **Mark as paid**. **Nothing here charges a
card, refunds, or sends an invoice or receipt** - those stay in the dashboard, behind wall 2 in
`guardrails.md`.

**`create_purchase` bills a client, which is why it is `send`.** The purchase appears in the
client's app; an installment plan arms the payment reminders that email or push the client as each
installment comes due, and a time-limited purchase arms the access-expiry reminders (both only when
the coach has switched reminders on); and an `ACTIVE` purchase notifies the coach's own connected
integrations, which on some accounts message the client. Confirm the client, product, amount and
schedule with the coach before calling it. A wrong purchase can only be removed in the dashboard.

**`record_payment` reaches nobody**, so it is `write`: it writes the payment into the invoice's
ledger (status `PARTIALLY_PAID` or `PAID`) and the client's history (`invoice.paid`), and can only
stop a reminder from going out. It refuses more than is owed, a voided invoice, and a purchase with
nothing owed (a `PENDING` paid-in-full purchase has no invoice yet). There is no un-record on this
surface, so record only what the coach says was received.

Read purchases with `find kind=purchase` - filter with `installmentState` (`OVERDUE` /
`DUE_SOON` / `ON_TRACK` / `FULLY_PAID`) for collections and with `expiresBefore` /
`expiresWithinDays` for renewals - and `get kind=purchase` for one in full.
