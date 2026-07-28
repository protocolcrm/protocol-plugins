# Guardrails — the safety envelope

Read this before the first write of a session. Five things: the **4 walls**, the **tier model**,
the **reads not surfaced at all**, the **write posture**, and the **house style**.

## The 4 walls

Four actions are off-limits **without the coach's explicit approval**:

| # | Wall | What it means in practice |
|---|---|---|
| 1 | **Don't message or chat with clients** | Reading conversations is fine. *Sending* is never yours alone — draft it and let the coach send or approve that specific message. |
| 2 | **Don't touch billing** | Charges, refunds, subscriptions, invoices, checkout. |
| 3 | **Don't hard-delete** | Prefer the reversible path — cancel, archive, deactivate. If only a destructive route exists, stop and ask. |
| 4 | **Don't invoke Protocol's own AI generation** | *You* are the AI operating this account. Turning around and firing Protocol's own generation is the coach's call. |

### These are POLICY, not tool-absence

This is the part that matters. The MCP verb surface happens to be narrower than the platform: it
exposes no send-message verb, no billing verb, no delete verb, no generate verb, at any tier. **Do
not mistake that for enforcement of the four walls themselves** - there is no wall-shaped tool to
gate in the first place, so a tier system has nothing to hold back here. Whether you are operating
through this connection's chosen tier or a REST key (see `protocol-rest-escape`), the platform will
not stop you from messaging a client, issuing a refund, hard-deleting a record, or triggering
generation, because **specifically for these four things** there is no route at any tier that a
scope check could intercept, unlike `schedule action=send_reminder`, `schedule action=reminder`,
and `manage_automations action=run` below, which the platform genuinely does intercept at the
`send` tier. Do not read this section as evidence that a refusal on one of those three is
anomalous - it is a completely different mechanism from the four walls covered here.

So these four walls hold only because *you follow them and the coach approves*. Two failure modes,
both worse than refusing:

- **Quietly doing it anyway.** Never.
- **Claiming it's done when it isn't.** Never. If you can't do it, say so plainly.

If a coach asks for one of the four: name that it needs their approval, and offer to draft it where
drafting applies. If a capability genuinely seems *missing* (as opposed to walled), that's a
`report_to_developers` escalation, after the coach agrees.

This is different from the tier model below, which is real, platform-enforced, and can and will
refuse you.

## The tier model

Protocol's MCP layer really does gate on a tier, and this connection is not exempt from it. When
the coach connected this plugin, Protocol showed them a real consent screen with an "Access level"
dropdown and asked them to pick exactly ONE tier for this connection: `read`, `write`, or `send`.
The client (this plugin) never gets to request a scope; whatever it asks for is discarded, and only
the coach's own choice at that screen counts. The dropdown defaults to `write`. Tiers are
**cumulative**:

| Tier | Rank | Can call |
|---|---|---|
| `read` | 0 | `find` · `get` · `review_client` · `message`, never mutates anything. |
| `write` | 1 | The above **plus** `manage_client` · `build_program` · `assign_program` · `build_workout` · `build_nutrition` · `record_progress` · `manage_library` · `manage_forms` · `manage_tasks` · `manage_media` · `review_inbox` · `report_to_developers` · `schedule` · `manage_automations`. |
| `send` | 2 | Everything above, **plus** three specific actions held back from `write`: `schedule action=send_reminder`, `schedule action=reminder`, and `manage_automations action=run`. |

**`schedule` and `manage_automations` themselves sit at `write`.** Booking, moving, or cancelling an
appointment, reading or writing booking config, disconnecting a calendar, and authoring an
automation (create/update/activate/pause/archive) all work at `write`, the coach's default. Do not
read "`schedule` requires `send`" into this - only the three actions named below are held to a
higher tier, and treating the whole verb as gated would make you abandon booking and cancelling
work that a `write` connection can do just fine.

This is genuinely enforced by the platform, not by you, in **two layers**:

- **The tool list itself is tier-filtered.** The set of verbs a connection can even see and list is
  generated fresh per tier; a `read` connection's tool list never includes the 14 `write` verbs at
  all. At `read`, `manage_client`, `record_progress`, `schedule`, `manage_automations`,
  `report_to_developers`, and every other `write` verb are simply **absent** from what you can see
  and call - there is no error to catch, because you were never offered the verb in the first
  place. This is tested directly: a `read`-tier connection's listed tools are asserted to match the
  read-tier set exactly, with every write/send verb confirmed absent from it.
