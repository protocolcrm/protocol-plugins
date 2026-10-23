# Protocol MCP surface - programming

Building and assigning what a client trains and eats: programs, workouts, nutrition templates,
and the exercise/food library behind them. Assumes `surface-core.md`, especially the replace
grammar - every array here follows it.

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

### `build_program`

No required params (omit `programId` to create). 2 actions: `build` (the default) and `translate`.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | `build` (default) · `translate`. |
| `programId` | string | Edit this program; omit to create. `translate`: the program to translate. |
| `name` | string | On create, the name; **on an edit it renames** (applied as a metadata patch). |
| `userId` | string | Assign to a client; omit for a library template. |
| `programType` | string enum | `WORKOUT` · `NUTRITION` · `FULL`. Also applies on an edit. |
| `sections` | string[] | Plan sections. Also applies on an edit. |
| `phases` | object[] | Weekly phases (one phase = one week). **Replaces the phase list** — see [the replace grammar](#the-replace-grammar-phases-items-groups); `add` is a **boolean** here. Dropping an existing week needs `confirmDelete`. |
| `confirmDelete` | boolean | `true` only after the coach confirmed deleting what a refused call listed: weeks (`weeksToDelete`) or content (`contentToDelete`). In a call that sends both `phases` and `content`, it confirms the weeks only. |
| `confirmContentDelete` | boolean | `true` only after the coach confirmed deleting the content (`contentToDelete`) a refused call listed. Needed when the call also sends `phases`; with `content` alone, `confirmDelete` works too. |
| `content` | object | `{ collections: [...] }` — replaces program content. The **menu-as-content** model; see below. Dropping an existing collection or item needs `confirmDelete`. |
| `metadata` | object | Partial patch — **only** `name` · `description` · `programGoal` · `difficultyLevel` · `totalDurationWeeks` · `tags` · `programType` · `sections` · `planningType` · `startDate` · `endDate`. Any other key is refused, naming it. |
| `duplicateFrom` | string | Copy this program into the coach's own library (see below). |
| `targetLocale` | string | `translate`: the language to translate into, e.g. `"sr"`, `"en"`, `"fr"`. |
| `translationInstructions` | string | `translate`: optional guidance for the translator (terminology, tone). |
| `importWorkoutId` | string | Import this workout into the library. |
| `duplicatePhaseId` | string | Duplicate this phase within the program. |

Copying/assigning to a client is `assign_program`, not this verb.

**Deleting weeks is confirmed, not implied.** `phases` replaces the list, so a week you leave out
is deleted. A write that would delete existing weeks is now **refused**: the error lists them under
`weeksToDelete` (name, position, workout placements) and nothing is saved. Show the coach the list;
only if they say yes, repeat the same call with `confirmDelete: true`. Never set `confirmDelete`
pre-emptively. Every agent edit also records a version, so the coach can restore a program from its
version history in the dashboard.

**Dates and status.** `metadata.startDate` / `endDate` are `YYYY-MM-DD`. Changing `startDate` or
`totalDurationWeeks` recomputes `endDate` as startDate + weeks × 7 unless you pass `endDate` too.
`status` is **not** a metadata field (it is refused, not dropped): use `assign_program` `activate`,
`deactivate` or `expire`.

**`duplicateFrom` — copy into the library.** `{ duplicateFrom: <programId>, name? }` makes a new
private template owned by the coach: no client, no dates, every week, workout and nutrition template
copied into new rows (the dashboard's Duplicate). `name` defaults to `"<source> (Copy)"`; `metadata`
applies to the copy. It refuses `programId`, `userId`, `phases` and `content` in the same call —
the copy has new ids, so duplicate first and then edit the copy with the returned `programId`. To
put a copy on a client, use `assign_program action=assign`.

**`action=translate` — translate a whole program.** `{ action: "translate", programId,
targetLocale }` starts a background job that overwrites the program, its weeks, its content
collections and **every workout and nutrition template it references** in the target language. It
spends the tenant's AI credits and is refused up front when the tenant is over its AI usage limit;
also refused while another translation of the program runs, and for template-linked or
public-library programs. A restore point is recorded per entity before it is overwritten. Follow
it with `get kind=program`: its `translation` field carries `status` (`PENDING` / `RUNNING` /
`COMPLETED` / `ERROR`), `targetLocale`, step counts and `lastLog` (why an `ERROR` stopped). Confirm
with the coach before starting: it changes the language the client sees.

`phases[].nutritionGoal` values: `AGGRESSIVE_DEFICIT` · `DEFICIT` · `MINOR_DEFICIT` ·
`MAINTENANCE` · `MINOR_SURPLUS` · `SURPLUS`. This one is **not** schema-validated — a bad value is
persisted verbatim.

#### `content` — the menu-as-content model

Some coaches deliver food as a menu on the program instead of as day plans. `content.collections` is
that library: a collection groups items, and an item of type `SUBCOLLECTION` nests more items to any
depth (max 6) — meal-time sections like Doručak / Ručak / Večera.

```
content: { collections: [
  { name: "JELOVNIK", type: "recipes", items: [
      { type: "SUBCOLLECTION", name: "Doručak", items: [
          { type: "NUTRITION_TEMPLATE", nutritionTemplateId: "<recipe id>" } ] } ] } ] }
```

| Item `type` | Carries |
|---|---|
| `SUBCOLLECTION` | `name` + nested `items` (a meal-time section) |
| `NUTRITION_TEMPLATE` | `nutritionTemplateId` — a library recipe, **or** an embedded `nutritionTemplate` snapshot (a text-only option) |
| `MEDIA` | `mediaId` — photo / video / PDF |
| `WORKOUT` | `workoutId` |
| `ARTICLE` | `articleId` — an article you can open (`find kind=article`) |

Collection `type` is a free label (`recipes`, `guides`, `general`, …). New items are referenced **by
id** and snapshotted in; an id that does not resolve (or names something you cannot open) is skipped
and listed in the result's `unresolvedIds`.

**Editing a menu: read, change, write everything back with its ids.** Call `get kind=program`,
change what you need inside its `contentCollections`, and send **all of it** back as
`content.collections`, keeping every collection's and item's `id`. Each item is matched to the
stored one by `id`, and every field you send back unchanged stays exactly as stored (hydrated media
URLs and expanded articles included), so the write changes only what you edited. Most menu options
are **text only**: a `NUTRITION_TEMPLATE` item whose `nutritionTemplate` has `id: null`, no rows,
and the recipe in its `description`. There is no library recipe to reference — edit the option by
changing that text. A new text option is the same item without an `id`, its `nutritionTemplate`
carrying a `name` or `description`.

A **`nutritionTemplateId` re-snapshots the current library recipe** into the item, replacing the
recipe text stored in it; the item keeps its `name` unless you send a new one.

A content write keeps the program's `sections` as stored, unset (null) included. An empty or unset
list is how a content-only program is saved, and every app shows its content as it is, so there is
no need to add `CONTENT`.

**Removing content is confirmed, not implied.** A content write that would drop existing
collections, sections or items is **refused**: the error lists them under `contentToDelete` (`kind`,
`name`, `path`) and nothing is saved. Show the coach; only if they say yes, repeat the call with
`confirmDelete: true`, or `confirmContentDelete: true` when the same call also sends `phases` (the
weeks confirmation never deletes content). A call with both `phases` and `content` checks the
content before writing anything, so a content refusal leaves the weeks as they were. On the create
path the program already exists when a later step is refused; the error carries its `programId`.
A content write is also refused on a **template-linked** program (its content
is read from the template, whose id the error names); editing the template changes every client
linked to it. An embedded snapshot that adds media you cannot open (another account's, or a
teammate's private media) is refused (`inaccessibleMediaIds`); reference media you can open by
`mediaId`. Likewise an `{ type: "ARTICLE", articleId }` placed anywhere inside the content (in a
snapshot, an edited item) that you cannot open (another account's, a teammate's, or no such article)
is refused (`inaccessibleArticleIds`); articles already on the program stay.

> **Read/write asymmetry.** You WRITE a new subcollection's children as `items`. `get kind=program`
> returns them under `subcollection.items`. Both are accepted on write — but read the nested items
> from `subcollection.items` or you will think they vanished.

For per-DAY content use `phases[].contentDays` instead.

#### Mixed weeks — training *and* nutrition on the same phase

The common real shape. A phase carries parallel day arrays, seven entries each, and they are
independent: editing one does not disturb the other.

| Array | A day looks like |
|---|---|
| `workoutDays` | `{ workoutIds: ["<workoutId>"], isRestDay: false }`, or `{ isRestDay: true }` |
| `nutritionDays` | `{ plannedNutritionTemplates: ["<templateId>"] }` |
| `contentDays` | `{ mediaIds: ["<mediaId>"] }` |

Nutrition placed on a `workoutDay` is **not rendered** — it must go in `nutritionDays`.

**A workout id may appear only once per program.** Completion is stored per workout row, with no
week and no day attached, so two weeks referencing one id also share their tick: the client trains
one Monday and every other week shows that day done. Write the same session into as many weeks as
the plan calls for — each repeat is given its own copy automatically, and the response reports
`copiedRepeatedWorkouts`. The consequence worth knowing is that the weeks are then independent:
editing week 3 no longer changes weeks 1, 2 and 4. Template programs are left alone, because
`assign_program` already copies per placement.

`plannedNutritionTemplates` ids are **not** affected — reuse one meal plan across every week freely,
which is what makes editing it once update them all.

Day objects are **not** reconciled the way rows are: they are stored as given. A day whose fields are
not from the real vocabulary is rejected outright rather than coerced (an invented
`{ day: "Monday", workout: "Upper A" }` used to become a silent rest day). The accepted fields are
`workoutIds`, `plannedNutritionTemplates`, `recoveryActivities`, `mediaIds`, `isRestDay`, `name`,
`dayType`, `dayFocus`, `notes`. A bare `{}` is a valid empty day.

---

### `assign_program`

Required: `action`. 6 actions.

| Param | Type | Notes |
|---|---|---|
| `action` | string enum | **Required.** `assign` · `activate` · `deactivate` · `expire` · `move` · `unlink` |
| `programId` | string | The program to act on (activate / deactivate / expire / move / unlink). |
| `copyFromProgramId` | string | `assign`: the source program/template to deep-copy. |
| `userId` | string | `assign`: the client who will own the new copy. |
| `name` | string | `assign`: name for the new program. |
| `startDate` | string | `activate`: `YYYY-MM-DD`; defaults to today. A new start moves `endDate` with it (start + `totalDurationWeeks` × 7, or the old span). |
| `position` | integer | `move`: 0-based position in the client's program list. |

**Which lifecycle action.** `deactivate` sets **PAUSED** — on hold, reversible with `activate`.
`expire` sets **EXPIRED**, `isActive: false` and `endDate` today (an end already in the past is
kept): the old block when a new mesocycle replaces it. `expire` refuses a library program.

**`copyFromProgramId` is the source; `programId` is not.** On `assign` they are different things —
`programId` names the program to act on for the *other four* actions. Passing `programId` to
`assign` is the easiest mistake on this verb, and it used to create a brand-new **empty** program
assigned to the client while reporting success. It is refused now, but know which key you mean.
`assign` also requires `userId`: a copy owned by nobody is a library duplicate, not an assignment.

`assign` produces an **independent deep copy** — its `templateId` stays null by design. That is not
a bug to correct. The copy is independent all the way down: the source's **workouts and nutrition
templates are duplicated into new rows too**, so editing the client's session does not change the
template it came from. Edit the client's copy for that client; edit the source to change the recipe
for everyone you assign it to next.

`unlink` only applies to programs that carry a `templateId` — those come from the web app's link
flow. A program you assigned is already independent, so `unlink` answers "not linked to a template".
That is the correct answer, not a failure.

**The copy duplicates per DAY-REFERENCE, not per distinct item.** A 4-week block whose seven days
each point at the same nutrition plan yields 28 copies of it; a workout used in all four weeks
becomes four rows. This matches ~99.7% of the product's real programs (measured), so it is the house
shape, not a defect — but two things follow. Titles are copied **verbatim**, so a workout named after
one client will carry that name into the next client's plan; name sessions by content, never by
person. And an edit to the client's copy applies to the day you edit, not to every day that
originally shared the item.

---

### `build_workout`

No required params (omit `workoutId` to create).

| Param | Type | Notes |
|---|---|---|
| `workoutId` | string | Edit this workout; omit to create. |
| `name` | string | |
| `userId` | string | Assign to a client; omit for a template. |
| `isTemplate` | boolean | |
| `difficulty` | string enum | `EASY` · `MODERATE` · `HARD` · `VERY_HARD` |
| `goal` | string enum | `WEIGHT_LOSS` · `MUSCLE_GAIN` · `STRENGTH` · `ENDURANCE` · `FLEXIBILITY` · `SPORT_SPECIFIC` · `GENERAL_FITNESS` · `REHABILITATION` |
| `durationMinutes` | number | |
| `exercises` | object[] | The exercise groups. **Replaces the whole tree** — see [the replace grammar](#the-replace-grammar-phases-items-groups); `add` names the row TYPE here (`"GROUP"` / `"EXERCISE"`). Dropping existing groups/exercises needs `confirmDelete`. |
| `confirmDelete` | boolean | `true` only after the coach confirmed deleting the rows a refused call listed. |
| `metadata` | object | Partial metadata patch (name / difficulty / goal / …). |
| `duplicateFrom` | string | Copy this workout into the coach's own library as a new private template. `name` / `difficulty` / `goal` / `metadata` apply to the copy; refuses `workoutId`, `userId` and `exercises` in the same call. |

The param is `exercises` — not `groups`, not `exerciseGroups`. Both enum values are also re-checked
on the `metadata` patch path, so a bad `metadata.difficulty` gives a clean error. Top-level `name`,
`difficulty`, `goal` and `durationMinutes` apply on an **edit** too, not only on create.

#### The two-level shape

GROUP rows at the top; each group's movements go in its own nested `exercises` array. Exercises
never sit at the top level.

```
{ add:"GROUP", section:"MAIN", format:"SUPERSET", restAfterGroupSeconds:120,
  exercises:[ { add:"EXERCISE", exerciseId:"<id>", sets:4, repRule:"8-10", restAfterSeconds:90 } ] }
```

| Field | On | Notes |
|---|---|---|
| `section` | GROUP | `WARMUP` · `MAIN` · `FINISHER` · `COOLDOWN`. Validated — a descriptive value like "Strength" is rejected. |
| `format` | GROUP | `REGULAR` (the overwhelming default) · `SUPERSET` · `CIRCUIT` · `DROPSET` · `AMRAP` · `EMOM` · `TABATA` · `FOR_TIME` · … Validated. |
| `rounds`, `timeCapSeconds`, `restBetweenRoundsSeconds` | GROUP | For circuits / AMRAP / EMOM. |
| `restAfterGroupSeconds` | GROUP | Rest after the block. |
| `exerciseId` | EXERCISE | A library id — resolve it with `find kind=exercise` first. A name is not a reference. |
| `sets`, `repRule` | EXERCISE | There is **no `reps` field**; reps live in `repRule`. |
| `restAfterSeconds` | EXERCISE | Rest between sets. **Set this** — it is on ~3 of every 4 real exercises. |
| `weight`, `tempo`, `notes` | EXERCISE | `rir` / `rpe` exist but are barely used in practice. |

`repRule` is a **structured grammar**, not free text — the same one the coach's builder enforces, so
a value it rejects would render red in their UI. Valid: `"10"`, `"8-10"`, `"10 / 8 / 6"`,
`"10x25kg"`, `"30s"`, `"5:00 / 4:30"`, `"bw"`, `"80%"`. Rejected: `"AMRAP"`, `"to failure"`,
`"60s hold"`, and anything with an unknown unit. Units: `kg` `lbs` `lb` `%` `bw` `bodyweight` `m`
`meters` `yd` `yards` `min` `s` `sec` — note `cal`, `km` and `in` are **not** supported by either
side. Unilateral work ("10 each side") has no notation: put it in `notes` and keep `repRule` numeric.

---

### `build_nutrition`

No required params (omit `templateId` to create).

| Param | Type | Notes |
|---|---|---|
| `templateId` | string | Edit this template; omit to create. |
| `name` | string | |
| `userId` | string | Assign to a client; omit for a library template. |
| `items` | object[] | The full **flat** item tree — a `LABEL` heading row followed by its foods as sibling `MEAL` rows, all top-level (foods do **not** nest inside the label; nested `items` are only for `GROUP` alternatives). A food row is `{ add:"MEAL", foodId, quantity, measure }`. **Replaces the tree** — see [the replace grammar](#the-replace-grammar-phases-items-groups). |
| `confirmDelete` | boolean | `true` only after the coach confirmed deleting the rows a refused call listed. Dropping existing rows without it is refused. |
| `metadata` | object | Partial patch. Only `name` / `description` / `tags` / `templateMode` are honored. |
| `duplicateFrom` | string | Copy this template into the coach's own library (items, macros and mode carried; every row gets a new id, so re-read the copy before editing). A team owner's or admin's copy belongs to the business library (`ownerId` = the team owner), like everything they create. `name` / `metadata` apply to the copy; refuses `templateId`, `userId` and `items` in the same call. |

A top-level `name` renames the template on an **edit** too, not only on create.

`metadata.templateMode`: `ADVANCED` (default — a "My Daily Nutrition" day plan) · `SIMPLE` (a
"My Meals/Recipes" entry). Pass it at create time so the template lands in the right place.
Macros are computed for you — don't hand-total them.

**Choose-one groups.** A `GROUP` row's `items` are its options. A `MEAL` there is a one-food
option; a `GROUP` holding several `MEAL`s is **one** multi-food option (banana + apple). An option
counts the **sum** of its foods and the group the **average** across its options, as the web
builder shows it. Sending the tree back with `ref`s leaves it exactly as it was.

**Swapping a food.** `{ modify: <rowId>, foodId: <newFood> }` with no `name` renames the row to the
new food, and a one-food option still named after the old food follows it; pass `name` to keep a
custom one. Nothing regenerates the template `description`: on a `SIMPLE` recipe it is the coach's
own ingredient list and method, so patch `metadata.description` in the same call when it names the
food you swapped.

---

### `manage_library`

Required: `action`. 5 actions.

| Param | Type | Action | Notes |
|---|---|---|---|
| `action` | string enum | — | **Required.** `create_exercise` · `update_exercise` · `resolve_foods` · `favorite_exercise` · `unfavorite_exercise` |
| `exerciseId` | string | update_exercise, favorite_exercise, unfavorite_exercise | Required for those three. |
| `name` | string | both exercise actions | |
| `primaryMuscleGroups` | string[] | exercise | The supported muscle-group path. |
| `equipmentRequired` | string[] | exercise | |
| `exerciseType` | string | exercise | |
| `difficultyLevel` | string | exercise | |
| `isCompound` | boolean | exercise | |
| `instructions` | string | exercise | |
| `videoUrl` | string | exercise | |
| `names` | string[] | resolve_foods | Convenience form — plain food names. Up to 50 per call. |
| `queries` | object[] | resolve_foods | `{ name, quantity?, measure? }`. Takes precedence over `names`. Up to 50 per call. |

**`resolve_foods` output.** `results[]` holds `{ query, matches[] }`, up to 3 matches per name, each
with `foodId`, per-100g macros and `measures`. Names past the 50th are not searched: they come back
in `notProcessed` with a `note`, so resend them in another call. A match carrying `unmatchedWords`
does not contain those words of the query: "sojin jogurt" matching "Grčki jogurt" with
`unmatchedWords: ["sojin"]` is a dairy yogurt, not a soy one, so treat it as a different food.
The same goes for a near-miss on a different word: "maslac" (butter) matching "Masline" (olives)
comes back with `unmatchedWords: ["maslac"]`, while an inflected form of the same word ("jaja" for
"Jaje", "mleka" for "Mleko") is not flagged. When every match is flagged, the coach has no such
food under that name: try the English name ("butter") before settling for one. The
shared library names most foods in English, plant-based and dairy-free ones included (Soy Yogurt,
Almond Milk, Oat Milk, Soy Protein Isolate), while a coach's own foods may be in their language.
Serbian qualifiers are understood (sojin, bademovo, ovseno, kokosovo, pirinčano, bez laktoze,
biljno, veganski, posno, integralno, lajt/light): "sojin jogurt" also searches "soy yogurt", and every
food whose name carries the qualifier ranks above every food that lacks it, so "Soy Yogurt" comes
before "Grčki jogurt". A food named exactly what you searched (case, accents and punctuation aside:
"Coca-Cola Light" for "coca cola light") always ranks first, and "light" in a query whose other
words already name a food is an ordinary word, not a qualifier. For other qualifiers or languages, when a qualified name finds nothing, search
the English name too. A food's `category` is coarse: plant
milks are `BEVERAGES` (subcategory `PLANT_MILKS`) and soy yogurt is `SOY`, but a coach's own foods
and older rows can still sit under `DAIRY`, so never decide "dairy-free" from it.

**Favorites are the coach's own shelf.** `favorite_exercise` stars an exercise for this coach and
`unfavorite_exercise` removes the star; both are idempotent and touch only the coach's own list,
never the exercise (a teammate's favorites are untouched, and a public-library exercise can be
starred). `find kind=exercise` rows carry `isFavorite`, and `favorites: true` narrows the list to
the starred ones: read them first when a coach says "use my usual exercises". `find
kind=recent_exercise` is the other half, the exercises this coach put into their own workouts most
recently. Both beat a free-text search through 7,000 public rows. `find kind=exercise` also takes
`tags` (`key:value`, all must match) for a tenant that tags its library, and `variants: "all"` to
see every gender/version variant of a public exercise instead of one row each.
