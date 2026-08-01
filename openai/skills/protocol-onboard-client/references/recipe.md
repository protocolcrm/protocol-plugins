# Recipe: onboard a new client

Bringing a new client into the tenant: create the base record, fill in the profiles that drive
programming, then place them on the pipeline and assign a trainer. All of this is one verb,
`manage_client`, called a few times with different params.

Verb + param source: **[surface-clients.md](../../protocol-reference/references/surface-clients.md)**.

## Sequence

1. **Create the base record.**
   ```
   manage_client create={firstName, lastName, email, phoneNumber?, sendAccessInstructions?}
   ```
   `create` is the object param that carries the new client's base fields
   (`firstName`/`lastName`/`email` required-in-practice, `phoneNumber` and
   `sendAccessInstructions` optional). The response returns the new client's id — hold onto it,
   every later call in this recipe needs it as `clientId`.

2. **Fill in the profiles** that drive programming and coaching decisions:
   ```
   manage_client clientId=<clientId> healthProfile={...} fitnessProfile={...} nutritionProfile={...} behavioralProfile={...}
   ```
   Each of `healthProfile`/`fitnessProfile`/`nutritionProfile`/`behavioralProfile` is a **patch**,
   not a full replace — pass only what the intake actually gave you, and only the profiles you
   have data for (you don't have to send all four in one call, or at all if the coach will fill
   them in later).

3. **Place them on the pipeline and assign a trainer** — usually done together, in the same call:
   ```
   manage_client clientId=<clientId> lifecycleStageId=<stageId> assignTrainerId=<trainerId>
   ```
   `lifecycleStageId` is nullable — pass an id to place the client on a stage, or `null` to clear
   it. `assignTrainerId` creates the trainer assignment; the response now surfaces the assignment
   itself under an `assignment` key (including its `id`), so if you ever need to reverse the
   assignment later you already have the id `unassignAssignmentId` expects — no extra lookup call.

## Gotchas

- **`lifecycleStage` (no `Id` suffix) is a different, easy-to-confuse param.** It's a sub-object
  (`{action: create|update|reorder, ...}`) that edits the tenant's *stage list itself* — the
  pipeline's columns — not any one client's position on it. To move **this** client, always use
  the top-level `lifecycleStageId`; only reach for `lifecycleStage` when the coach is asking to
  add/rename/reorder pipeline stages tenant-wide.
- Param-fidelity rule applies here same as everywhere on this surface: use `healthProfile` /
  `fitnessProfile` / `nutritionProfile` / `behavioralProfile` exactly as spelled in
  `../../protocol-reference/references/surface-clients.md` — a call that "succeeds" with a
  misspelled profile key will silently skip that patch rather than error.

## See also

- `../../protocol-reference/references/surface-clients.md` — the full `manage_client` param list.
- `../../protocol-reference/references/pitfalls.md`: the `lifecycleStage` vs. `lifecycleStageId`
  distinction (see "Two things that look like bugs but are not").
- `../../protocol-reference/references/guardrails.md` — write posture: this hits the live database
  directly, no draft/approve step.
- [find and review a client recipe](../../protocol-client-review/references/recipe.md) — how to
  re-locate this client afterward.
- [build and assign a program recipe](../../protocol-build-program/references/recipe.md) — the
  natural next step once onboarding is done.
