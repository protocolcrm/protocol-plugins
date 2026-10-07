# Protocol MCP surface - running the practice

The calendar, the kanban, the media library, articles, the inbox, and automations - the operational
half that is not about one client's programming. Assumes `surface-core.md`.

> **One of four.** The surface is split by the job you are doing, so you read the part you need
> rather than all 22 verbs:
>
> | File | Verbs |
> |---|---|
> | `surface-core.md` | `find` · `get` · `report` · the kind table · the replace grammar · `report_to_developers` |
> | `surface-programming.md` | `build_program` · `build_workout` · `build_nutrition` · `assign_program` · `manage_library` |
> | `surface-clients.md` | `manage_client` · `record_progress` · `manage_forms` · `review_client` · `message` |
> | `surface-operations.md` | `manage_tasks` · `manage_media` · `manage_content` · `schedule` · `manage_automations` · `review_inbox` · `manage_support` |


---

### `manage_tasks`

Required: `action`. 16 actions.

`create_task` · `update_task` · `complete_task` · `move_task` · `archive_completed` ·
`create_subtask` · `update_subtask` · `toggle_subtask` · `reorder_subtasks` · `create_board` ·
`update_board` · `create_column` · `update_column` · `reorder_columns` · `create_label` ·
`update_label`

Everything except `action` is forwarded verbatim; pass the fields that action needs.

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
it is silently dropped - the write returns success and nothing changes. `create_task` also needs a
`boardId` or a `columnId`; a title alone is refused. The per-action parameter list is published on
the `action` enum - read it rather than guessing, because anything an action does not read is
dropped rather than rejected. `reorder_subtasks` / `reorder_columns` take the COMPLETE id list.

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

Required: `action`. 6 actions. Articles are the coach's rich content pieces - a title, category,
cover, and a block body - that clients read in the app's Learn tab and inside programs. You write
and read the body as **Markdown**; the platform stores structured content and renders it itself.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `create_article` · `update_article` · `publish_article` · `unpublish_article` · `attach_to_program` · `create_audience` |
| `articleId` | string | Every article action but `create_article`. |
| `title` | string | Required on create. The slug is derived from it once and does not follow later renames. |
| `markdown` | string | The body. On `update_article` it **replaces the whole body** - read with `get kind=article` first. |
| `category` | string | Free text. Reuse what `find kind=article` already shows rather than inventing a near-duplicate. |
| `excerpt` | string | One or two sentences for the card. Falls back to the first 200 characters of the text when empty. |
| `coverMediaId` | string | A media id (image or video). |
| `slug` | string | Optional override; made unique in the tenant. |
| `audience` | object | `{ all: true }` or `{ audiences: [saved audience names or ids] }`; reaching any listed audience is enough. Default on create: everyone. |
| `name` · `description` · `conditions` | | `create_audience`, see below. |
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
the body as `markdown`, and `unresolvedAudience` when a name did not match. An audience list that
resolves to **nothing** is refused outright.

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
| `endTime` | string | create, update | ISO-8601 instant. |
| `clientId` | string | create, reminder | |
| `type` | string | create, update | |
| `modality` | string | create, update | |
| `location` | string | create, update | |
| `description` | string | create, update, reminder | |
| `status` | string | update | |
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
`booking_config`, and `gcal_disconnect` are the ones that are genuinely internal writes and reach
nobody.

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

`create` lands the automation in DRAFT. `run` **dispatches an execution now**: outward. Read a
run's outcome with `find kind=automation_run` + `automationId`.
There are two registered kinds: `PROGRESS_REPORT` (triggers `PROGRESS_ENTRY_CREATED` and
`MANUAL`) and `FORM_REPORT` (triggers `FORM_SUBMITTED` and `MANUAL`); confirm with
`find kind=automation_kind` before assuming another exists. `run` needs `triggerData` - for
PROGRESS_REPORT that is `{ entryId: "<progress entry uuid>" }`, for FORM_REPORT that is
`{ submissionId: "<form submission uuid>" }`. **`run` is the
only outward action here, and it is the one held to the `send` tier** (see `guardrails.md`): if
this connection is at `write` (the default), calling `run` is refused with a
`PermissionDeniedError` naming the tier needed - that is expected, not a bug, so tell the coach
plainly and stop rather than escalating or falling back to a direct API key. If the
connection is at `send`, confirm with the coach before calling it anyway: getting the payload
wrong still dispatches a real, client-facing execution, so it costs more than a wasted call.
Authoring (create/update/activate/pause/archive) is a plain write, at `write` tier. Read what an
execution actually did with `find kind=automation_run`.

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
