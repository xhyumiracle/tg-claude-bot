<div align="center">

# tg-claude-bot

**Your local Claude Code, in your pocket.**

A single-file Telegram bridge to the Claude Code CLI: resume any session from
your phone, keep every tool and skill, answer permission prompts with inline
buttons, talk to it with voice messages.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](pyproject.toml)
[![Built on claude-agent-sdk](https://img.shields.io/badge/built%20on-claude--agent--sdk-d97757.svg)](https://github.com/anthropics/claude-agent-sdk-python)
![Single file](https://img.shields.io/badge/single%20file-~1.6k%20lines-brightgreen.svg)
![No database](https://img.shields.io/badge/database-none-lightgrey.svg)

</div>

---

Start a session in your terminal, walk away, and continue it from Telegram —
same session, same context, same tools. The bot is a thin stateless router over
the Claude Agent SDK; the CLI keeps owning sessions, tools, skills, and
persistence. There is very little here that can break.

## Highlights

| | |
|---|---|
| 🔁 **Resume any real session** | `/resume` opens an inline picker over your actual CLI session store (`~/.claude/projects/*.jsonl`), showing the CLI's own AI-generated titles, project, and age. Cross-project: the working directory is auto-detected from the session file. |
| 🧵 **Per-topic sessions** | Every Telegram forum topic is an independent Claude session, each resumable separately. One conversation per `(chat, topic)`. |
| ⏩ **Zero command remapping** | Unknown `/commands` are forwarded verbatim to the CLI — `/compact`, custom skills, anything headless just works. Local-command output (`/context`, `/cost`, …) is captured and relayed. |
| 🔘 **Buttons instead of a TUI** | Permission requests, plan-mode approval (`ExitPlanMode`), and Claude's clarifying questions (`AskUserQuestion`) all surface as inline buttons — a generic `can_use_tool` bridge, no per-tool code. |
| 🎤 **Voice messages** | Local transcription via faster-whisper (bilingual zh/en incl. code-switching, editable 🎤 transcript, lazy model download). No audio leaves your machine. |
| 🖼 **Native media** | Images travel as base64 content blocks inside the message — part of the session transcript, lifecycle owned by the CLI. Other files land in an auto-gitignored media dir with TTL cleanup. |
| 📟 **Live status** | Tool activity shows as a `⏳ Working…` message edited in place, which morphs into the reply. Long commands get an elapsed-time ticker. No notification spam, no token cost. |
| 🎛 **CLI parity** | `/model` (live switch, list from the official `/v1/models` API), `/effort` (levels probed from the CLI's own validator), `/usage` (subscription limits from the OAuth endpoint), `/stop` to interrupt a turn. |
| 🟠 **Context warnings** | 🟠 at 80% / 🔴 at 90% of the model's real context window, computed from per-turn API usage — same source as `/context`. |
| ♻️ **Graceful deploys** | Touch a flag file; the bot restarts only when every conversation is idle. In-flight replies are never lost. Restarts show as one silent `♻️ → ✅` message. |
| 🔒 **Owner/guest profiles** | Allowlisted chats only. Owner gets full access; guests get scoped read/write with button escalation to the owner for anything else. |

## How it compares

| | **tg-claude-bot** | tmux-scraping bridges | direct-API bots |
|---|---|---|---|
| Backend | Claude Agent SDK (structured events) | terminal ANSI scraping | raw Anthropic API |
| Your real CLI sessions | ✅ resume any, with AI titles | ⚠️ attach to live panes only | ❌ separate world |
| Tools, skills, MCP | ✅ everything the CLI has | ✅ | ❌ reimplemented, if at all |
| Permission prompts | ✅ inline buttons | ❌ blind keypresses | n/a |
| Survives restarts | ✅ stateless, CLI owns state | ❌ tied to tmux lifetime | needs a database |
| Moving parts | one Python file | tmux + parser + bot | bot + DB + API glue |

The trade-off is deliberate: no attaching to a *live* interactive terminal
(that's what tmux bridges like [ccbot](https://github.com/six-ddc/ccbot) do,
at the cost of scraping ANSI output). This bridge trades that for structured
SDK events and statelessness.

## Commands

| Command | What it does |
|---|---|
| `/resume` | Inline session picker (titles, project, age); `/resume <id>` binds directly |
| `/new` | Start a fresh session in this chat/topic |
| `/status` | Current binding: session, project, model, effort |
| `/model` | Live model picker — real names and context windows from `/v1/models` |
| `/effort` | Reasoning-effort picker — levels discovered from the CLI itself |
| `/usage` | Subscription limits (5h / weekly / per-model / credits) from the OAuth usage endpoint |
| `/whisper` | Pick the voice-transcription model |
| `/stop` (`/esc`) | Interrupt the current turn — the CLI's ESC |
| anything else | Forwarded verbatim to the CLI: `/compact`, `/context`, `/cost`, your skills… |

The command menu is registered natively (`setMyCommands`), so `/` autocompletes
in Telegram.

## Architecture

```
Telegram ── python-telegram-bot ── bot.py (stateless router)
                                     │  claude-agent-sdk (one client per chat/topic)
                                     └─ Claude Code CLI ── ~/.claude/projects/*.jsonl
```

All session state lives in the CLI's own files. The bot holds nothing worth
losing: kill it, redeploy it, nothing is forgotten.

## Quick start

You already run Claude Code — it's the prerequisite — so the fastest path is
letting it install its own bridge. Paste this into Claude Code **on the machine
that should host the bot**:

> Set up https://github.com/xhyumiracle/tg-claude-bot for me: clone it, install
> dependencies with uv (ask me whether I want voice-message support), walk me
> through creating a bot with @BotFather and finding my numeric Telegram user
> id, fill in `.env`, then run it (or install the systemd unit) and stay with
> me until `/status` answers on my phone.

It will drive the whole checklist below and you only tap @BotFather.

### Manual setup

**1. Prerequisites** — a machine with the
[Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and
logged in, plus [uv](https://docs.astral.sh/uv/).

**2. Create your bot** — message [@BotFather](https://t.me/BotFather) →
`/newbot` → copy the token. For use inside a group, also either disable privacy
mode (`/setprivacy` → Disable) or make the bot a group admin, so it can see
messages.

**3. Find your user id** — message [@userinfobot](https://t.me/userinfobot);
it replies with your numeric id.

**4. Install & configure**

```bash
git clone https://github.com/xhyumiracle/tg-claude-bot && cd tg-claude-bot
uv sync                 # add --extra voice for local voice transcription
cp .env.example .env    # fill in TG_BOT_TOKEN and OWNER_USER_ID at minimum
```

**5. Run it**

```bash
uv run python bot.py
```

DM your bot `/status` — you're live. For always-on operation, adapt the paths
in [tg-claude-bot.service](tg-claude-bot.service) and install it as a systemd
unit.

### Configuration

Everything lives in `.env` (see [.env.example](.env.example)):

| Variable | Purpose |
|---|---|
| `TG_BOT_TOKEN` | Bot token from @BotFather *(required)* |
| `OWNER_USER_ID` | Your numeric Telegram id — full access *(required)* |
| `GUEST_USER_IDS` | Extra user ids served with the restricted guest profile |
| `TARGET_GROUP_ID` | A group to serve (guest profile; topics = separate sessions) |
| `OWNER_DEFAULT_CWD` | Default working directory for new owner sessions |
| `RESUME_SESSION_ID` | Session to bind the owner's DM to on first contact |
| `GUEST_READ_DIRS` / `GUEST_WRITE_DIRS` | Colon-separated dirs guests may read / write |
| `GUEST_SYSTEM_PROMPT_FILE` | Custom system prompt for the guest profile |
| `WHISPER_MODEL` | faster-whisper model (default `large-v3-turbo`) |
| `TGBOT_MEDIA_TTL_DAYS` | Retention for received files (default 14) |

## Security model

- Only allowlisted chats are served; everything else is ignored.
- The owner runs with full permissions; guests run a scoped profile
  (read/write limited to configured dirs, custom system prompt) and
  out-of-scope tool calls escalate to the owner as Allow/Deny buttons.
- Session management commands are owner-gated everywhere.
- Voice notes are transcribed locally and deleted; images live inside the
  CLI's own transcript retention.

## Non-goals

- Attaching to a *live* interactive terminal session — see the comparison
  above; that's a different trade-off.
- Replicating TUI-only dialogs verbatim (`/config` etc.); their capabilities
  are rebuilt as bot commands where they matter (`/model`, `/effort`, `/usage`).
- Being a framework. It's one file — read it, fork it, make it yours.

## License

[MIT](LICENSE)
