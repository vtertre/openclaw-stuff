# OpenClaw on Discord — Setup Guide

A clean, production-grade setup for running me on your Discord server.

---

## Overview

OpenClaw supports Discord fully: DMs, guild channels, threads, slash commands, reactions, and voice. This guide covers the complete setup with best practices for security, architecture, and organisation.

---

## 1. Discord Bot Creation

### 1.1 Create the Application

1. Go to [Discord Developer Portal](https://discord.com/developers/applications)
2. **New Application** → name it (or whatever you'll call it on Discord)
3. Go to **Bot** tab → set the username to match

### 1.2 Enable Privileged Intents

In **Bot → Privileged Gateway Intents**, enable:

- ✅ **Message Content Intent** — required
- ✅ **Server Members Intent** — required for allowlists and name resolution
- ⬜ Presence Intent — optional (only if you want presence updates)

### 1.3 Get the Bot Token

**Bot** tab → **Reset Token** → copy and store it securely (treat it like a password).

### 1.4 Generate the Invite URL

**OAuth2 → URL Generator**:

Scopes:
- ✅ `bot`
- ✅ `applications.commands`

Bot permissions (least-privilege baseline):
- ✅ View Channels
- ✅ Send Messages
- ✅ Read Message History
- ✅ Embed Links
- ✅ Attach Files
- ✅ Add Reactions
- ❌ Administrator (never needed)

Copy the generated URL, open it in your browser, select your server.

---

## 2. Collect Your IDs

Enable Developer Mode in Discord: **Settings → Advanced → Developer Mode ✅**

Then right-click to copy:
- **Server ID** (right-click server icon)
- **Your User ID** (right-click your avatar)
- **Channel IDs** you want to use (right-click channels)

---

## 3. Inject the Bot Token Securely

**Do NOT paste the token in chat.** Set it directly on the server:

```bash
openclaw config set channels.discord.token '"YOUR_BOT_TOKEN"' --json
openclaw config set channels.discord.enabled true --json
openclaw gateway restart
```

Or via env variable (simpler for server deployments):
```bash
export DISCORD_BOT_TOKEN=your_token_here
```

---

## 4. Pairing (First DM)

1. DM your bot in Discord → it replies with a pairing code
2. Send me the code on Telegram: `"Approve this Discord pairing code: <CODE>"`

Or via CLI:
```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

Pairing codes expire after 1 hour.

---

## 5. Guild (Server) Configuration

Recommended config for a **private personal server** (just you and the bot):

```json5
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "YOUR_BOT_TOKEN",
      "groupPolicy": "allowlist",
      "guilds": {
        "YOUR_SERVER_ID": {
          "requireMention": false,
          "users": ["YOUR_USER_ID"]
        }
      }
    }
  }
}
```

Key points:
- `groupPolicy: "allowlist"` — only your server can talk to the bot (secure default)
- `requireMention: false` — responds to every message in allowed channels (fine on a private server)
- `users` — allowlist by numeric ID (safer than names)

Apply:
```bash
openclaw gateway restart
```

---

## 6. Channel Architecture

Each Discord channel gets its **own isolated session with its own context**. Design your channels intentionally:

| Channel | Purpose | Notes |
|---|---|---|
| `#general` | Casual chat, quick questions | requireMention: true |
| `#coding` | Dev tasks, code review, Codex | Long-running tasks |
| `#research` | Web search, article summaries | |
| `#jobs` | Veille emploi discussions | |
| `#home` | Personal tasks, reminders | |
| `#voice` | (optional) Voice conversations | Needs extra config |

To restrict the bot to specific channels only (optional):

```json5
{
  "channels": {
    "discord": {
      "guilds": {
        "YOUR_SERVER_ID": {
          "requireMention": false,
          "users": ["YOUR_USER_ID"],
          "channels": {
            "CHANNEL_ID_1": { "allow": true },
            "CHANNEL_ID_2": { "allow": true, "requireMention": true }
          }
        }
      }
    }
  }
}
```

---

## 7. Memory & Context in Guild Channels

**Important:** `MEMORY.md` does NOT auto-load in guild channels (by design — security). It only loads in DM sessions.

What loads everywhere:
- `AGENTS.md` — workspace rules
- `USER.md` — your profile and preferences
- `SOUL.md` — my personality

To access long-term memory in a channel, just ask: "Remember what we decided about X?" and I'll use `memory_search`.

If you want stable context in a specific channel, put it in the **channel topic** (injected as untrusted context) or create a dedicated file I can reference.

---

## 8. Threads & Sub-agents (Advanced)

Discord threads can be bound to isolated agent sessions — great for coding tasks:

Enable thread bindings in config:

```json5
{
  "channels": {
    "discord": {
      "threadBindings": {
        "enabled": true,
        "idleHours": 24,
        "spawnSubagentSessions": true,
        "spawnAcpSessions": true
      }
    }
  }
}
```

Then use slash commands:
- `/focus` — bind current thread to a sub-agent session
- `/unfocus` — release binding
- `/agents` — see active runs

This is how you'd delegate coding tasks to Codex or Claude Code in a dedicated thread.

---

## 9. Quality of Life Settings

### Streaming (see replies as they're generated)

