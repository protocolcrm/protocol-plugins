# Recipe: build a program (with a workout) and assign it to a client

Building structure and assigning it to a client are deliberately **separate steps** on this
surface: `build_program`/`build_workout` shape the content, `assign_program` handles the
client-facing lifecycle (copy, activate, move). Don't try to shortcut this by writing `userId`
directly into a template you intend to reuse — assignment is what makes a client's copy
independent.

Verb + param source: **[surface-programming.md](../../protocol-reference/references/surface-programming.md)**.

## Sequence

1. **Build the workout content** (skip if you're assigning a nutrition-only program):
   ```
   build_workout name=<name> isTemplate=true difficulty=<EASY|MODERATE|HARD|VERY_HARD> goal=<enum> durationMinutes=<n> exercises=[...]
   ```
   `difficulty` and `goal` are enum-validated client-side — use the exact values from
   `../../protocol-reference/references/surface-programming.md`'s enum table (`difficulty`:
   `EASY`·`MODERATE`·`HARD`·`VERY_HARD`; `goal`:
   `WEIGHT_LOSS`·`MUSCLE_GAIN`·`STRENGTH`·`ENDURANCE`·`FLEXIBILITY`·`SPORT_SPECIFIC`·
   `GENERAL_FITNESS`·`REHABILITATION`). `exercises` is the full exercise tree and **replaces the
   whole thing** on every call — always pass the complete tree, not a delta. Mark it
   `isTemplate=true` if this is reusable structure rather than a one-off for a single client.
   Hold onto the returned `workoutId`.

2. **Build the program structure**, referencing that workout:
   ```
   build_program programType=<WORKOUT|NUTRITION|FULL> name=<name> phases=[...] sections=[...]
   ```
   `programType` is enum-validated (`WORKOUT`·`NUTRITION`·`FULL`). `phases` is an array where one
   phase = one week, and it **replaces the whole list** on every call, same as `exercises` above.
   If a phase carries a `nutritionGoal`, note it is **not** schema-validated by `build_program`
   itself (unlike `difficulty`/`goal` on `build_workout`) — double-check the value against
   `../../protocol-reference/references/surface-programming.md`'s enum table
   (`AGGRESSIVE_DEFICIT`·`DEFICIT`·`MINOR_DEFICIT`·`MAINTENANCE`·
   `MINOR_SURPLUS`·`SURPLUS`) yourself before sending it, since a typo won't be caught for you.
   Hold onto the returned `programId` — this is your reusable template/source program.

3. **Assign a copy of it to the client:**
   ```
   assign_program action=assign copyFromProgramId=<programId> userId=<clientId> name=<optional override name>
   ```
   `action=assign` **deep-copies** the source program as an independent copy for that client —
   `templateId` on the resulting copy stays `null` by design, so don't "fix" it later. Hold onto
   the **new** program id the assignment returns; that's the client's own copy, distinct from the
   source `programId`.

4. **Activate the client's copy:**
   ```
   assign_program action=activate programId=<assignedProgramId> startDate=<date>
   ```
   `action` is required and enum-validated; `startDate` only applies to `activate`.

## Gotchas

- **`build_workout.exercises` forwards to the legacy tool as `groups`, not `exerciseGroups`.**
  This is the #1 pitfall in `../../protocol-reference/references/pitfalls.md` — a past bug where
  create-only calls worked but any call also passing exercises silently created an empty workout.
  The param you pass is still `exercises` (don't rename it yourself); just know it isn't forwarded
  verbatim under the hood.
- **`assign_program`'s `assign` action never sets `templateId`** on the resulting copy — this is
  intentional (the copy is independent of its source), not a bug to correct.
- A failing sub-call inside a composite verb (e.g. `build_program`'s create → phases → content
  sequence) stops the whole call — it won't partially apply the rest. If a build call errors,
  assume nothing after the failure point was saved and retry the whole call once you've fixed the
  input.

## See also

- `../../protocol-reference/references/surface-programming.md` — full param lists for
  `build_program`, `build_workout`, `assign_program`, and the enum vocabularies table.
- `../../protocol-reference/references/data-model.md` — `Program`/`WorkoutModel` entity shapes
  (`phases`/`sections` and `exercise_groups` are jsonb blobs with no schema enforcement at the DB
  layer).
- [onboard a new client recipe](../../protocol-onboard-client/references/recipe.md) — the step
  that usually precedes this one.
- [weekly check-in and report recipe](../../protocol-checkin-cycle/references/recipe.md) — how
  progress against an assigned program gets reviewed later.
