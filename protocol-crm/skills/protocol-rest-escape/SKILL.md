---
name: protocol-rest-escape
description: Use when no MCP verb or find/get kind covers what the operator needs, or a real analysis would take dozens of MCP round trips - calls Protocol's live REST API directly with a full-access pk_live_ key, so know exactly what that buys and what it does not guard.
---

# protocol-rest-escape - full account access, on a shell, with no tier check

This skill hands you a `pk_live_` key that authenticates as the coach with full permissions on
every REST route, with no scope or tier attached to it at all. That is a genuine difference from
your MCP connection, which carries a real tier the coach chose at consent (`read`, `write`, or
`send`) and which the platform genuinely enforces on the three send-gated actions
(`schedule action=send_reminder`, `schedule action=reminder`, `manage_automations action=run`) -
see [`../protocol-reference/references/guardrails.md`](../protocol-reference/references/guardrails.md).
A REST key has none of that: whatever the coach's account can do, this key can do,
unconditionally. It can message real clients, move money, and permanently delete data, and neither
the four walls nor the tier system stop it. The only thing standing between this skill and a
destructive action on a real person's business is the judgment you apply below.

## Never use this to route around a tier refusal, or a verb the tier hid from you

If an MCP call was refused with `PermissionDeniedError` because it needed a tier this connection
does not have, that is the coach's own tier choice holding, not a bug and not a missing capability.
The same is true when a verb never shows up in your MCP tool list at all: the list itself is
tier-filtered (see `guardrails.md`), so on a `read`-only connection every `write` verb, including
`manage_client`, `record_progress`, `schedule`, `manage_automations`, and `report_to_developers`, is
simply absent, not broken. Either shape means the same thing: the coach chose a lower tier than the
task needs. Reaching for this skill because REST has no tier and would let the call through is the
worst possible response to either one: it deliberately overrides the access level the coach set at
consent. Tell the coach what was refused, or what verb was missing and why, and let them decide
whether to reconnect the plugin at a higher tier - never decide it for them by falling back to a
REST key mid-task, and never treat a `read`-only connection's missing write verbs as the "no MCP
verb covers this" case below - that case is for genuinely unsurfaced reads (see "Reads not
surfaced at all" in `guardrails.md`), not for writes a lower tier deliberately hides.

## Requires a shell

