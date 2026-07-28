# REST API — the escape hatch, not the front door

**Default to MCP.** Everything in **[`../../protocol-reference/SKILL.md`](../../protocol-reference/SKILL.md)**
is the 18-verb surface you should reach for first — it's shaped for an operating agent,
tenant-scoped to one coach automatically, and covered by the delegation-pitfall notes there.
Reach for the raw REST API only when:

- MCP has **no verb or `find`/`get` kind** for the thing you need (check the "Reads not surfaced
  at all" section in
  [`../../protocol-reference/references/guardrails.md`](../../protocol-reference/references/guardrails.md#reads-not-surfaced-at-all),
  and the kind table in the reference hub, first: a gap may be intentional, not an oversight).
- You need a **bulk read** that would take many MCP round-trips (e.g. paging every workout in a
  tenant) and the equivalent REST endpoint supports larger page sizes or filters MCP doesn't
  expose.
- You've hit a genuine coverage drop, not a tier-filtered absence (see `guardrails.md`'s tier
  model), and are about to `report_to_developers`: the API can sometimes get you unblocked in the
  same session while that gap gets fixed upstream.

Don't reach for the API assuming it buys you something MCP won't allow - if anything, it is the
opposite. This connection carries whatever single tier the coach picked at consent (`read`,
`write`, or `send`, `write` by default; see `guardrails.md`), and the platform genuinely enforces
it in two layers: verbs above the connection's tier are absent from the MCP tool list entirely, and
any call that still reaches above the tier is refused with a `PermissionDeniedError`. A REST key
has no tier at all: it authenticates as the coach with **full permissions**, on every route,
unconditionally (see below). So a verb the tiered MCP connection can't even see, or a call it just
refused, would both go through over REST with the same underlying key, because REST checks
neither. Never use that fact to route around a refusal or an absent verb - if the coach set this
connection to `write`, reaching for the API the moment `send_reminder` is refused, or the moment
`manage_client` turns out to be missing on a `read` connection, is deciding for them that they
meant a higher level.

The four walls in `guardrails.md` (message, billing, delete, generate) are policy on both paths
equally, since neither path has a verb for them at all - that part really is the same on both
surfaces. What differs is the tier system: MCP has one and enforces it, REST does not.

- Coach-facing plain-language counterpart to key management: **help.protocolcrm.com** (Integrations
  → API keys page) — point a confused coach there rather than walking them through curl commands.
- The four walls and what an agent may write at all:
  **[`../../protocol-reference/references/guardrails.md`](../../protocol-reference/references/guardrails.md)**.
- Entity/column shapes the API's JSON actually serializes:
  **[`../../protocol-reference/references/data-model.md`](../../protocol-reference/references/data-model.md)**.

## Auth: `pk_live_` API access keys

External callers (including you, when you fall through to the API) authenticate with a
long-lived, revocable key instead of a login JWT:

```
Authorization: Bearer pk_live_<48 hex characters>
```

- **Format:** `pk_live_` followed by 48 hex characters. Anything else is not a REST key.
- **One auth branch, every route.** Any Bearer token starting with `pk_` is routed to key
  authentication instead of JWT verification, and a valid key resolves to the same principal the
  login path produces. It therefore works on every authenticated route with no per-route setup.
- **Acts as the owner, full permissions. There are no scoped keys in this version.** A key
  authenticates AS the team owner who created it. Treat it as equivalent to that coach's full
  login session: anything their account can do, the key can do.
- **Re-checked on every call.** Authentication reloads the user from the database and re-checks
  their current standing rather than what was true when the key was minted, so a demoted owner or
  a deactivated user's key stops working immediately even if it was never explicitly revoked.
- **Per user and per tenant.** A key is stamped with the creating user and tenant, and listing or
  revoking only ever sees keys belonging to that same pair.
- **Minting is owner-only.** Creating, listing, and revoking keys all require the account owner. A
  non-owner is refused with a 403 before anything touches the database.
- **Reveal-once, hashed at rest.** The raw key is returned only in the create response. Only a
  sha256 digest is stored, plus a short display prefix so the UI can show which key is which. If
  the coach loses the raw value they must mint a new one; it cannot be recovered.
- **Unknown, revoked, or unauthenticatable key gives 401, not 403.** A 403 means the key
  authenticated but the account lacks permission for that route.

### Three distinct key stores — do not conflate

| Store | Table | Who it authenticates as | Format |
|---|---|---|---|
| **API access keys** (this doc) | `api_access_keys` | The owner who minted it, full permissions, on any `@Authorized()` REST route | `pk_live_<48 hex>` |
| MCP agent connections | `agent_connections` | A coach principal, tier-scoped (`read`/`write`/`send`, cumulative). The coach picks exactly one tier at consent when they connect this plugin (`write` by default); the client's requested scope is discarded and never honored. Platform-enforced: a call above the connection's tier is refused. | `pk_<48 hex>` (no `live_`), see [`../../protocol-reference/SKILL.md`](../../protocol-reference/SKILL.md) |
| Legacy `api_keys` | `api_keys` | Tenant only, no per-user actor, no soft-revoke/audit trail. `DELETE /v1/auth/apiKey/:apiKeyId` hard-deletes the row (WordPress/Zapier) | plaintext, on `/v1/auth/apiKey`. Left untouched, do not extend |

