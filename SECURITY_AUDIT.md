<div align="center">

# 🛡️ Security Audit

**Audited by Claude (Fable 5) — the model this bot bridges, reviewing its own bridge.**

![Date](https://img.shields.io/badge/date-2026--07--22-blue.svg)
![Passes](https://img.shields.io/badge/review%20passes-3-blue.svg)
![Findings](https://img.shields.io/badge/findings-10%20fixed%20·%200%20open-blue.svg)

</div>

**Scope:** full line-by-line review of `bot.py`, `tg-claude-bot.service`, setup docs,
and `.env` handling — three passes: two review-and-fix rounds, then a final-state
verification round that (a) mechanically asserted every prior fix is present in the
shipped tree and (b) re-reviewed the code the fixes themselves introduced.

> Honest framing: this is an **AI self-audit**, not a third-party professional
> audit. Its value is a complete, systematic pass over every trust boundary by
> a reviewer that understands both sides of the bridge. Treat it as a strong
> baseline, not a certification.

## 🎯 Threat model

- **T1 — Unauthorized Telegram access:** strangers messaging the bot; forged
  callback presses; guests exceeding their scope.
- **T2 — Local machine attackers:** other Unix users on the host reading
  secrets or influencing the bot.
- **T3 — Prompt-level attacks:** sender impersonation inside message text,
  injection via quoted replies, files, or fetched web content.
- **T4 — Operational failure:** deploys or restarts losing state or silently
  breaking.

## ✅ What holds (verified)

| Control | Notes |
|---|---|
| Chat allowlist | Only owner DM, configured guest ids, and one configured group are served; ids are Telegram-authenticated, unspoofable. `TARGET_GROUP_ID=0` default disables groups. |
| Callback authorization | Every inline button press is identity-checked server-side. Session/model/effort/whisper buttons: owner only. Permission/plan buttons: owner only. Question buttons: owner or the user who triggered the turn. Forged `callback_data` from modified clients hits the same checks; `bt:` tokens are 40-bit random and single-use. |
| Guest sandbox | No Bash. Read/Write tools path-checked against configured dirs after `resolve()` (symlink- and `..`-safe). Absolute or traversing glob patterns are not auto-allowed. Pathless Glob/Grep only allowed when read dirs are configured; guest cwd falls back to `/tmp`, never `$HOME`. Out-of-scope calls: deny, or Allow/Deny escalation to the owner when the owner asked. |
| Escalation integrity | Scope escalation keys off the bot-recorded sender id, not message text — prompt-level impersonation cannot unlock it. Queued-message turns attribute to the least-privileged sender in the batch. |
| Secrets | All secrets in `.env` (chmod 600); systemd unit template is secret-free (unit files are world-readable). OAuth token read from the CLI's own credential store, never logged, never echoed to chat. |
| Shell surface | No user input ever reaches a shell: bot-side commands are a fixed string table (owner-only); CLI probes use argv arrays; session ids go through `glob` against a fixed root, not paths. |
| Local attackers (T2) | Restart flag/notice files in `/tmp` are honored only if owned by the bot's uid. `umask 077` keeps transient voice/image files and media dirs private. |
| Error hygiene | Exception details (internal paths) go to the owner only; guests get a generic message. |
| Ops | Graceful deploys wait for idle conversations; if no sudo rule exists the bot exits nonzero and systemd's `Restart=on-failure` completes the restart. Stateless design: a crash or restart loses no session data. |

## 🔧 Findings fixed during this audit

| # | Severity | Finding | Fix |
|---|---|---|---|
| 1 | 🔴 High (doc) | Service template invited placing `CLAUDE_CODE_OAUTH_TOKEN` in the world-readable systemd unit | Template made secret-free; token documented into `.env` (600) |
| 2 | 🟠 Medium | Guest cwd fell back to `$HOME`; pathless Grep could search it; absolute glob patterns could escape scoped dirs | `/tmp` fallback; pathless Glob/Grep gated on configured read dirs; absolute/`..` patterns excluded from auto-allow |
| 3 | 🟠 Medium | `/tmp` restart flag honored regardless of owner — any local user could trigger restarts; missing sudo rule made deploys silently no-op | Flag/notice ownership check; nonzero exit fallback for supervised restart |
| 4 | 🟡 Low | Turns drained from the queue inherited the lock-holder's escalation privilege | Least-privileged sender attribution for drained batches |
| 5 | 🟡 Low | Error replies leaked internal paths to guest DMs; `/status` leaked cwd + session id to guests | Owner-only details |
| 6 | 🟡 Low | Transient voice/image files in `/tmp` were world-readable | `umask 077` at startup |
| 7 | 🟡 Low | Guests couldn't answer `AskUserQuestion` buttons (owner-only check), stalling their turns | Question buttons pressable by the turn's initiator |
| 8 | ⚪ Info | Sender-prefix impersonation inside message bodies could confuse the model (never the permission layer) | Guest system prompt pins the outermost bridge prefix as the only authority |
| 9 | 🟡 Low | Button callbacks accepted an unvalidated index: a prompted user forging `callback_data` could crash the turn (`labels[idx]` out of range) | Option count stored per prompt; index parsed and bounds-checked before delivery |
| 10 | ⚪ Info | Round-2 glob gate over-matched: Grep *regex* patterns containing `..` lost auto-allow (false positive, availability only) | Gate scoped to Glob patterns; Grep regexes exempt |

**Final-state verification:** after the last fix, all 10 findings were re-checked
against the shipped tree with mechanical assertions (13 checks, all passing), and
the fix-introduced code paths were re-reviewed. No known unfixed issues remain
within the threat model above.

## ⚖️ Accepted risks (by design, documented)

- **Owner profile auto-allows every tool** except plan/question dialogs. The
  owner is operating their own machine with full shell anyway; the bridge adds
  no privilege it doesn't already have. Don't run the bot as root.
- **Guests get WebFetch/WebSearch** without per-call prompts. In an
  injection scenario this is an exfiltration channel for data already inside
  the guest's scoped context; scope your guest read dirs accordingly.
- **Every member of the configured group** is served with the guest profile.
  Only add groups whose membership you control.
- **No rate limiting.** Allowed users can queue expensive turns (LLM tokens,
  whisper CPU). Acceptable for a personal allowlisted bot; add throttling
  before widening the allowlist.
- **Passthrough `/commands` are available to guests** in their own scoped
  sessions; tool use inside them remains permission-gated.

## 🚫 Out of scope

The Claude Code CLI, the Agent SDK, python-telegram-bot, and Telegram's own
transport security are trusted dependencies. Voice models come from the
faster-whisper/HF distribution chain (owner-selected only).
