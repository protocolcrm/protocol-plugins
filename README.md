# Protocol CRM for Claude

Run your Protocol CRM from inside Claude: review a client, build a program or nutrition plan,
manage scheduling, and clear your inbox, all in plain conversation.

## Install

In Claude, type these two lines:

```
/plugin marketplace add dejankeri/protocol-claude-plugin
/plugin install protocol-crm@protocol
```

That's it. Claude connects to Protocol on its own the first time you use it, there's no
separate setup step.

**Already connected Protocol yourself?** If you previously added a "ProtocolCRM" connector by
hand (through claude.ai's connector settings), remove that one first, before or after
installing this plugin. Otherwise the same Protocol connection ends up registered twice under
two different names, and Claude won't know which one to use.

## Connecting

The first time you ask Claude to do something in Protocol, it opens a sign-in screen for
`api.protocolcrm.com`. Sign in and approve the connection, the same way you'd approve any app
asking for calendar or email access, and you're connected. As part of approving, you also pick an
access level: **Read & write** is the default and right for most coaches; pick **Send** only if
you also want Claude to be able to fire appointment reminders and run automations on its own,
since those two specifically need the higher level. There's no password or API key to copy for
normal, everyday use.

## Turn on auto-update

This matters: plugins from a third-party marketplace like this one do **not** update
themselves by default. If you skip this step, you'll stay on today's version forever, even
after fixes and new skills ship.

To turn it on:

1. Type `/plugin`.
2. Choose **Marketplaces**.
3. Select **protocol** from the list.
4. Choose **Enable auto-update**.

Takes about ten seconds, and you only do it once.

## What it does day to day

- Review a client, catch up before a session, or run a weekly check-in
- Onboard a new client and fill in their intake
- Build and assign training programs, workouts, and nutrition plans
- Manage your schedule and appointments
- Triage your inbox and tasks

## Optional: direct API access (Claude Code only)

Most coaches never need this. It's for two things: pulling a lot of data at once, for example
one client's full progress history instead of dozens of one-off lookups, and the rare request
the normal connection above just can't do.

To turn it on:

1. In Protocol, go to your profile menu, then **Integrations**, then **API keys**, and create a
   key. It starts with `pk_live_`.
2. Set it as an environment variable named `PROTOCOL_API_KEY` wherever you run Claude Code.

**This page is owner-only.** If you're a coach on someone else's team rather than the account
owner, you won't see Integrations at all, that's expected, not a bug. Ask the account owner to
create a key and pass it to you instead.

Two things to know before you set this up:

- This key is **full access to your account**, the same as logging in yourself. It is not
  limited the way the normal connection above is. Treat it like a password.
- It only works in **Claude Code**, the terminal app. Claude Desktop and the Claude web app
  have no terminal to run it from, so this option isn't available there.

If you're not sure you need this, you probably don't. The normal connection covers everyday
coaching work on its own.

## What it will not do without asking you

- **Message or chat with your clients.** It can read conversations, but sending is always your
  call, it drafts a message and lets you send or approve it.
- **Touch billing.** No charges, refunds, subscriptions, or invoices.
- **Permanently delete anything.** It prefers a reversible action, like cancel or archive.
- **Trigger Protocol's own AI generation.** It's the AI operating your account already, it
  won't turn around and invoke another one on Protocol's side.

## Learn more

[https://help.protocolcrm.com/ai-agent-claude-plugin](https://help.protocolcrm.com/ai-agent-claude-plugin)
