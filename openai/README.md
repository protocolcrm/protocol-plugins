# Protocol CRM for ChatGPT

Run your Protocol CRM from inside ChatGPT: review a client, build a program or nutrition plan,
manage scheduling, and clear your inbox, all in plain conversation.

## What it does day to day

- Review a client, catch up before a session, or run a weekly check-in
- Onboard a new client and fill in their intake
- Build and assign training programs, workouts, and nutrition plans
- Manage your schedule and appointments
- Triage your inbox and tasks

## What is in the plugin

One package with two parts, and you want both:

- **The Protocol connection** (an MCP server at `https://api.protocolcrm.com/mcp`). This is what
  lets ChatGPT reach your account and take actions in it. You sign in to Protocol to connect it;
  there is no API key to copy.
- **Nine coaching skills.** These are the recipes: how to ground a program in a client's real
  profile, how assignment avoids overwriting your template, what a realistic portion size looks
  like. The readable source for all nine sits in the `skills/` directory next to this file.

## Install it

### Once Protocol is listed in the plugin directory

1. In ChatGPT, open **Plugins** in the sidebar.
2. Search for **Protocol CRM** and open it.
3. Choose **Install**, then **Connect**.
4. Sign in to Protocol when prompted.
5. Pick an access level on Protocol's consent screen (see below) and approve.

### Until then: upload the plugin file

Protocol is not in the directory yet, so early coaches install the same plugin from a file.

1. Download the Protocol plugin file (a `.zip`) from
   [help.protocolcrm.com/ai-agent-chatgpt](https://help.protocolcrm.com/ai-agent-chatgpt). Keep it
   zipped; do not unzip it.
2. Go to [chatgpt.com/plugins](https://chatgpt.com/plugins).
3. Choose **Add**, then **Upload plugin archive**, and pick the file you downloaded.
4. Open the plugin and choose **Connect**.
5. Sign in to Protocol when prompted.
6. Pick an access level on Protocol's consent screen (see below) and approve.

This installs the connection and all nine skills together. You do not need to upload skills
separately.

### Fallback: connect without the skills

If **Upload plugin archive** is not offered on your account, you can still connect Protocol
itself. You get every action your access level allows, but not the coaching skills, so ChatGPT
reasons from scratch each time instead of from a tested recipe. Expect more round trips and more
of your own correction.

1. Go to [chatgpt.com/plugins](https://chatgpt.com/plugins).
2. Choose **Add** (the **+** button), then **Add custom MCP server**.
3. Give it a name, for example `Protocol`.
4. For the server URL, paste `https://api.protocolcrm.com/mcp`.
5. For authentication, choose **OAuth**. Leave the client ID and client secret blank; ChatGPT
   registers itself with Protocol automatically.
6. Read the warning, choose **I understand and want to continue**, then **Create as a plugin**.
7. Sign in to Protocol when prompted and pick an access level.

### Which ChatGPT plans can make changes

This is OpenAI's side, not Protocol's, and it is still moving. As OpenAI documents it today,
full MCP support including write actions (updating a client, building a program) is for
**Business, Enterprise, and Edu** workspaces, and **Pro** can connect a custom MCP server for
reading only. Whether a **Plus** account can run Protocol's write actions through an uploaded
plugin has not been confirmed yet; we are testing it and will update this page. In a Business or
Enterprise workspace, your admin may also need to allow uploading plugins or creating plugins
with MCP servers.

If ChatGPT can read your clients but tells you it cannot change anything, check the plan first,
then the access level you picked on Protocol's consent screen.

## The consent screen and access levels

When you approve the connection, Protocol shows you an access level to choose, and this choice
is what actually gates what ChatGPT can do to your account, not anything in this plugin.

- **Read and write** is the default, and it's right for most coaches. It covers everything on
  the list above: reviewing clients, building and assigning programs, nutrition, scheduling,
  and your inbox.
- **Send** is opt-in, on top of read and write. It exists for exactly three actions, all of them
  reaching a real client: firing an appointment reminder right now, arming a recurring reminder
  (which can fire within minutes if its start time is already due), and running an automation on
  demand. Pick this level only if you want the assistant able to trigger one of those three
  without you doing it yourself.

## An absent action is normal, not a bug

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

[https://help.protocolcrm.com/ai-agent-chatgpt](https://help.protocolcrm.com/ai-agent-chatgpt)
