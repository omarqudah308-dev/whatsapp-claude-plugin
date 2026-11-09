# WhatsApp Channel for Claude Code

Drive your Claude Code session from WhatsApp. Choose a **linked device** for a
personal account or the **official WhatsApp Cloud API** for business DMs.

The default Baileys provider connects like WhatsApp Web: pair your phone, then
Claude replies from your own number. Existing installations keep this provider.
The optional Cloud API provider uses Meta's business API and a public HTTPS
webhook, with its own credentials, messaging policies and applicable fees.
Both providers feed the same MCP channel and access gate.

[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blue)](https://code.claude.com/docs/en/plugins)
[![MCP Server](https://img.shields.io/badge/MCP-Server-green)](https://modelcontextprotocol.io)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

> **Community project.** This plugin is not made or endorsed by Anthropic or Meta.
> Its Baileys provider uses an unofficial linked-device protocol, which may
> conflict with WhatsApp's terms and put the number at risk. The optional Cloud
> provider uses official API transport; confirm your use is permitted under
> Meta's business and AI-provider rules before enabling it.

## Installation

```sh
claude plugin marketplace add Rich627/whatsapp-claude-plugin
claude plugin install whatsapp-channel@whatsapp-claude-plugin
claude --dangerously-load-development-channels plugin:whatsapp-channel@whatsapp-claude-plugin
```

The `--dangerously-load-development-channels` flag matters: it registers the plugin as a **channel**, so an inbound WhatsApp message wakes your session immediately. Without it the tools still load, but nothing wakes the session when messages arrive — they sit unanswered until you (or a [watchdog](./scripts/watchdog.sh)) prompt Claude to check. `--channels` does not accept this plugin yet (it is not on the research-preview allowlist), so the development flag is currently the only way.

> **Know what this flag does.** Claude Code warns against loading downloaded channels this way: a loaded channel can put text from other people straight into your session. Read this plugin's code before you use it, keep the [allowlist](./ACCESS.md) to people you trust, and keep permission prompts on. Never combine it with `--dangerously-skip-permissions`: with prompts off, anyone in an allowlisted chat could get Claude to run commands on your computer.

Inside the session, choose your connection:

```text
/whatsapp-channel:setup
/whatsapp-channel:configure provider
```

The wizard runs in your own terminal so Cloud credentials never pass through
chat. Or run `bun scripts/provider.ts choose` from the plugin directory.

| Connection                         | Setup                                                                       | Main limits                                                                            |
| ---------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Linked device (`baileys`, default) | Pair your phone; no Meta API credentials                                    | Unofficial protocol; existing personal/group behavior                                  |
| Official (`cloud-api`)             | Meta business number, token, app secret, verify token, public HTTPS webhook | DMs only in this implementation; service window/templates and Meta fees/policies apply |

For linked-device pairing, save your number with
`/whatsapp-channel:configure <phone>` (country code + number, no `+`). Enter the
printed pairing code in WhatsApp → Settings → Linked Devices → Link a Device →
**Link with phone number instead**, or scan the QR.

For official business DMs, follow [Cloud API setup](./docs/cloud-api.md). This
provider does not yet support groups, message editing, phone history/address-book
import or coexistence synchronization. Add an explicitly allowed personal owner;
the business number is never auto-added, and the owner must be set explicitly with
`/whatsapp-channel:access set owner <jid>`. A non-empty key saved in
`~/.whatsapp-channel/.env` by the terminal tools wins over plugin `/config` values
for that key; remove it from `.env` to return it to plugin management.
Configuration readiness is not proof
that the public webhook works. Rates vary; consult
[Meta pricing](https://business.whatsapp.com/products/platform-pricing) and
[FAQ](https://whatsappbusiness.com/resources/faq/).

## Other MCP clients (Codex CLI, Gemini CLI, Cursor)

The server is a plain stdio MCP server, so any MCP client can run it. Two things are Claude Code specific and worth knowing before you start:

- **Inbound messages are not pushed.** Waking a session on an incoming message uses `notifications/claude/channel`, a Claude Code extension. MCP has no standard equivalent that reaches the model, and other clients drop unknown notifications silently. Elsewhere the plugin is poll-based: call `wait_for_messages` (parks up to 40s; the first call on a connection returns whatever is already unreplied, later calls only what arrived since) or `catch_up` / `unreplied`. Most tool results also carry a count of unreplied messages, so a client finds out there is traffic on its next call. Three do not: `unreplied` and `wait_for_messages`, which just returned those very messages, and `catch_up` with no arguments, which is that count already.
- **Setup is done from a terminal, not a slash command.** `/whatsapp-channel:access` and friends are Claude Code skills. Use `bun scripts/access.ts` instead (see [Access control from a terminal](#access-control-from-a-terminal)).

Register the server with an absolute path — `${CLAUDE_PLUGIN_ROOT}` is substituted by Claude Code only:

**Codex CLI** (`~/.codex/config.toml`)

```toml
[mcp_servers.whatsapp]
command = "bun"
args = ["run", "--cwd", "/absolute/path/to/whatsapp-channel", "start"]
startup_timeout_sec = 30   # default 10 is tight for a first Baileys connect
tool_timeout_sec = 120     # default 60; wait_for_messages parks for up to 40s
```

**Gemini CLI** (`~/.gemini/settings.json`)

```json
{
  "mcpServers": {
    "whatsapp": {
      "command": "bun",
      "args": ["run", "--cwd", "/absolute/path/to/whatsapp-channel", "start"],
      "timeout": 600000
    }
  }
}
```

**Cursor** (`~/.cursor/mcp.json` for all projects, `.cursor/mcp.json` for one)

```json
{
  "mcpServers": {
    "whatsapp": {
      "type": "stdio",
      "command": "bun",
      "args": ["run", "--cwd", "/absolute/path/to/whatsapp-channel", "start"]
    }
  }
}
```

Only one client at a time can hold a state directory’s singleton lock. With Baileys, two connections using the same linked-device auth would kick each other off. Use separate state directories for separate numbers. A second server does not fail silently — it stays up and serves a single `whatsapp_unavailable` tool naming the process that holds the connection.

## Access control from a terminal

Everything the access skill does, without Claude Code:

```sh
bun scripts/access.ts status                 # policy, allowlist, pending codes, groups
bun scripts/access.ts policy pairing         # open the door
bun scripts/access.ts pair <code>            # approve someone who messaged you
bun scripts/access.ts allow <jid>            # add directly
bun scripts/access.ts remove <jid>
bun scripts/access.ts group add <groupJid> [--mention] [--allow jid1,jid2]
bun scripts/access.ts set replyToMode first  # ackReaction, textChunkLimit, chunkMode, mentionPatterns
```

Approving always needs the specific code, even when only one pairing is waiting: anyone can create a pending entry just by messaging the account, so "approve the pending one" is exactly what a prompt-injected request looks like. For the same reason this is a terminal command and deliberately **not** an MCP tool, so nothing arriving over WhatsApp can reach it.

## Features

The list below describes the default linked-device provider. See the
[provider capability comparison](./docs/cloud-api.md#capabilities-in-this-plugins-first-cloud-provider)
for the initial Cloud API scope. Shared DM access, attachment processing,
transcription and context recovery also work with Cloud API.

- **Bidirectional messaging.** Send and receive from the session; long replies are chunked to WhatsApp's limits or sent as a document attachment past a configurable threshold.
- **Typing indicator.** The chat shows "typing…" while Claude works on its message, and "recording audio…" while a voice note uploads (linked device).
- **@-mentions.** `reply` can tag people so they actually get notified — ids are accepted as phone, LID, or full JID, and mentions attach only to the chunk that names them.
- **Full media support.** Photos, voice notes, video, documents, and stickers, in both directions.
- **Polls.** `send_poll` asks a chat a question with up to 12 options; every vote comes back to the session as a message (linked device).
- **Voice transcription.** Incoming voice notes are transcribed locally via mlx-whisper (see [setup](#voice-transcription-optional)); without the script they arrive as plain attachments.
- **Access control.** Pairing codes, allowlists, and per-group policies gate every inbound message — strangers never reach your session. Managed via `/whatsapp-channel:access` in Claude Code, or `bun scripts/access.ts` anywhere.
- **Per-group personalities.** Each group gets its own `config.md` with a custom personality and conversation memory.
- **Permission relay.** Approve or deny Claude's tool requests from WhatsApp with an emoji reaction (👍 / 👎).
- **Cron tasks.** A `## Cron Jobs` section in a group's `config.md` schedules recurring server-side tasks.
- **Context recovery.** After a restart, `catch_up` with no arguments says how many messages are waiting per chat - counts only, no message text - plus open tasks from `tasks.md`. Name a chat and it replays that room's recent two-way conversation, so a fresh session resumes mid-flight work without reading every chat it has.
- **Dual accounts.** Run personal and business numbers side by side with separate state and behaviors.
- **Self-diagnosis.** `/whatsapp-channel:doctor` checks the server process, device link, singleton lock, and config, then walks you through the fixes — no more guessing why replies stopped.

## How it works

```text
WhatsApp linked device <──Baileys────────> MCP Server <──stdio──> Claude Code
WhatsApp business DMs   <──Cloud API/HTTPS─> MCP Server
```

The server (a single Bun process) holds either the linked-device connection or the Cloud API webhook listener and forwards inbound messages after they pass the access gate. Claude acts through shared MCP tools such as `reply`, `react`, `download_attachment`, `status`, `unreplied` and `catch_up`. Provider-specific tools include linked-device group/edit operations, polls (`send_poll`), `set_presence`, read receipts (`mark_read`), and Cloud `send_template`. Runtime state lives in `~/.whatsapp-channel/` (or `WHATSAPP_STATE_DIR`), never in the repo.

Messages sent by Claude appear from your linked personal number or configured business number. Use a dedicated number if you want a distinct assistant identity.

## Voice transcription (optional)

One-time setup (Apple Silicon, mlx-whisper):

```bash
brew install ffmpeg                      # mlx-whisper uses it to decode audio
python3 -m venv ~/whisper-env
source ~/whisper-env/bin/activate
pip install mlx-whisper
cp scripts/whisper-transcribe.sh ~/whisper-transcribe.sh
chmod +x ~/whisper-transcribe.sh
~/whisper-transcribe.sh path/to/sample.ogg   # optional: test
```

The reference script uses `mlx-community/whisper-large-v3-turbo` — accurate, fast, multilingual. Swap the model in the script if you prefer a smaller one.

**Cloud alternative.** To skip the local setup, open `/plugin`, pick this plugin, and set **Voice transcription provider** to `groq` or `openai` along with that provider's API key. Keys are kept in your OS keychain, not in `settings.json`. Voice notes are then uploaded to `api.groq.com` or `api.openai.com` for transcription. The older `TRANSCRIPTION_PROVIDER`, `GROQ_API_KEY` and `OPENAI_API_KEY` environment variables still work.

## Troubleshooting

| Issue                               | Solution                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pairing code not showing            | Run `/whatsapp-channel:configure <phone>` first, then relaunch                                                                                                                                                                                                                                                                                                                                      |
| 440 disconnect error                | Only one connection per auth state allowed. Kill stale processes: `pkill -f "whatsapp.*server"`                                                                                                                                                                                                                                                                                                     |
| Session not waking on new messages  | Most common cause: launched without `--dangerously-load-development-channels plugin:whatsapp-channel@whatsapp-claude-plugin`. Tools work but inbound pushes are dropped (`Channel notifications skipped` in the MCP debug log) — relaunch with the flag.                                                                                                                                            |
| Messages not arriving               | Known Claude Code client bug ([#37933](https://github.com/anthropics/claude-code/issues/37933)). Server-side is correct, awaiting client fix.                                                                                                                                                                                                                                                       |
| Replies still send, nothing arrives | Send a **DM** to the connected number as well as a group message — a DM that lands while groups stay silent means the group sender-key path, not the connection. `~/.whatsapp-channel/diag.log` records an `inbound upsert` line for every batch WhatsApp delivers, so it distinguishes "never arrived" from "arrived and was dropped". Set `WHATSAPP_DIAG_DEBUG=1` for Baileys' full debug stream. |
| Auth expired                        | Run `/whatsapp-channel:configure reset-auth` and re-pair                                                                                                                                                                                                                                                                                                                                            |

## Documentation

Full documentation lives in [USAGE.md](./USAGE.md): [access control](./USAGE.md#access-control), the [tools exposed to the assistant](./USAGE.md#tools-exposed-to-the-assistant), [dual-account setup](./USAGE.md#dual-account-setup), [session conflicts](./USAGE.md#session-conflicts), and [resetting auth](./USAGE.md#resetting-auth).

## Contributing

Issues and pull requests are welcome — read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening one. Report security issues privately per [SECURITY.md](./SECURITY.md).

## Star History

<a href="https://www.star-history.com/?type=date&repos=Rich627%2Fwhatsapp-claude-plugin">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=Rich627/whatsapp-claude-plugin&type=date&theme=dark&legend=top-left&sealed_token=NrfP1Fv0z7ipQM961lFZJbXE76GS7paukclIhr6km37t0lJAzivyX0JUNQTkRaxa5lSpRCYmef3xvHaiUKCgBS0KbwpeIohfMOqur0ULPiTt2h2DWcUui1YJ2nux4W9Ug8u8D6CNl91ZYInSZCrrdNi5hydWjSLy89XtzYYM83F-mhgJI44lLZoxj7Na" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=Rich627/whatsapp-claude-plugin&type=date&legend=top-left&sealed_token=NrfP1Fv0z7ipQM961lFZJbXE76GS7paukclIhr6km37t0lJAzivyX0JUNQTkRaxa5lSpRCYmef3xvHaiUKCgBS0KbwpeIohfMOqur0ULPiTt2h2DWcUui1YJ2nux4W9Ug8u8D6CNl91ZYInSZCrrdNi5hydWjSLy89XtzYYM83F-mhgJI44lLZoxj7Na" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=Rich627/whatsapp-claude-plugin&type=date&legend=top-left&sealed_token=NrfP1Fv0z7ipQM961lFZJbXE76GS7paukclIhr6km37t0lJAzivyX0JUNQTkRaxa5lSpRCYmef3xvHaiUKCgBS0KbwpeIohfMOqur0ULPiTt2h2DWcUui1YJ2nux4W9Ug8u8D6CNl91ZYInSZCrrdNi5hydWjSLy89XtzYYM83F-mhgJI44lLZoxj7Na" />
 </picture>
</a>

## License

[Apache 2.0](./LICENSE) — Copyright 2025 Richie Liu


# WhatsApp

Connect WhatsApp to your Claude Code session via linked-device protocol.

The MCP server connects to WhatsApp as a linked device (like WhatsApp Web) and provides tools to Claude to reply, react, edit messages, and handle media. When someone messages the linked number, the server forwards the message to your Claude Code session.

> **Identity notice:** This plugin connects as a linked device to your existing WhatsApp account. Messages sent by Claude will appear as coming from your phone number — recipients cannot distinguish them from messages you send personally. If you need a separate bot identity, use a dedicated number (e.g. a second SIM or WhatsApp Business account) with the [dual-account setup](#dual-account-setup).

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. See [bun.sh](https://bun.sh/docs/installation) for install options.
- A WhatsApp account with an active phone number.

## Quick Setup

**1. Install the plugin.**

```
/plugin marketplace add Rich627/whatsapp-claude-plugin
/plugin install whatsapp-channel@whatsapp-claude-plugin
/exit
```

Restart to activate the plugin:

```sh
claude
```

**2. Configure your phone number.**

```
/whatsapp-channel:configure 886912345678
/exit
```

Use your WhatsApp phone number with country code, no leading `+`.

**3. Launch with the channel flag.**

```sh
claude --dangerously-load-development-channels plugin:whatsapp-channel@whatsapp-claude-plugin
```

The pairing code appears automatically in your session. On your phone:

1. Open WhatsApp > **Settings** > **Linked Devices** > **Link a Device**
2. Tap **Link with phone number instead**
3. Enter the pairing code

Once paired, your own number is **auto-added to the allowlist** and the policy is **auto-locked to allowlist mode**.

> `--dangerously-load-development-channels` is what lets an inbound message wake the session; `--channels` only accepts plugins on the research-preview allowlist, which this one is not. Claude Code warns against loading downloaded channels this way, because a channel puts other people's text into your session. Keep the allowlist to people you trust and keep permission prompts on.

**4. Add other contacts (optional).**

Have someone DM the linked number. Briefly flip to pairing mode:

```
/whatsapp-channel:access policy pairing
```

They'll receive a 6-character code. Approve in your Claude Code session:

```
/whatsapp-channel:access pair <code>
```

After pairing, the policy auto-locks back to `allowlist`.

**5. Add groups (optional).**

```
/whatsapp-channel:access group add <groupJid>
```

Each group gets its own personality config at `~/.whatsapp-channel/groups/<groupJid>/config.md`. Edit that file to customize how Claude behaves in each group. Conversation memory is auto-saved to `memory.md` in the same directory.

See [ACCESS.md](./ACCESS.md) for group options (`--mention`, `--allow`, `--roster`). Setting up several groups or contacts at once? Ask for `/whatsapp-channel:access review`, or run `bun scripts/access.ts wizard` yourself - see [Guided bulk setup](./ACCESS.md#guided-bulk-setup-wizard).

## Daily use

After initial setup, just run:

```sh
claude --dangerously-load-development-channels plugin:whatsapp-channel@whatsapp-claude-plugin
```

- `--dangerously-load-development-channels` — load this third-party channel plugin so inbound messages wake the session

Do not add `--dangerously-skip-permissions`. With permission prompts off, anyone in an allowlisted chat could get Claude to run commands on your computer. To skip prompts for the WhatsApp tools only, use the fine-grained allowlist below.

Auth is saved in `~/.whatsapp-channel/.baileys_auth/`. The session must stay open to receive messages — closing the session disconnects WhatsApp.

### Fine-grained permissions

To auto-allow only the WhatsApp tools, add them to your `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "mcp__plugin_whatsapp-channel_whatsapp__reply",
      "mcp__plugin_whatsapp-channel_whatsapp__react",
      "mcp__plugin_whatsapp-channel_whatsapp__status",
      "mcp__plugin_whatsapp-channel_whatsapp__download_attachment",
      "mcp__plugin_whatsapp-channel_whatsapp__edit_message",
      "mcp__plugin_whatsapp-channel_whatsapp__send_poll",
      "mcp__plugin_whatsapp-channel_whatsapp__set_presence",
      "mcp__plugin_whatsapp-channel_whatsapp__mark_read"
    ]
  }
}
```

### Permission relay

When Claude needs to run a tool that requires approval and no one is at the terminal, the request is forwarded to all allowlisted WhatsApp contacts. Reply `yes <code>` or `no <code>` from WhatsApp to approve or deny.

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, groups, mention detection, delivery config, skill commands, and the `access.json` schema.

## Tools exposed to the assistant

| Tool                  | Purpose                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `reply`               | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` (quote-reply), `files` (attachments), and `mentions` (names over raw numbers; `"all"` needs [roster access](./ACCESS.md#group-roster--all-mentions)).                                                                                                                                                                                                                |
| `send_poll`           | Send a poll (2-12 options, `multiple` for several picks). Each vote arrives as a `[Poll vote]` message with the tally. Linked device only; see [Polls](#polls).                                                                                                                                                                                                                                                                      |
| `react`               | Add an emoji reaction to a message by ID. Any emoji is supported.                                                                                                                                                                                                                                                                                                                                                                    |
| `set_presence`        | Show `composing` ("typing…") or `recording` in a chat until the next reply there, or clear it with `paused`. Linked device only; see [Typing indicator](#typing-indicator).                                                                                                                                                                                                                                                          |
| `download_attachment` | Download media from a received message. Returns the local file path.                                                                                                                                                                                                                                                                                                                                                                 |
| `edit_message`        | Edit a message the account previously sent.                                                                                                                                                                                                                                                                                                                                                                                          |
| `mark_read`           | Mark a received message read (blue ticks) by ID. Linked device only. `WHATSAPP_READ_RECEIPTS=1` does it for every delivered message ([ACCESS.md](./ACCESS.md#read-receipts)).                                                                                                                                                                                                                                                        |
| `status`              | Check connection state and get the pairing code if not yet paired.                                                                                                                                                                                                                                                                                                                                                                   |
| `unreplied`           | List received messages not yet replied to, with their chat_ids. Pass `chat_id` for one chat. Shows at most 100 per call (the newest), after that filter; `wait_for_messages` has the same ceiling.                                                                                                                                                                                                                                   |
| `wait_for_messages`   | Park up to 40s for messages this connection has not been handed yet. The first call returns whatever is already unreplied; later calls only what arrived since. Mainly for MCP clients other than Claude Code, which pushes messages into the session itself.                                                                                                                                                                        |
| `catch_up`            | Without `chat`: **counts only** - one line per chat that has something waiting, with an `@` where a mention-gated group addressed you, plus open items from `~/.whatsapp-channel/tasks.md`. No message text, no chat_id. Pass `chat` (a chat_id or part of a group/contact name) to read one chat: its recent two-way conversation, waiting messages first (retention: see "Your own replies" below), and the chat_id `reply` needs. |
| `list_groups`         | List the allowlisted groups the account is in, with JID and roster-grant state; other groups are not shown (add one with `/whatsapp-channel:access review`). Also refreshes the local group name cache the access wizard reads.                                                                                                                                                                                                      |
| `group_roster`        | List a group's members by saved name, or a masked number — never raw. Requires [roster access](./ACCESS.md#group-roster--all-mentions).                                                                                                                                                                                                                                                                                              |

### Your own replies

When you answer a chat from your phone instead of asking Claude to, the server logs that message
too — but only for chats already on the allowlist (a DM from someone in `allowFrom`, or a group
you have enabled). Anything you send to any other chat is discarded before its text is read: the text is never
read, logged or stored, and the drop itself is not recorded either — the drop path writes no
diagnostic line and no log line. The one diagnostic line per inbound batch that precedes it masks every number to its last four digits (a group keeps its identifying form; a legacy `<number>-<timestamp>@g.us` group has its creator's number masked) and carries no message id or text; Baileys' own logging (`info`+, or `debug` under `WHATSAPP_DIAG_DEBUG=1`) is not masked by us.

`catch_up` **with a `chat` argument** replays the last **5** messages from the other side and the last **5** of your own for that chat, merged in time order, so your own texts never crowd out what the room said. Messages still waiting for a reply take those inbound slots first, so whatever the counts list flagged is what you see - in a mention-gated group the @-mention that raised the count can no longer be pushed out of the window by ordinary chatter. If more are waiting than fit, the header says how many. Pass `limit` to widen both sides together (min 5, max 100) - it is deliberately symmetric, so a run of your own texts never leaves you without the same amount of context behind it, and it widens how many of your own hand-typed messages are replayed as well as theirs. If the name you pass matches more than one chat, you get the matching chats instead of any message text, so nothing is guessed for you - up to 8 of them, a group by its id and a DM by its number masked to the last four digits (a masked handle is accepted back as `chat`; if two rows render identically, the full jid is the way through). Session start (no `chat`) shows none of this - only how many are waiting per chat. In a mention-gated group it also shows what members said without addressing Claude (tagged "not addressed to Claude") - kept for context only, never routed or counted as unreplied; opt a group out of that with `group add <jid> --no-context` ([ACCESS.md](./ACCESS.md#mention-detection)) and nothing unaddressed is stored for it at all. Your own hand-typed text reads in
full for as long as the line is kept, the same as theirs, so a chat never comes back half-blank.
How long that is, is the one retention rule for message text: the line stays in
`~/.whatsapp-channel/messages.jsonl` for **7 days**, the same as every other
context line there (replies Claude sent for you, and a mention-gated group's unaddressed chatter).
An **unanswered** message addressed to Claude is kept exactly as long as everything else. It used to expire after 24 hours, which meant the one message you had not got to yet was the first thing to disappear. Set `WHATSAPP_MESSAGE_TTL_DAYS` to keep lines for a different number of days. It accepts **1 to 30**: anything that is not a positive number is ignored and the 7-day default stands, and anything outside that range is clamped into it — so `90` gives you 30, not 90. Either way the server says what it did in `diag.log`.
The ceiling is not red tape. The same horizon prunes `~/.whatsapp-channel/inbox/`, and that prune is the only thing stopping every photo and voice note you have ever received from staying on disk forever; the floor is there because the unit is days and a value like `0.05` would delete an attachment about an hour after it arrived — possibly one Claude was still reading.
The setting belongs to whichever terminal holds the singleton lock, because only that process prunes. With two terminals open and different values set, the lock holder's value wins, and it changes if that process exits and another is promoted.
Backfill goes exactly as far as WhatsApp's own offline queue: whatever was sent while no server
was connected is delivered on the next reconnect and logged then; anything older than that queue
is gone for good.

Separately, `~/.whatsapp-channel/sent.jsonl` (owner-only, pruned to 24 hours) records only the **id and timestamp** of the plugin's own sends — never text, number or chat — so a send WhatsApp re-delivers after a restart is not mistaken for a reply you typed.

One record outlives the horizon on purpose: `~/.whatsapp-channel/.aged-out-chats.json`. When the prune drops a message that was still waiting for a reply, the chat is noted there - as a truncated hash of the chat_id and a timestamp, never the id itself - so the session-start counts can say "N chats had something waiting that has since aged out" after you have been away longer than the horizon. Entries expire after a fixed **30 days**; deleting `messages.jsonl` to clear history does not clear them. Static mode (`WHATSAPP_ACCESS_MODE=static`) never writes the file.

## Typing indicator

On linked device, the chat shows "typing…" while the assistant works on a message from it. The server sends it as soon as an allowed message is picked up and re-sends it every 8 seconds (WhatsApp drops it after about 10), until the assistant replies to that chat or `WHATSAPP_TYPING_MAX_MINUTES` runs out, 10 minutes by default. While `reply` uploads a voice note the chat shows "recording audio…" instead. The `set_presence` tool sets either state, or clears it with `paused`, by hand.

Set `WHATSAPP_TYPING_MAX_MINUTES` in `~/.whatsapp-channel/.env` to change the cap: any number of minutes up to 60 (higher is clamped to 60). `0` turns the kept-up indicator off and leaves only the single "typing…" at pickup that earlier versions sent. A message the assistant never answers keeps "typing…" up until the cap, so keep it short if that happens often.

## Photos & Media

Inbound **photos** are downloaded eagerly to `~/.whatsapp-channel/inbox/` and the local path is included in the notification so the assistant can read it. A media message with no caption is shown in `catch_up` and `unreplied` as `[photo]`, `[voice]`, `[video]` or `[file]` rather than as an empty line.

Other media types (**voice notes, audio, video, documents, stickers**) are lazy — the notification includes an `attachment_file_id`. The assistant calls `download_attachment` to fetch the file on demand.

Outbound, `reply`'s `files` picks the message type from the extension: images send as photos, `.mp4`/`.mov`/`.avi` as video, `.mp3`/`.m4a` as audio, and anything else as a document. On linked device an `.ogg` or `.opus` file that contains Opus goes out as a **voice note** (push-to-talk, the green play button); an `.ogg` holding Vorbis stays a document, since WhatsApp voice notes have to be Opus. To turn other audio into a voice note: `ffmpeg -i in.wav -c:a libopus -b:a 32k -ac 1 note.ogg`. On Cloud API `.ogg` sends as plain audio, as before.

### Restricting which files can be attached

By default `reply` can attach any readable file except the channel's own state (credentials, logs) and a short blocklist (`~/.ssh`, `~/.aws`, `~/.gnupg`, files named `.env*`). That list is a speed bump: a secret stored anywhere else, under any other name, can still be sent, and a reply is only one prompt injection away from an allowlisted chat.

To make it a real boundary, set `WHATSAPP_ATTACH_DIRS` in the server's `.env` to the directories attachments may come from, separated by `:` (`;` on Windows, so `D:\share` keeps its drive letter). A leading `~` is your home directory; relative entries are ignored, and if none of the entries is usable only the inbox can be attached.

```bash
# ~/.whatsapp-channel/.env
WHATSAPP_ATTACH_DIRS=~/.whatsapp-channel/outbox:~/Pictures/share
```

With it set, only files received into `~/.whatsapp-channel/inbox/` or living under a listed directory can be attached; everything else is refused before it is read. The blocklist above still applies inside the listed directories. Unset or empty keeps the default behaviour.

## Polls

`send_poll` sends a WhatsApp poll to an allowlisted chat (linked device only): a question and 2 to 12 options, single choice unless `multiple` is true. Every vote, change of vote or withdrawal comes back to the assistant as an ordinary inbound message, for example `[Poll vote] Akash voted: "Yes" on "Ship it?"` followed by the current tally, with `poll_id` and `option` in its meta. A vote goes through the same access gate as a message from that chat, minus the mention requirement (a vote on our own poll is addressed to us), and is listed by `catch_up` and `unreplied` like any other message.

Votes are encrypted with a secret that only the poll's sender and its recipients hold. The server keeps that secret for every poll it sends, in `~/.whatsapp-channel/polls.json`, for 30 days (at most 200 polls). It cannot read votes on a poll it did not send (one created by hand on the phone, or sent before this version), or on a poll older than that.

## Dual-account setup

You can run two WhatsApp accounts simultaneously — for example, your personal number and a dedicated bot number (WhatsApp Business or a second SIM). Each account runs as a separate MCP server with its own auth, allowlist, and state directory.

**1. Set environment variables for each account.**

Create separate `.env` files:

```sh
# ~/.whatsapp-channel/personal/.env
WHATSAPP_PHONE_NUMBER=886912345678

# ~/.whatsapp-channel/business/.env
WHATSAPP_PHONE_NUMBER=886987654321
```

**2. Add both servers to your MCP config.**

In your project or user `.mcp.json`:

```json
{
  "mcpServers": {
    "whatsapp-personal": {
      "command": "bun",
      "args": [
        "run",
        "--cwd",
        "<plugin-path>",
        "--shell=bun",
        "--silent",
        "start"
      ],
      "env": {
        "WHATSAPP_STATE_DIR": "~/.whatsapp-channel/personal",
        "WHATSAPP_ACCOUNT_NAME": "personal"
      }
    },
    "whatsapp-bot": {
      "command": "bun",
      "args": [
        "run",
        "--cwd",
        "<plugin-path>",
        "--shell=bun",
        "--silent",
        "start"
      ],
      "env": {
        "WHATSAPP_STATE_DIR": "~/.whatsapp-channel/business",
        "WHATSAPP_ACCOUNT_NAME": "bot"
      }
    }
  }
}
```

Each account gets fully isolated state (auth, allowlist, groups, inbox). Claude sees tools from both accounts with different namespaces (e.g. `mcp__whatsapp-personal__reply` vs `mcp__whatsapp-bot__reply`) and inbound messages include an `account` field in the meta so Claude knows which account received the message.

**3. Pair each account separately.** Launch and follow the normal pairing flow for each.

## Session conflicts

WhatsApp allows only **one real connection per auth state** - a protocol limit, not something this plugin works around. What changed is what happens when a second `server.ts` process starts (a second Claude Code terminal, most commonly) while another is already connected.

**Both terminals stay usable.** The first process to start becomes the primary and holds the real WhatsApp connection. Every later process becomes a secondary: it relays its tool calls (`reply`, `react`, etc.) to the primary over a local, same-machine, token-authenticated channel, and re-emits the primary's inbound-message notifications as its own. From inside Claude Code both terminals behave the same - send, react, and receive all work in either one.

**Every inbound message reaches every terminal, and each one can reply independently.** An inbound message is delivered to the primary's own Claude Code session AND broadcast to every connected secondary's session - each is a live Claude instance that may act on it. This means one WhatsApp message can produce two (or more, with more terminals) independent replies if more than one session decides to respond. This is a deliberate tradeoff for keeping every terminal fully usable rather than picking one "active" terminal; if you only want one terminal actually replying, treat the others as read-only for that purpose.

**If the primary closes, the other one takes over automatically.** A clean exit (`/exit`, Ctrl+C, the terminal closing normally) hands off within a few seconds. A hard-killed process (crash, force-quit) is only caught by the existing 15-second parent-liveness check, so that takeover can take up to ~30-40 seconds. No manual restart needed either way - the secondary keeps retrying in the background and promotes itself the moment it wins.

**While neither the original connection nor a takeover has succeeded**, a secondary falls back to a `whatsapp_unavailable` stub tool instead of a call that hangs or fails silently - if a tool call reports WhatsApp unavailable right after a crash or restart, wait a few seconds and retry.

**Limits:** same machine only, no cross-machine relay; tested with two terminals, three or more is unsupported.

If something still seems stuck (most likely after killing a terminal outside Claude Code's normal exit path), clear stale processes directly:

```sh
pkill -f "whatsapp.*server"
```

## Statusline (optional)

`scripts/statusline-role.ts` prints `WA:primary`, `WA:secondary` or
`WA:reconnecting` (colored) for whichever terminal it's run in, a dim
`WA:…` while a terminal that asked for the channel is still bringing a
server up, and an empty string when there is no server here at all. Append
it to a Claude Code `statusLine` command to see at a glance which terminal
holds the real connection:

```json
{
  "statusLine": {
    "type": "command",
    "refreshInterval": 5,
    "command": "your-existing-statusline-command && bun <plugin-dir>/scripts/statusline-role.ts"
  }
}
```

`refreshInterval` matters: without it Claude Code only re-runs the command
when you type, so the segment can sit on the dim `WA:…` marker (or a stale
role) until your next message.

Each server records which Claude Code session owns it, so the segment reads
ownership rather than guessing at it: the script climbs its own parent chain
a few hops and takes the role file stamped with one of those pids. Another
terminal's server can never match, and a leftover file from a dead session
cannot either, since a dead process is nobody's parent.

A server started before this shipped, or one whose client does not identify
itself, has no stamp. For those the script falls back to the process tree:
climb to the CLI that owns this terminal (a statusLine setting is a compound
command, so the script runs under a shell that is the plugin's sibling, not
its parent), find the plugin's wrapper among that CLI's descendants, then
`server.ts` under the wrapper. Several processes can carry the plugin dir
name - the statusline shell itself does - so every candidate is tried and
the one with a `server.ts` under it wins. Restarting the session upgrades it
to the stamped path.

Same-machine, read-only, never throws: a miss just means no segment, not a
broken statusline.

## Known limitations

**The message key/proto store is capped and FIFO, shared across every chat.** `download_attachment`, replies and lookups by `message_id` only work for messages still in this in-memory store. The default cap is 500 entries total — on a high-volume account (several active groups, hundreds of unread messages) that can churn through in minutes, so `download_attachment` can fail with `Message not found in store` well before the underlying media has actually expired on WhatsApp's side. Raise it with `WHATSAPP_MAX_STORE=5000` (or whatever fits your traffic) in the server's `.env` — each entry is just a message key plus a small proto, so a much larger cap costs negligible memory. Must be a positive integer; anything else (unset, non-numeric, fractional, zero, negative, `Infinity`) falls back to the 500 default.

**Inbound message delivery is Claude Code only.** Messages are pushed into a session with `notifications/claude/channel`, which is a Claude Code extension, not part of MCP. Other MCP clients (Codex CLI, Gemini CLI, Cursor) drop unknown notifications silently, so there the plugin is poll-only: call `wait_for_messages`, `catch_up` or `unreplied` to see what arrived. Delivery inside Claude Code was broken by client bugs ([#37933](https://github.com/anthropics/claude-code/issues/37933), [#36477](https://github.com/anthropics/claude-code/issues/36477), [#37633](https://github.com/anthropics/claude-code/issues/37633)) and worked again when last checked on v2.1.235; if messages stop appearing, check those issues before suspecting this plugin.

## Resetting auth

```
/whatsapp-channel:configure reset-auth
```

Then relaunch to re-pair.


# WhatsApp — Access & Delivery

WhatsApp has no bot API — this channel connects as a **linked device** (like WhatsApp Web). Any contact who can message the linked phone number can reach the server. The access model described here decides who gets through.

By default, a DM from an unknown sender triggers **pairing**: the server replies with a 6-character code and drops the message. You run `/whatsapp-channel:access pair <code>` from your Claude Code session to approve them. Once approved, their messages pass through.

All state lives in `~/.whatsapp-channel/access.json`. The `/whatsapp-channel:access` skill commands edit this file; the server re-reads it on every inbound message, so changes take effect without a restart. Set `WHATSAPP_ACCESS_MODE=static` to pin config to what was on disk at boot (pairing is unavailable in static mode since it requires runtime writes).

## At a glance

|                     |                                                   |
| ------------------- | ------------------------------------------------- |
| Default policy      | `pairing`                                         |
| Sender ID           | WhatsApp JID (e.g. `886912345678@s.whatsapp.net`) |
| Group key           | Group JID (e.g. `120363424405607157@g.us`)        |
| `ackReaction` quirk | Any emoji — WhatsApp has no fixed whitelist       |
| Config file         | `~/.whatsapp-channel/access.json`                 |

## DM policies

`dmPolicy` controls how DMs from senders not on the allowlist are handled.

| Policy              | Behavior                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/whatsapp-channel:access pair <code>`. |
| `allowlist`         | Drop silently. No reply. Prevents strangers from knowing the linked device is active.             |
| `disabled`          | Drop everything, including allowlisted users and groups.                                          |

```
/whatsapp-channel:access policy allowlist
```

## User IDs (JIDs)

WhatsApp identifies users by **JIDs** — phone number + `@s.whatsapp.net`, e.g. `886912345678@s.whatsapp.net`. The allowlist stores JIDs.

Pairing captures the JID automatically. To add one manually, use the phone number with country code, no leading `+`, followed by `@s.whatsapp.net`.

```
/whatsapp-channel:access allow 886912345678@s.whatsapp.net
/whatsapp-channel:access remove 886912345678@s.whatsapp.net
```

## Groups

Groups are off by default. Opt each one in individually.

```
/whatsapp-channel:access group add 120363424405607157@g.us
```

Group JIDs end in `@g.us`. To find one, add the linked device to the group — the server logs the group JID when it receives a message from an unenabled group.

With the default `requireMention: false`, the server responds to every message. Pass `--mention` to require @mention, or `--allow jid1,jid2` to restrict which members can trigger it. Pass `--roster` to also grant roster access (see below) — off by default, same as everything else. Pass `--no-context` to keep a mention-gated group's unaddressed messages out of the local log entirely (see "Mention detection" below).

Running `group add` again on an already-configured group **merges**, it doesn't start over: any flag you don't pass this time keeps whatever was already set. Adding `--roster` to a group that already has `--mention` on doesn't reset `--mention` back off — only the flags you actually pass change anything. To explicitly turn `--mention` or `--roster` back off (rather than just never setting them), pass `--no-mention` / `--no-roster` — passing both a flag and its negation at once is refused, not silently resolved one way.

```
/whatsapp-channel:access group add 120363424405607157@g.us
/whatsapp-channel:access group add 120363424405607157@g.us --mention
/whatsapp-channel:access group add 120363424405607157@g.us --allow 886912345678@s.whatsapp.net
/whatsapp-channel:access group add 120363424405607157@g.us --roster
/whatsapp-channel:access group add 120363424405607157@g.us --no-roster
/whatsapp-channel:access group rm 120363424405607157@g.us
```

### Per-group personality & memory

Each enabled group gets a config directory at `~/.whatsapp-channel/groups/<groupJid>/`:

| File        | Purpose                                                                            |
| ----------- | ---------------------------------------------------------------------------------- |
| `config.md` | Personality, goals, and instructions for Claude in this group. User edits this.    |
| `memory.md` | Conversation summaries appended by Claude automatically. Persists across sessions. |

Created automatically when a group is added. Edit `config.md` to customize Claude's behavior per group. View or clear with `/whatsapp-channel:access group config <jid>` and `/whatsapp-channel:access group memory <jid>`.

### LID identifiers

Baileys 7 uses LID (Local Identifier) format alongside phone JIDs. The same person may appear as both `16024101202@s.whatsapp.net` and `21737517412478@lid`. The server maintains a mapping at `~/.whatsapp-channel/lid-map.json` and resolves both formats automatically. Both work in allowlists.

## Names and privacy

**On by default.** The server caches saved contact names from WhatsApp's own contact sync (the same list your phone already has) at `~/.whatsapp-channel/contacts.json` — never the `contacts.md` some people keep for their own DM habits, which this plugin never reads. Set `WHATSAPP_CACHE_CONTACTS=0` to turn that off. Only a contact's **name** is cached this way, and only a name — self-reported "About"/display text (`.notify`) is kept separately and never trusted for anything security-relevant, since anyone messaging the account can set that to whatever they want, including someone else's real name or number.

With it off, name resolution and masking still work for the lifetime of the running server (built fresh each run from WhatsApp's own contact sync), it just isn't written to disk — so a restart starts blank again and the wizard's contact screen has nothing to rank. Either way, `WHATSAPP_ACCESS_MODE=static` disables this cache regardless of `WHATSAPP_CACHE_CONTACTS` — a static deployment can't be made to write local state by an env var. The recency cache the wizard ranks by (`dm-activity.json`) follows the same switch, since it's the same kind of per-sender data — including for a sender the access gate rejected, which is exactly what the opt-out is there for.

Saved names sync on connect only when the cache holds no saved name at all (WhatsApp otherwise sends them just once, at linking); the server asks for the contact list again, logs a count (`address book synced: 3 saved name(s)` — never a name or number), and stops asking once one is cached. It never asks more than once a day (tracked in `~/.whatsapp-channel/.addressbook-sync`, so a restart doesn't reset the clock), and never runs with `WHATSAPP_CACHE_CONTACTS=0` or in static mode. After it, `contacts.json` holds the names of everyone saved on your phone, on your disk only. Chat-list order cannot be re-fetched, so the wizard ranks by the DM activity it has seen since.

Someone who merely messaged the account and was never approved doesn't sit in these caches forever: on the server's hourly prune, a `dm-activity.json` entry older than **90 days** is dropped unless its number is allowlisted (for a DM or any group), and a `contacts.json` entry with no saved name is dropped once it has no allowlist entry, no DM younger than that, and no sighting in the last **30 days**. A sighting is any message or contact sync in which WhatsApp showed the server that person, so a member of a group the account is in stays cached while they keep appearing and for 30 days after they were last seen; their `lid-map.json` entry is dropped once nothing above keeps them. Saved names — your own phone's address book — never age out this way.

**Names may reach the AI model; raw phone numbers should not.** When you ask Claude to reply and mention someone, it can use a saved name (`"Akash"`) instead of a number — that name is what appears in Claude's context. Anywhere a number would otherwise be shown to a human (an ambiguous-name error, `group_roster` for someone with no saved name) it's masked to the last 4 digits (`•••••5122`) before it's built into any string, not filtered afterward.

This is a **best-effort mitigation, not a hard guarantee**: it depends on a saved contact actually being a name and not, say, a phone number typed into the name field, and on `.notify` not being trusted in place of it (checked explicitly for group rosters — see below). If you'd rather no name data ever reaches an AI model at all, don't grant any group `roster` access and use raw JIDs/numbers in `mentions` instead of names.

The in-session `review` (see the wizard section below) never lets the candidate lists enter the session — only the `+`/`-` list comes back, by saved name or masked number.

## Group roster & @all mentions

`group_roster` (an MCP tool) and `"all"` (a reserved value in the `reply` tool's `mentions` array) both require a group's `roster` flag — separate from whether Claude can act in the group at all, so you can let Claude reply in a large group without ever handing it the member list.

- **`group_roster`** lists a group's current members by saved contact name, or a masked number when no name is known. It never shows a raw number, even when a contact's self-reported name happens to look like one.
- **`"all"` in `mentions`** is sent as WhatsApp's own everyone-tag: one `@all` in the text and no member list, so it reaches every current member, saved or not. WhatsApp only honours that tag from a group admin once a group has more than 32 members, so there, unless the linked account is an admin, the message lists every member in its mentions instead (as 0.26.2 did for every group). A saved contact literally named "All" is unaffected: name resolution always wins over the reserved value.

Both fail with a clear error if the group's `roster` flag isn't granted.

## Guided bulk setup (wizard)

`bun scripts/access.ts wizard` draws **one screen**: a search line, a `Picked:` chip
line, then **CONTACTS** on the left and **GROUPS** on the right - everything Claude
can already reach comes pre-ticked, so the screen shows you the current state instead
of an empty list. Untick to take access away, tick to grant, in the same pass.

Keys: type to filter both columns, `↑↓` move, `Tab` (or `←→`) switches column,
`Space` toggles, `Enter` in the search box ticks the highlighted row and clears the
search, `Enter` on an empty search box submits, `Backspace` on an empty search box
removes the last chip, `r` flags a group you are granting for roster access, `Ctrl+Z`
undoes one step, `Ctrl+R` restores the access list from before the last run, `Esc`
clears the search or quits, `Ctrl+C` quits. Mouse is best effort: click a row to
toggle it, click a chip's `×` to remove it.

A chip with `-` and struck through is a grant you are taking away; a chip with `+` is
a new grant.

**It needs a real terminal.** Run it through a pipe or from inside an AI session and
it says so and exits, rather than hanging.

`/whatsapp-channel:access review` opens this same screen in a new terminal window, waits for you, and reports back only what changed; `bun scripts/access.ts undo` (or `wizard --undo`) is still the one step back.

- Roster is still asked only for groups you are granting in this run - flag one with
  `r` while it is ticked. An already-granted group keeps whatever roster flag it has
  (change it with `group add --roster` / `--no-roster`).
- Archived groups are still skipped by default and the wizard says how many it hid —
  pass `--include-archived` to include them.
- Nothing is written until you have seen the full `+`/`-` list and answered **Apply these changes?**. Answer `n`, or press Ctrl-C at any point, and nothing is written at all.
- After a write, `bun scripts/access.ts wizard --undo` (or plain `undo`) puts the access list back to just before that run - run it with `--dry-run` first. Cached names are not restored, only the access list.
- A group the wizard adds gets `requireMention: true` (only reply when addressed) — a more cautious default than `group add`'s own CLI default of `false` (reply to everything), deliberately: the wizard is the guided path for a less technical setup, the CLI is for someone already comfortable with explicit flags. Change it after the fact with `group add --mention` or `--no-mention`.
- Narrow terminal (under 70 columns)? The two columns stack instead of sitting side
  by side. `NO_COLOR` is respected.

### Taking access back (`wizard --revoke`)

`--revoke` is now an **alias for the same screen** - there is only one wizard, and
revoking is unticking a row that came pre-ticked. The old command keeps working so a
habit or an old note does not break.

- **An untouched screen changes nothing.** Submitting without touching a tick is
  read as "leave everything as it is" - it can neither remove nor grant.
- Revoking a group keeps its `config.md` and `memory.md`, same as `group rm`, in case
  you add it back.
- Revoking a contact **keeps their cached name and recency** - only the allowlist
  entry goes. `remove <jid>` still forgets a cached name (unchanged), and
  `forget <jid>` still clears one deliberately - see "Removing someone already
  granted access" below.

Group/contact names and recency come from caches (`~/.whatsapp-channel/groups-meta.json`, `dm-activity.json`, `contacts.json`) that only the running server writes — automatically, as WhatsApp reports chat activity, no manual step needed once the account has connected at least once. If it's never connected yet, the wizard has nothing to show; pair it first. `groups-meta.json` is always written; `dm-activity.json` and `contacts.json` follow `WHATSAPP_CACHE_CONTACTS`, which is on unless you set it to `0` (see "Names and privacy" above) — with it off, the CONTACTS column has nothing to rank. The CONTACTS column lists recent DM activity first, then every contact whose _saved_ name arrived from the phone's contact sync — so someone you've never messaged is still findable by name. A number with no saved name is never offered: if it was not worth saving on the phone it is not worth a row, and a self-reported display name never earns one on its own. The only unnamed rows are numbers that are _already_ allowed — they sit at the bottom of the column, masked, so they can still be unticked. The wizard says so on screen instead of silently leaving the column empty: one line when no DM activity is on record at all, another when there is a record but nothing new in it. Its search line runs over the whole cached pool, so every contact on record is reachable without paging.

It's a terminal command, deliberately not a Claude Code skill: running it yourself, outside any chat, is what makes this true —

> No group or contact data was sent to any AI model during this setup — this ran entirely in your terminal.

The `/whatsapp-channel:access` skill will point you at it if you ask for guided setup there, but will never run it on your behalf.

### Removing someone already granted access

`/whatsapp-channel:access remove <jid>` (or the equivalent CLI command) also forgets their cached name from `contacts.json`, not just their allowlist entry — it does **not** touch `lid-map.json` (needed for correct message/mention matching if they're still an active participant in a group you can see) and it does **not** remove them from any shared group or block them on WhatsApp, neither of which this plugin does. If they're still in a group with roster access, they'll show up there as a masked number from then on instead of by name — the honest consequence of choosing to forget someone this plugin was never going to remove from a group.

### Forgetting a stranger who was never allowlisted

`remove` requires the JID to actually be on the allowlist, and refuses otherwise — but with caching on, a name/activity entry gets cached the moment anyone DMs once, allowlisted or not (contact and chat sync events fire before the access gate runs). `bun scripts/access.ts forget <jid>` purges the cached name and activity entry directly, with no allowlist requirement — the general-purpose way to clear someone's cached identity regardless of whether they were ever approved.

## Mention detection

A message that does **not** trigger the server in such a group is still kept, text only, in `~/.whatsapp-channel/messages.jsonl` with `routed: false`, so `catch_up` can show you the room; it is never routed, notified or counted as unreplied (retention rules: [USAGE.md, "Your own replies"](./USAGE.md#your-own-replies)). Nothing is kept for a group that is not allowlisted or for a sender the group's own `allowFrom` rejects. If you'd rather mention gating keep its stricter pre-0.22 meaning for a group — _nothing_ about the room is stored unless Claude was addressed — opt that group out with `group add <jid> --no-context` (`--context` turns it back on; a group that never set it keeps context, the 0.22.0 default). This is per group, because the trade is per group: context is what lets Claude draft a message that fits the conversation, and the people it stores are the group's other members.

In groups with `requireMention: true`, any of the following triggers the server:

- A structured @mention of the linked account's JID
- A match against any regex in `mentionPatterns`

```
/whatsapp-channel:access set mentionPatterns '["claude", "assistant"]'
```

## Delivery

Configure outbound behavior with `/whatsapp-channel:access set <key> <value>`.

**`ackReaction`** reacts to inbound messages on receipt. WhatsApp supports **any emoji** — there's no fixed whitelist like Telegram.

```
/whatsapp-channel:access set ackReaction 👀
/whatsapp-channel:access set ackReaction ""
```

**`replyToMode`** controls threading on chunked replies. When a long response is split, `first` (default) threads only the first chunk under the inbound message; `all` threads every chunk; `off` sends all chunks standalone.

**`textChunkLimit`** sets the split threshold. Default is 4096.

**`chunkMode`** chooses the split strategy: `length` cuts exactly at the limit; `newline` prefers paragraph boundaries.

### Read receipts

**Off by default.** Set `WHATSAPP_READ_RECEIPTS=1` (in the environment or `~/.whatsapp-channel/.env`) to mark each message read, turning the sender's ticks blue, as soon as it is delivered to Claude. Only messages that pass the access gate are marked: a dropped sender, a pairing request, an unaddressed message in a mention-gated group and the backlog replayed after a reconnect are left unread. In a group the receipt names the sender, as the phone does. If read receipts are off in your phone's privacy settings, the chat is marked read on your own devices only and senders see no blue ticks. The `mark_read` tool marks one message read by hand. Linked device only: Cloud API sends no read receipts and has no `mark_read`.

### Proactive notifications

**On by default.** Claude is told to surface things to you the moment they happen, rather than waiting for its next natural reply: an inbound message arriving, this terminal's role _changing_ (becoming primary/secondary, or losing/regaining the primary connection — see multi-terminal sync below; the initial role a terminal starts in isn't a change and never announces), a pairing code, and the "paired and connected" confirmation. Set `WHATSAPP_QUIET=1` to turn all of that off for one terminal — a pairing code you need is always in the diagnostics log regardless, so nothing is lost. It's an env var, not a saved setting, so it only applies to sessions started with it set — nothing persists it past the terminal that set it.

This only controls whether Claude _announces_ a role change. The underlying primary/secondary mechanism, the role file, and the statusline segment (see `scripts/statusline-role.ts`) all keep working the same either way. The update notice below is separate — it isn't gated by `WHATSAPP_QUIET`.

### Update notice

Two different notices, from the same script (`scripts/update-notice.ts`):

**"What's new"** — the first Claude Code session that starts after the plugin's version has changed since it last recorded one (tracked per account in `~/.whatsapp-channel/.last-seen-version`) gets a one-time notice listing what changed, including a nudge toward the [guided setup wizard](#guided-bulk-setup-wizard) if you haven't run it yet — the wizard itself has always required running manually, this notice is just what tells you it exists. Shown once per version, not on every session start of the same version. Delivered by the SessionStart hook (`hooks-handlers/session-start.sh`, calling `scripts/update-notice.ts`), the same hook that guides first-time setup — a version bump has nothing to do with the WhatsApp connection itself, so it doesn't come from `server.ts`. Skipped in `WHATSAPP_ACCESS_MODE=static`, which can't write local state (see above).

**"Update available"** — tells you a newer version exists that you have **not** installed yet, which the notice above can never do (it reads the running plugin's own `plugin.json`, so by the time it fires you already have the update). It compares the running version against what your marketplace clone advertises — both already on disk, refreshed by Claude Code itself, so there is no network call and nothing to configure. Unlike the one-time notice it is not gated on `.last-seen-version`: it repeats every session until you actually update, and goes quiet by itself once you do. Also skipped in static mode.

## Skill reference

| Command                                                          | Effect                                                                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/whatsapp-channel:access`                                       | Print current state: policy, allowlist, pending pairings, enabled groups.                                                                                                                                                                                                                                                                          |
| `/whatsapp-channel:access pair a4f91c`                           | Approve pairing code `a4f91c`. Adds the sender to `allowFrom` and sends a confirmation on WhatsApp.                                                                                                                                                                                                                                                |
| `/whatsapp-channel:access deny a4f91c`                           | Discard a pending code. The sender is not notified.                                                                                                                                                                                                                                                                                                |
| `/whatsapp-channel:access allow 886912345678@s.whatsapp.net`     | Add a JID directly.                                                                                                                                                                                                                                                                                                                                |
| `/whatsapp-channel:access remove 886912345678@s.whatsapp.net`    | Remove from the allowlist.                                                                                                                                                                                                                                                                                                                         |
| `/whatsapp-channel:access policy allowlist`                      | Set `dmPolicy`. Values: `pairing`, `allowlist`, `disabled`.                                                                                                                                                                                                                                                                                        |
| `/whatsapp-channel:access group add 120363424405607157@g.us`     | Enable a group (merges into an existing entry). Flags: `--mention`/`--no-mention`, `--allow jid1,jid2`, `--roster`/`--no-roster`, `--context`/`--no-context`.                                                                                                                                                                                      |
| `/whatsapp-channel:access group rm 120363424405607157@g.us`      | Disable a group.                                                                                                                                                                                                                                                                                                                                   |
| `/whatsapp-channel:access set ackReaction 👀`                    | Set a config key: `ackReaction`, `replyToMode`, `textChunkLimit`, `chunkMode`, `mentionPatterns`.                                                                                                                                                                                                                                                  |
| `/whatsapp-channel:access set owner 886912345678@s.whatsapp.net` | The chat that receives permission requests and the only one that may approve them. Must already be on `allowFrom` (so it cannot be set before the first connect on a fresh install, when the linked account is added). Stamped once by the server if missing; removing that contact later sends requests to your own chat until you set a new one. |

## Config file

`~/.whatsapp-channel/access.json`. Absent file is equivalent to `pairing` policy with empty lists, so the first DM triggers pairing.

```jsonc
{
  // Handling for DMs from senders not in allowFrom.
  "dmPolicy": "pairing",

  // WhatsApp JIDs allowed to DM.
  "allowFrom": ["886912345678@s.whatsapp.net"],

  // Where permission requests go and who may approve them. Must be in allowFrom.
  // Stamped by the server when missing (the linked account on a fresh install).
  "owner": "886912345678@s.whatsapp.net",

  // Groups the channel is active in. Empty object = DM-only.
  "groups": {
    "120363424405607157@g.us": {
      // true: respond only to @mentions.
      "requireMention": true,
      // Restrict triggers to these senders. Empty = any member (subject to requireMention).
      "allowFrom": [],
      // Grants group_roster and "all" mentions. Separate from acting in the
      // group at all - see "Group roster & @all mentions" above.
      "roster": false,
    },
  },

  // Case-insensitive regexes that count as a mention.
  "mentionPatterns": ["claude"],

  // Any emoji. Empty string disables.
  "ackReaction": "👀",

  // Threading on chunked replies: first | all | off
  "replyToMode": "first",

  // Split threshold.
  "textChunkLimit": 4096,

  // length = cut at limit. newline = prefer paragraph boundaries.
  "chunkMode": "newline",
}
```
