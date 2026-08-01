# Canonical external sources

These are authoritative and kept current by rule — prefer them for coach-workflow and endpoint
detail; this pack carries the agent-operational depth (verbs, params, footguns, recipes).

## help.protocolcrm.com — the coach's guide

The public help center is the single source of truth for **how coaches use Protocol**. Each
section is a top-level area of the public help site at https://help.protocolcrm.com. Maintain a reading
relationship with the sections relevant to your task.

| Section | Purpose |
|---------|---------|
| **Getting started** | New to Protocol CRM? Start here — what it is, how to set up your business, and the core ideas that make everything else click. |
| **Clients** | Your roster and each athlete's profile — contact info, fitness and health profiles, progress, biometrics, labs, assigned plans, billing, and chat. |
| **Vault** | Your reusable library — training and exercises, nutrition templates, media, multi-phase programs, forms and check-ins, and automations. Build once, assign to anyone. |
| **Workouts** | Build standard, simple, or video-guided workouts from your exercise library — sections, supersets, sets and reps with an advanced intensity builder — then watch clients log them on mobile and review every session from your dashboard. |
| **Forms & reports** | Build forms and questionnaires, collect onboarding intake and weekly progress check-ins, watch the numbers and photos build up in each client's profile, and turn a check-in into a progress report — written by you or drafted by AI — that you approve before it reaches the client. |
| **Planning** | Your day-to-day operations — calendar and appointments, tasks, a public booking page, and meeting transcripts. |
| **Messaging** | Talk to your clients in real time — rich-formatted messages, photos and voice notes, @mentions, edits, read receipts, and announcement-only groups. Web and mobile, fully in sync. |
| **Shop & billing** | Sell to your clients and get paid — products and packages, subscriptions and renewals, purchases and invoices, and checkout settings. |
| **Your app** | The branded mobile app your clients use — preview it and configure its branding so it looks like yours, not ours. |
| **AI agent** | **Deep read for this pack:** Connect your own AI assistant — Claude or ChatGPT — to your Protocol account and run your coaching by chat. Build and assign programs, analyse your roster, and get plain-language rundowns. Changes happen live, bounded by the access you grant when you connect — and you can disconnect anytime. See `ai-agent/mcp-reference` for the served verb list and `ai-agent/approvals-and-safety` for safety guardrails. |
| **Integrations** | Connect your clients' wearables and health apps — Apple Health, Health Connect (Android) and WHOOP — so their steps, sleep, heart-rate and recovery flow straight into their profile. You review it all in one place. |
| **Account & settings** | Manage your own account — profile, organization, team, your Protocol subscription, and notifications. |
| **FAQ** | Quick answers to the questions coaches actually ask — the non-obvious ones about logging in, onboarding, programs, nutrition, check-ins, billing, and the app. Grouped by area, most-asked first. |
| **For clients & athletes** | Resources for your athletes — coming soon. |
| **Release notes** | What's new in Protocol CRM, by release — new features, improvements, and fixes, newest first. Many entries link straight to the guide for the feature. |
| **API reference** | **Deep read for integration work:** For developers — authenticate with an API key and call the Protocol REST API to build your own integrations. See `api/api-keys` for key setup and scoping. |

### Key pages for this pack

- **`ai-agent/mcp-reference`**: the human-readable counterpart to the MCP surface, starting at `./surface-core.md` (split across the four `surface-*.md` files in this pack). Lists all verbs the agent can call, parameters, constraints, and examples.
- **`ai-agent/approvals-and-safety`**: plain-language safety guardrails. Same rules as `./guardrails.md`, written for a coach.
- **`api/api-keys`** — API key management (create, scope, rotate, revoke) and tier assignment for external integrations.

## api.protocolcrm.com — the REST API reference

The live REST API is served at `https://api.protocolcrm.com/docs` (Swagger UI) and the full
OpenAPI schema is available at:

```
GET https://api.protocolcrm.com/v1/openapi.json
```

Use the schema to verify endpoint signatures and response shapes. Endpoint behavior details and
coach workflows live in the help center sections above.