This is a deliberate three-way split, not drift — mixing REST keys into `agent_connections` would
conflate two different lifecycles/UIs, and hardening the legacy plaintext table in place risked
breaking existing integrations.

If you are staring at a
`pk_` token and unsure which store it belongs to, check the substring after `pk_`: `live_` means
`api_access_keys` (REST); no `live_` means `agent_connections` (MCP).

## Response envelope & pagination

Every REST response (this API, not MCP) is wrapped identically to the legacy Kotlin contract:

```jsonc
{ "success": true, "message": "…", "data": { /* or [] or null */ } }
```


List endpoints nest a paginated envelope inside `data`:

```jsonc
{ "items": [...], "total": 137, "page": 1, "pageSize": 20, "totalPages": 7 }
```


**Page indexing is NOT consistent across controllers — check each endpoint's own doc string
before assuming.** Media endpoints are explicitly 0-indexed with `page` defaulting to `0`. Progress entries are
1-indexed. Do not assume every
other resource family follows the same convention — verify against `GET /v1/openapi.json` (or the
specific controller) rather than copying the media convention elsewhere.

## Resource families (base paths)

This is a map of the ~12-15 resource families an agent is likely to touch, not a transcription of
every endpoint — **the live spec is the source of truth for exact routes, params, and schemas**:

- `GET /v1/openapi.json` — machine-readable OpenAPI 3 spec, generated at boot from the same
  decorators that define the routes, so it cannot drift from the code. Public: no auth is
  required to fetch the spec itself
  (per-route auth still applies to calling the endpoints it describes).
- Swagger UI: **`https://api.protocolcrm.com/docs`**. Paste a
  `pk_live_` key into the "Authorize" button to call live endpoints from the browser.

| Family | Base path | Note |
|---|---|---|
| Users / clients | `/v1/users` |  |
| Client profiles (health/fitness/nutrition/behavioral) | `/v1/users/:id/profile/*` |  |
| Programs | `/v1/programs` |  |
| Workouts | `/v1/workout` |  |
| Workout tracking | `/v1/workoutTracking` |  |
| Exercises | `/v1/workout/exercises` |  |
| Nutrition tracking | `/v1/nutrition/tracking` |  |
| Nutrition food items | `/v1/nutrition/food-items` |  |
| Nutrition templates | `/v1/nutrition/templates` |  |
| Tasks | `/v1/tasks` |  |
| Progress entries | `/v1/progress-entries` |  |
| Appointments | `/v1/scheduling/appointments` |  |
| Conversations | `/v1/communication/conversations` |  |
| Messages | `/v1/communication/messages` | **WALL 1. Reading is fine. Sending a message to a client needs explicit coach approval.** |
| Media | `/v1/media` |  |
| Automations | `/v1/automations` | **OUTWARD. Running an automation can email or message a client. Needs explicit coach approval.** |
| API access keys (this doc's own auth) | `/v1/api-keys` | **NEVER call from this CLI. See below.** |

For anything not listed here (webhooks, integrations, admin-only routes, etc.), search
`GET /v1/openapi.json` rather than guessing a path.

## Never mint an API key from this CLI

`POST /v1/api-keys` returns the **raw `pk_live_` key in the response body**, and it is the only
time that value is ever shown. This CLI prints response bodies verbatim, so calling that route
would print a live, full-access, non-expiring credential for the coach's whole account straight
into the conversation, where it lands in transcripts and logs.

Do not call it. There is no task that justifies it.

If the coach needs a key, they create one themselves in the web app under Integrations, then API
keys, which only the account owner can reach. The same applies to `GET /v1/api-keys` and
`DELETE /v1/api-keys/:id`: key management belongs in the web app, not here.

## Rate limits

Requests carrying a `pk_live_` key are rate limited **per key**:

- **1000 requests per minute**
- **10000 requests per hour**

Breaching either returns **429** with a `Retry-After` header, in the usual
`{ success, message, data }` envelope. Browser sessions and MCP traffic are not affected by this
limiter.

This matters most for the bulk reads that are the main reason to be here. A roster-wide sweep can
reach the per-hour ceiling. If you get a 429:

- Stop. Do not retry in a tight loop.
- Read `Retry-After` and tell the coach how long the wait is.
- **Never report a partial sweep as complete.** Say exactly how much you covered. A summary that
  quietly omits the rows you could not fetch reads as authoritative and is worse than no summary.
