# tg-claude-bot

A single-file Telegram ↔ Claude Code bridge: chat with your local Claude Code CLI
sessions from Telegram, resume any session, per-topic routing, local voice
transcription, and inline-button interactivity. ~750 lines, no database, no tmux.

## Features

**Session routing**
- `/resume` — inline-button picker over your real CLI session store
  (`~/.claude/projects/*/*.jsonl`), showing the CLI's own AI-generated session
  titles, project and age; `/resume <id>` binds directly
- One conversation per `(chat, forum topic)`: every Telegram topic is an
  independent Claude session, each resumable separately
- `/new`, `/status`; cross-project resume (cwd auto-detected from the session file)

**Command passthrough, no re-mapping**
- Unknown `/commands` are forwarded verbatim to the CLI, so `/compact`, custom
  skills, and anything the CLI supports headlessly just works — zero per-command
  code
- CLI local-command output (`/context`, `/cost`, …) is captured and relayed
- A small data-driven `SHELL_CMDS` table serves bot-side commands (e.g. `/usage`
  via [ccusage](https://github.com/ryoppippi/ccusage)) instantly, with no LLM
  round-trip
- Native command menu registered via `setMyCommands` (type `/` to autocomplete)

**Interactivity as inline buttons**
- Generic permission bridge: out-of-scope tool calls surface as Allow/Deny
  buttons through the SDK's `can_use_tool` hook — works for every tool, no
  per-tool code
- Model picker for voice transcription (`/whisper`)
- Plan-mode approval: `ExitPlanMode` surfaces the plan text with Approve / Keep
  planning buttons; `AskUserQuestion` renders Claude's clarifying questions as
  option buttons and injects the answers back
- `/model` — live model switching via `set_model`, list sourced from the
  official `/v1/models` API (real names, context windows); `/effort` — levels
  discovered from the CLI's own runtime validator (wording-independent probe);
  `/stop` (`/esc`) interrupts the current turn
- `/usage` — subscription limits (5h / weekly / per-model / usage credits)
  straight from the OAuth usage endpoint, matching the official panel

**Media**
- Images travel as native base64 content blocks inside the message (no disk
  files; lifecycle owned by the CLI's transcript retention); other files land
  in `<cwd>/.tgbot/media/` (auto-gitignored) with opportunistic TTL cleanup
- Context-usage warnings: 🟠 at 80% / 🔴 at 90% of the model's real context
  window, computed from per-turn API usage

**Voice messages**
- Local transcription via faster-whisper (default `large-v3-turbo`; switchable
  with `/whisper`, persisted to `.env`), bilingual zh/en including code-switching
- Lazy model download with visible progress; transcript shown as an editable
  `🎤` message so you can verify what was heard

**Message lifecycle**
- Each assistant text segment is delivered immediately as its own message
- Tool activity appears as a live `⏳ Working…` status message, edited in place
  (throttled), which morphs into the next text segment — no extra notifications,
  no token cost
- Quote/reply context: replying to any message forwards the quoted text to the
  agent

**Operations**
- Graceful deploys: touch a flag file; the bot restarts only when all
  conversations are idle, so in-flight replies are never lost
- Restart lifecycle shown as a single silent message: `♻️ Restarting…` edited to
  `✅ Online`
- Stateless by design: all session state lives in the CLI's own jsonl files;
  restarts lose nothing. systemd unit included

**Security model**
- Allowlisted chats only; per-chat profiles (owner: full access with
  `bypassPermissions`; others: scoped read/write with button escalation to the
  owner)
- Session-management commands are owner-gated everywhere

## Architecture

```
Telegram ── python-telegram-bot ── bot.py (router)
                                     │  claude-agent-sdk (one client per chat/topic)
                                     └─ Claude Code CLI ── ~/.claude/projects/*.jsonl
```

The bot is a thin stateless router; the CLI owns sessions, tools, and
persistence. That is where the stability comes from: there is very little here
that can break.

## Setup

```bash
uv sync
cp .env.example .env   # fill in TG_BOT_TOKEN and OWNER_USER_ID at minimum
uv run python bot.py   # or adapt and install tg-claude-bot.service
```

All configuration lives in `.env` (see `.env.example`): owner/guest user ids,
optional group, guest read/write scopes and system prompt, whisper model, media TTL.

## Non-goals

- Attaching to a *live* interactive terminal session (tmux-based bridges like
  [ccbot](https://github.com/six-ddc/ccbot) do this at the cost of scraping
  ANSI output; this bridge trades that for structured SDK events)
- Replicating TUI-only dialogs verbatim (`/config` etc.); their capabilities are
  rebuilt as bot commands where they matter (`/model`, `/effort`, `/usage`)