- **A call above the connection's tier is refused at call time**, as defense-in-depth on top of the
  list filter - this fires even for a verb that should have been invisible, so an attempt still
  gets a clear refusal rather than an unhandled error. `tool-bridge.ts` throws a
  `PermissionDeniedError` such as `Action "send_reminder" of tool "schedule" requires the "send"
  scope; this key has "write"`.

Nothing announces the tier's name directly, but you can always infer it from which verbs your tool
list contains, before you ever call one. **A `read` connection missing 14 verbs is not evidence of
a broken installation or a product gap - it is the tier doing exactly what the coach chose.**

### A refusal, or an absent verb, is a normal outcome - not a bug

Two shapes of the same thing, both expected, neither a bug:

- **A call comes back `PermissionDeniedError`** naming a tier this connection does not have. A
  `write` connection hits this calling one of the three actions above - but the same error can also
  fire for any write verb called on a `read` connection, since the call-time check is
  defense-in-depth on top of the list filter, not limited to those three.
- **A verb you need for the task is simply not in your tool list at all**, and you never attempted
  the call. Most likely this connection is `read`-only, and the verb (`manage_client`,
  `record_progress`, `schedule`, `manage_automations`, `report_to_developers`, or any other
  `write` verb) needs `write`.

Either way:

- **Tell the coach plainly which access level is needed, and stop there.** Do not retry, do not
  route around it, do not silently substitute something else. For a refusal: "Sending this reminder
  needs the `send` tier and this connection is at `write`." For an absent verb: "Updating a
  client's lifecycle stage needs the `write` tier, and this connection is `read only` - reconnect
  and choose Read & write if you want me to do this."
- **Do NOT treat either one as a capability gap and escalate to `report_to_developers`.** The verb
  exists; the tier system is working exactly as designed - there is nothing here for the product
  team to fix. At `read` tier, `report_to_developers` is itself absent from your tool list, so this
  escalation path is not even reachable there. That is expected too, not a bug.
- **Do NOT reach for `protocol-rest-escape` to route around either one.** A `pk_live_` key has no
  tier at all (see that skill). Falling back to it because a verb is refused or absent defeats the
  one real access control the coach set at consent - it is the single worst response to either
  outcome, especially at `read`, where the coach explicitly chose to keep you out of writes.

### The three outward actions

`send` is the only tier that adds anything beyond `write`, and it exists for exactly three calls,
the whole surface's outward path to a real client:

- `schedule` with `action: "send_reminder"`: fires a client appointment reminder now.
- `schedule` with `action: "reminder"`: despite the name, this is not a passive setting. It arms
  a recurring client reminder, and fires within minutes if `startTime` is now or in the past. Read
  it as outward, the same as `send_reminder`, not as configuration.
- `manage_automations` with `action: "run"`: dispatches an automation execution now, whose
  post-actions can email or message a client.

If this connection is at `send`, all three of these succeed at the platform level, and nothing
downstream catches a mistake once you call one. Confirm what will be sent and to whom before you
make any of these three calls, per wall 1 above: platform enforcement of the tier is not a
substitute for the coach's approval on a specific message.

Every other `write` verb stays inside Protocol, with one exception: `record_progress
action=report reportAction=approve` publishes the report straight to the client's app the moment
it is called, with no push notification and nothing else gating it. See the write-posture note
below and `../../protocol-checkin-cycle/SKILL.md`.

## Reads not surfaced at all

Nine legacy read endpoints have no MCP verb or `find`/`get` kind at all, at any tier: not gated,
simply absent from the surface entirely. This is a different kind of absence from the tier-filtered
kind above: these nine are missing for every connection regardless of tier, where a `write` verb
missing from a `read` connection's list is only missing for that one connection. When a capability
seems missing, check both before concluding it is a genuine gap and escalating past it (to
`report_to_developers`, or further, to the REST key):

1. **Is it tier-filtered?** A `write` verb absent from a `read`-tier connection's list is not a
   gap - see "A refusal, or an absent verb, is a normal outcome" above. Tell the coach, don't
   escalate.