This skill runs a `node` command against a CLI file on disk. Claude Desktop, Claude web, and
Cowork do not give you a shell, so this skill **cannot execute in those surfaces at all**. If
you are running in one of them and the task needs this escape hatch: say plainly that the task
needs direct REST access, name what that access would have bought (for example, "a bulk
progress-entries pull with measurements included" or "reading a route the MCP tools don't
cover"), and stop there. Do not print `curl` or `protocol` commands into a chat that cannot run
them, and do not quietly fall back to a partial MCP-only answer while implying the full analysis
was done.

## When to use

- No MCP verb or `find`/`get` kind covers the request. Check the "Reads not surfaced at all"
  section in
  [`../protocol-reference/references/guardrails.md`](../protocol-reference/references/guardrails.md#reads-not-surfaced-at-all)
  first - a gap may be intentional, not an oversight.
- A bulk read that would otherwise be dozens of MCP round trips.

## The bulk-read case

This is the most common legitimate reason to reach for REST instead of MCP. `find` returns list
rows without measurements, so a real trend (a client's weight across four years of check-ins, a
whole roster's progress history) needs one `get` per entry: dozens of calls, sometimes hundreds.
The REST list endpoint returns the same records with `measurements` already included, paginated:

```
GET /v1/progress-entries?userId=<clientId>&page=1&pageSize=100
```

**This endpoint is 1-indexed:** page 1 is the first page, not page 0. Starting the `page` param at
zero does not error - it silently clamps to page 1 - so a naive sweep beginning at zero refetches
the first page twice and never reaches the last one, then reports the trend as complete when it is
actually missing the tail. One call per hundred entries instead of one per entry, and `total` in
the response tells you whether you actually have all of it, which the MCP reads never confirm.

## Setup

Set `PROTOCOL_API_KEY` in your environment to a `pk_live_` key from the coach's Protocol
account: Integrations, API keys. This is **not** the tiered `pk_` agent key from the AI Agent
page - that key authenticates MCP connections only. An agent key will not authenticate here; the
CLI will fail with a 401.

Integrations is **owner-only**: it exists only for the account owner, not for an invited team
coach. If the operator you're working with can't find it, they aren't the owner - tell them
plainly and have them ask whoever is to create the key and pass it along, rather than looking
for a menu that will never appear for them.

`PROTOCOL_API_URL` is optional and defaults to `https://api.protocolcrm.com`.

## Invocation

```
node "${CLAUDE_PLUGIN_ROOT}/skills/protocol-rest-escape/bin/protocol" GET /v1/users --query role=CLIENT
node "${CLAUDE_PLUGIN_ROOT}/skills/protocol-rest-escape/bin/protocol" whoami
```

`${CLAUDE_PLUGIN_ROOT}` is expected to be the installed plugin's root directory, but this is
**unconfirmed for a SKILL.md body specifically** - flag it rather than trust it blindly. The
evidence for it is real but indirect: other official plugins on this machine use it in
`plugin.json` hook commands and in slash-command bodies, and one of them documents it as the way
to reference plugin files after install. None of that evidence is a SKILL.md body using it the
way this line does, and it is not present in this shell's environment right now. Try the command
as written first. If it fails because the variable is empty or unset, fall back to a path that
does not depend on knowing the plugin root at all: resolve the CLI relative to this file
(`./bin/protocol` from the directory this SKILL.md lives in), or, if that directory is not known
either, search for it: `find ~/.claude/plugins -path '*protocol-rest-escape/bin/protocol' 2>/dev/null`.

A later task settles this empirically against a real plugin install; until then, treat the
`${CLAUDE_PLUGIN_ROOT}` form as the first thing to try, not a guarantee.

## The auth model

This key authenticates as the coach with full access to every route: the same permissions as
their own login session. There is no read-only or resource-scoped variant of this key. A call
this CLI can make is a call the coach's own dashboard login could make.

## `--confirm` is friction, not a boundary

GET, HEAD, and OPTIONS run freely. Every mutating method (POST, PUT, PATCH, DELETE) requires
`--confirm`, or the CLI refuses and prints the request it would have sent. This is deliberate
friction to make writes intentional, a checkpoint to get the coach's approval before a live
write. It is **not** a security boundary: you (or anyone reading this transcript) hold the raw
key and could bypass the CLI entirely with `curl` or a raw `fetch`. Treat `--confirm` as a pause
to get approval, never as something that is stopping a bad call from happening.

## The four walls, as policy

Four actions need the coach's explicit approval before you take them, exactly as in
[`../protocol-reference/references/guardrails.md`](../protocol-reference/references/guardrails.md).
These are policy on the MCP surface too, since there is simply no verb for messaging, billing,
deleting, or generating at any tier - that absence has nothing to do with the tier system covered
above, which genuinely is enforced. The difference here is only that this key can also reach the
underlying REST route directly, so the same wall-policy has to hold without even a narrower verb
surface to lean on:

1. No messaging or chatting with clients.
2. No billing: charges, refunds, subscriptions, invoices, checkout.
3. No hard deletes. Prefer a reversible action (cancel, archive, deactivate) if one exists.
4. No invoking Protocol's own AI generation.

Every one of these routes is reachable with this key. Nothing rejects the call. The wall is you
choosing not to make it without the coach saying yes first.

## Never mint an API key from here

`POST /v1/api-keys` returns the **raw `pk_live_` key in its response body**, and that is the only
time the value is ever shown. This CLI prints response bodies verbatim, so calling that route
prints a live, full-access, non-expiring credential for the coach's entire account into the
conversation, where it lands in transcripts and logs.

Do not call it. No task justifies it. The same goes for `GET /v1/api-keys` and
`DELETE /v1/api-keys/:id`: key management belongs in the web app.

If the coach needs a key, they create it themselves under Integrations, then API keys, which only
the account owner can reach.

## Never work around a wall silently

If the coach asks for something that one of the four walls covers, and you use the REST API to
do it, say so before you do it, not after. Never quietly reach for REST because the MCP verb
surface happens to omit something, and never let a coach's request slide past a wall because the
platform did not stop it. A denied or withheld approval is the coach saying no; stop there.

## Endpoint discovery

`GET /v1/openapi.json` is the source of truth for exact routes, params, and schemas. Read it
rather than guessing a path, especially before a write.

Every response is wrapped `{ "success": true, "message": "...", "data": ... }`. List endpoints
nest `{ "items": [...], "total", "page", "pageSize", "totalPages" }` inside `data`. Page indexing
is **not** uniform across controllers (some start at 0, some at 1); verify against the spec for
the specific endpoint rather than assuming.

Full route map and controller references:
[`references/api-reference.md`](references/api-reference.md).
