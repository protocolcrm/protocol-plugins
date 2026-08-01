# Protocol CRM for ChatGPT

Run your Protocol CRM from inside ChatGPT: review a client, build a program or nutrition plan,
manage scheduling, and clear your inbox, all in plain conversation.

## What it does day to day

- Review a client, catch up before a session, or run a weekly check-in
- Onboard a new client and fill in their intake
- Build and assign training programs, workouts, and nutrition plans
- Manage your schedule and appointments
- Triage your inbox and tasks

## Connecting today

This pack is not yet listed in OpenAI's plugin directory, so setting it up takes two steps rather
than one line. They do different jobs, and you want both.

### Step 1: connect Protocol

This gives ChatGPT the Protocol actions themselves, the same MCP server the Claude plugin uses.

1. In ChatGPT, open **Settings**, then **Connectors**, and turn on **Developer mode**.
2. Choose **Add custom connector**.
3. For the server URL, paste `https://api.protocolcrm.com/mcp`.
4. Sign in to Protocol when prompted and approve the connection.

There's no API key to copy for this, sign-in and approval are all it takes.

### Step 2: add the skills

This gives ChatGPT the coaching recipes: how to ground a program in a client's real profile, how
assignment avoids overwriting your template, what a realistic portion size looks like.

1. Download
   [`protocol-all-skills.zip`](https://github.com/dejankeri/protocol-claude-plugin/releases/latest/download/protocol-all-skills.zip).
   If you would rather add them one at a time, each skill is a separate download on the
   [releases page](https://github.com/dejankeri/protocol-claude-plugin/releases/latest).
2. In ChatGPT, go to **Plugins**, then **Skills**.
3. Choose **Create**, then **Upload from your computer**, and upload what you downloaded.
   ChatGPT scans each skill before making it available.

The readable source for all nine sits in the `skills/` directory next to this file, if you want
to see exactly what you are giving ChatGPT before you upload it. The downloads are built from it.

Upload `protocol-reference` first if the interface lets you choose an order. Every other skill
defers to it for exact parameter names, and a wrong parameter name loses data silently.

**Step 1 without step 2 still works**, and this is worth understanding rather than skipping.
ChatGPT will be able to reach your account and call every action it has access to. What it will
not have is the accumulated knowledge of how a coach uses them, so it will reason from scratch
each time instead of from a tested recipe. Expect more round trips and more of your own
correction.

## The consent screen and access levels

When you approve the connection, Protocol shows you an access level to choose, and this choice
is what actually gates what ChatGPT can do to your account, not anything in this pack.

- **Read and write** is the default, and it's right for most coaches. It covers everything on
  the list above: reviewing clients, building and assigning programs, nutrition, scheduling,
  and your inbox.
- **Send** is opt-in, on top of read and write. It exists for exactly three actions, all of them
  reaching a real client: firing an appointment reminder right now, arming a recurring reminder
  (which can fire within minutes if its start time is already due), and running an automation on
  demand. Pick this level only if you want the assistant able to trigger one of those three
  without you doing it yourself.

## An absent verb is normal, not a bug

At the **read** access level, the write actions aren't refused, they're simply not in the tool
list at all. If you ask the assistant to update a client or build a program while connected at
read only, it will tell you it can't, the same way it would tell you about any missing
capability, not throw an error.

That's expected, and it's not a sign that something is broken or missing from the product. The
fix is to reconnect at a higher access level (read and write, or send, whichever the task
needs), not to look for a workaround. In particular, it is never the right move to reach for a
direct Protocol API key to route around it. A key like that has no access level at all, it's
full account access, and using one to get past a level you (or your coach) deliberately chose
defeats the one real control you set at the consent screen. If ChatGPT ever suggests that path,
don't take it, reconnect at the level you actually need instead.

## What it will not do without asking you

- **Message or chat with your clients.** It can read conversations, but sending is always your
  call, it drafts a message and lets you send or approve it.
- **Touch billing.** No charges, refunds, subscriptions, or invoices.
- **Permanently delete anything.** It prefers a reversible action, like cancel or archive.
- **Trigger Protocol's own AI generation.** It's the AI operating your account already, it
  won't turn around and invoke another one on Protocol's side.

## Learn more

[https://help.protocolcrm.com/ai-agent-claude-plugin](https://help.protocolcrm.com/ai-agent-claude-plugin)