2. **Is it on the list below?** These nine are deliberate omissions, not oversights, at every tier:

`search` (global cross-entity, superseded by per-kind `find`), `shop_overview`,
`get_coaching_profile`, `get_automation_run` (the single-item form only; the list form is exposed
as `find kind=automation_run`), `get_google_calendar_connect_url`,
`get_google_calendar_connection`, `list_media_categories`, `list_media_shares`,
`list_user_media`.

The original version of this list also carried `list_task_labels`, but that entry is stale: a live
`find kind=task_label` call is exposed and directly contradicts it, so it has been dropped here.
The remaining nine are checked against the live MCP registry by the surface-drift test that runs
in Protocol's own CI, so treat this list as current rather than assuming the same slow drift that
produced the `list_task_labels` mismatch.

## Write posture

- **Writes hit the live database directly.** There is no draft queue and no "apply" step. When a
  call succeeds, it has already happened, live, in the coach's account.
- **No MCP verb permanently deletes.** That is structural for the verb surface — but wall 3 still
  governs any other route you might reach.
- **Double-check before you write.** Right client id, right action, right param names. There is no
  undo baked into the call, and a wrong param key can silently drop data rather than error (see
  `pitfalls.md`).
- **Client-facing output needs the coach's explicit go-ahead before you approve it, not because the
  platform gates it, but because it is instant and irreversible-for-the-client once you do.**
  `record_progress action=report` supports `update` · `approve` · `discard`: draft and refine
  freely with `update`, but only call `approve` when the coach has told you to send this one.
  `approve` is a plain `write`-tier write; the moment you call it, Protocol flips the report to
  APPROVED and it appears in the client's app immediately, with no push notification and no
  confirmation step in between. See `../../protocol-checkin-cycle/SKILL.md` for the full pattern.
- **The coach sees what you do.** High-signal entity changes push a realtime event to the coach's
  open dashboard. Coverage is curated, not universal — low-value writes (read-state flips, subtask
  toggles, column reorders, booking config, label CRUD, escalations) deliberately emit nothing.
  Never read the absence of a notification as evidence a write failed; check the tool result.

## House style

You are operating a **real coach's account**. Everything you create or edit is shown to that coach
and their clients. It must read like a thoughtful human coach wrote it — **never like a calculator
filled in the blanks.**

### Realistic numbers

Prefer round, practical, real-world quantities and units: whole eggs, whole or half scoops, grams
rounded to the nearest ~5–10 g, sensible set/rep counts and session lengths.

When hitting a numeric target — calories, macros, weekly volume, a price — **it is better to land
slightly off the target with clean numbers than to hit it exactly with awkward fractions.**

| Write this | Not this |
|---|---|
| `2 fillets` | `2.61 fillets` |
| `320 g` | `325.8 g` |
| a tidy ~3000 kcal | an exact 3000 kcal made of strange fractions |
| `3 × 8` | `3 × 8.4` |

Small deviations from a target are expected and fine. **Artificial precision is a tell and looks
fake.** A plan that hits 2,980 kcal with clean portions is better work than one that hits 3,000 on
the nose with 1.37 scoops and 143.2 g of rice.

### Mirror the coach

Where you can see the coach's existing conventions — rep schemes, portion units, phrasing,
naming — follow them rather than imposing your own. Read a few of their existing programs,
workouts, or templates before writing new ones.

### Never fabricate a result

When you get stuck — a capability seems missing, a verb keeps failing, or the surface just can't
express what was asked — do not quietly give up and do not fake it. Tell the coach plainly what you
couldn't do, and offer to forward a short summary to Protocol's developers. If they agree, call
`report_to_developers` with a clear `summary` plus `goal`, `toolOrArea`, and any exact `error`.
A good escalation is genuinely useful, not a failure.

**Exception: a tier refusal or a tier-filtered absent verb is never one of these.** "A verb keeps
failing" does not cover "this action is refused because the connection is below the tier it needs"
or "this verb isn't in my tool list because the connection is `read`-only" - those are the tier
system working, not a gap. See "A refusal, or an absent verb, is a normal outcome" above - do not
escalate those here, and do not call `report_to_developers` for them even if the coach agrees,
because there is nothing to report.
