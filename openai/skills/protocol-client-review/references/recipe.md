# Recipe: find a client, then review their full context

The cheapest, most common sequence: a coach names a client (by name, partial name, or email) and
you need their full context before doing anything else. Two read-tier verbs, no writes.

Verb + param source: **[surface-core.md](../../protocol-reference/references/surface-core.md)**
(`find`/`get`) and **[surface-clients.md](../../protocol-reference/references/surface-clients.md)**
(`review_client`).

## Sequence

1. **Locate the client.**
   ```
   find kind=client query=<name, partial name, or email>
   ```
   `kind` and `query` are both accepted `find` params. `find` forwards `query` as both `searchTerm`
   and `name` internally, so you don't need to guess which legacy tool is behind `kind=client` —
   just pass `query`. Read the returned list and pick the right client's `id` (watch for
   near-duplicate names — confirm with email or another distinguishing field before proceeding if
   more than one result comes back).

2. **Pull full detail on that one client**, either or both of:
   ```
   get kind=client id=<clientId>
   ```
   (`get`'s params are always `kind` + `id` — the entity's own id, here the client's user id.)
   ```
   review_client clientId=<clientId>
   ```
   `review_client` takes `clientId` (required) and returns a single bundle: the client record,
   their profiles, recent programs, nutrition, progress, upcoming appointments, open tasks, and
   insights — each section is null-safe on its own failure, so a missing profile or empty task
   list won't blank out the rest of the bundle.

   Use `get kind=client id=<clientId>` when you only need the base client record; reach for
   `review_client clientId=<clientId>` when the coach's question spans multiple areas (e.g. "how's
   this client doing overall") and you want the one-call bundle instead of several separate
   `find`/`get` calls.

3. **Read the bundle, section by section.** Because each section fails independently, check for
   nulls rather than assuming the whole call failed if one part (say, `insights`) comes back
   empty — that's expected behavior, not an error.

## Gotchas

- `find`'s filters (`status`, `isTemplate`, `clientId`, etc.) silently no-op for any kind whose
  underlying legacy list tool doesn't support that filter — if a filtered `find kind=client` looks
  like it ignored your filter, that's the known behavior, not a bug (see the "Filters that don't
  apply just no-op" pitfall in `../../protocol-reference/references/pitfalls.md`).

## See also

- `../../protocol-reference/references/surface-core.md` — `find`/`get` kind table.
- `../../protocol-reference/references/surface-clients.md`: the `review_client` entry.
- `../../protocol-reference/references/data-model.md` — `User` entity shape (profiles are separate
  one-to-one resources, not columns on the client record itself).
- [onboard a new client recipe](../../protocol-onboard-client/references/recipe.md) — the
  write-tier follow-on once you've found (or failed to find) a client.
