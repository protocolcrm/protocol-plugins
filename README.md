# Protocol CRM for your AI assistant

Run your Protocol CRM from inside your AI assistant: review a client, build a program or
nutrition plan, manage scheduling, and clear your inbox, all in plain conversation.

This repository holds two packs, one per assistant. Both connect to the same Protocol server and
carry the same nine coaching skills, so the assistant behaves the same way whichever one you use.

| Assistant | Where | How you install it |
|---|---|---|
| **Claude** | [`protocol-crm/`](./protocol-crm/) | Two lines in Claude. See below. |
| **ChatGPT** | [`openai/`](./openai/) | Add a connector, then upload the skills. See [`openai/README.md`](./openai/README.md). |

**About the name.** This repo is called `protocol-claude-plugin` because Claude was the first
assistant it supported. It is no longer Claude-only. The name stays for now so that the install
line coaches already have keeps working, and renaming it would break that line for everyone who
has it saved.

---

# Claude

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
you also want Claude to be able to reach a client on its own. Three actions need that level:
firing an appointment reminder now, arming a recurring one (which fires within minutes if its
start time is now or past), and running an automation on demand. There's no password or API key to copy for
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
- It only works in a terminal, so **Claude Code** rather than Claude Desktop or the Claude web
  app, which have no shell to run it from.

If you're not sure you need this, you probably don't. The normal connection covers everyday
coaching work on its own.

---

# ChatGPT

Not yet listed in OpenAI's plugin directory, so for now it takes two steps rather than one line.

1. **Connect Protocol.** In ChatGPT, open Settings, then Connectors, turn on Developer mode, and
   add a custom connector pointing at `https://api.protocolcrm.com/mcp`. Sign in and approve.
2. **Add the skills.** Go to Plugins, then Skills, then Create, then Upload, and upload the nine
   skills from [`openai/skills/`](./openai/skills/).

Step 1 alone gives ChatGPT the Protocol actions. Step 2 is what gives it the coaching recipes
that make those actions produce work a coach would actually send a client.

Full instructions, including the access levels, are in [`openai/README.md`](./openai/README.md).

---

# True on every assistant

## What it does day to day

- Review a client, catch up before a session, or run a weekly check-in
- Onboard a new client and fill in their intake
- Build and assign training programs, workouts, and nutrition plans
- Manage your schedule and appointments
- Triage your inbox and tasks

## The access level is the real control

Whichever assistant you use, what it can do to your account is decided by the access level you
pick when you sign in, not by anything in this repo. Read and write is the default. Send is
opt-in and covers only the three actions above that reach a client directly.

At the read level, the write actions are not refused, they are simply absent from the
assistant's list of what it can do. If you ask for a program build on a read-only connection, the
assistant will tell you it can't, which reads like a missing feature and is not one. Reconnect at
a higher access level. Never work around it with a Protocol API key: a key carries no access
level at all, it is full account access, and using one defeats the only control you actually set.

## What it will not do without asking you

- **Message or chat with your clients.** It can read conversations, but sending is always your
  call, it drafts a message and lets you send or approve it.
- **Touch billing.** No charges, refunds, subscriptions, or invoices.
- **Permanently delete anything.** It prefers a reversible action, like cancel or archive.
- **Trigger Protocol's own AI generation.** It's the AI operating your account already, it
  won't turn around and invoke another one on Protocol's side.

## Learn more

[https://help.protocolcrm.com/ai-agent-claude-plugin](https://help.protocolcrm.com/ai-agent-claude-plugin)

---

**For maintainers.** The nine skills are authored once in the monorepo and shared by both packs,
so `protocol-crm/skills/` and `openai/skills/` are two renderings of one source. Edit them in the
monorepo at `packs/skills/`, never here: this repository is a published mirror and is overwritten
on every publish.