```json5
{
  "channels": {
    "discord": {
      "streaming": "partial"
    }
  }
}
```

### Reply threading

```json5
{
  "channels": {
    "discord": {
      "replyToMode": "first"
    }
  }
}
```

### Ack reaction (bot reacts while thinking)

Automatically uses my emoji (🦉) — no config needed.

### Longer timeout for heavy tasks

```json5
{
  "channels": {
    "discord": {
      "accounts": {
        "default": {
          "eventQueue": {
            "listenerTimeout": 120000
          }
        }
      }
    }
  }
}
```

---

## 10. Security Checklist

- [ ] Bot token stored via `openclaw config set` or env var, never in chat
- [ ] `groupPolicy: "allowlist"` (not `open`)
- [ ] `users` allowlist uses numeric IDs, not usernames
- [ ] No `Administrator` permission on the bot
- [ ] Keep `allowBots: false` (default) to prevent bot loops
- [ ] Run `openclaw doctor` after setup to catch misconfigs
- [ ] Run `openclaw security audit` periodically
- [ ] Back up `~/.openclaw/openclaw.json` before running setup commands (config writes can overwrite!)

---

## 11. Observability (Optional but Useful)

If you want to inspect every API call I make (prompts, tokens, responses), route through [tapes](https://tapes.dev):

```bash
tapes serve \
  --provider anthropic \
  --upstream "https://api.anthropic.com" \
  --proxy-listen "0.0.0.0:8080" \
  --sqlite "./tapes.db"
```

Update config:
```json5
{
  "providers": {
    "anthropic": {
      "baseUrl": "http://localhost:8080"
    }
  }
}
```

Restart: `openclaw gateway restart`

---

## 12. The "Release Notes on Autopilot" Pattern

This is the pattern from the article that triggered this whole setup. It's a good example of what structured Discord channels unlock.

### The idea

> Give your agent standing instructions for recurring tasks, and let the system work while you don't.

**Information arrives → Agent has standing instructions → You get the output, not the input.**

### Concrete setup

**Step 1: Dedicated channels by function**

Don't dump everything in one place. Each channel = one clear context:

| Channel | Purpose |
|---|---|
| `#releases` | OpenClaw changelogs land here |
| `#engineering` | Tech planning, config work |
| `#research` | Research deliverables |

The agent reads these channels. Clear channel purpose = no need to explain context each time.

**Step 2: Pipe release notes into `#releases`**

Options:
- GitHub webhook → posts to `#releases` on each new release
- Follow the [OpenClaw announcements channel](https://discord.com/invite/clawd) and point it at your `#releases`

**Step 3: One heartbeat instruction**

In `HEARTBEAT.md`, add:

```
When a new release lands in #releases, analyze the changelog against my current config.
Flag breaking changes. List improvement opportunities. If nothing relevant, skip silently.
```

That's it. No code. On each heartbeat, I check `#releases`, compare against your setup, and proactively ping you if something is relevant — otherwise silence.

**Result:** Instead of skimming 86 changelog items every morning, you wake up to:
> "Version X released. Two changes relevant to your setup: A and B. No breaking changes. One new feature worth enabling: Z. Here's why."

### Why this works

The pattern scales beyond release notes:
- Meeting prep briefs
- Research digests  
- Status reports
- Job offer alerts

Same structure every time: information arrives somewhere the agent watches → standing instruction → you get a summary, not the raw feed.

---

## 14. Useful Commands

```bash
openclaw health                    # Check Discord connection status
openclaw doctor                    # Full diagnostics
openclaw channels status --probe   # Verify permissions per channel
openclaw logs --follow             # Live log stream
openclaw gateway restart           # Apply config changes
openclaw security audit            # Security check
```

---

## 15. Known Gotchas

- **Config overwrites**: Some `openclaw` commands rewrite `openclaw.json`. Always back it up first.
- **MEMORY.md**: Doesn't load in guild channels — by design. Use `memory_search` on demand.
- **requireMention default**: Bots are mention-gated in guilds by default. Set `requireMention: false` explicitly for private servers.
- **Channel IDs vs slugs**: Use numeric IDs everywhere in config — slugs work for routing but `--probe` can't verify permissions for them.
- **Pairing codes**: Expire after 1 hour. If it expires, DM the bot again.

---

## 16. Quick Setup Checklist (TL;DR)

1. [ ] Create Discord app + bot at developer portal
2. [ ] Enable Message Content + Server Members intents
3. [ ] Copy bot token, server ID, user ID
4. [ ] Invite bot to server with correct scopes/permissions
5. [ ] `openclaw config set channels.discord.token '"..."' --json`
6. [ ] `openclaw config set channels.discord.enabled true --json`
7. [ ] `openclaw gateway restart`
8. [ ] DM the bot → get pairing code → approve it
9. [ ] Configure guild allowlist in `openclaw.json`
10. [ ] `openclaw gateway restart`
11. [ ] `openclaw health` → verify all green
12. [ ] Set up your channels with intention

---

*Sources: [OpenClaw Discord docs](https://docs.openclaw.ai/channels/discord) · [exe.dev setup guide](https://dev.to/bdougieyo/setting-up-openclaw-on-exedev-with-discord-27n)*
