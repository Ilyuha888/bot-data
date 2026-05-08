# bot-data — Runtime State for Claude Telegram Bot

This repo holds the runtime state of the [Claude Telegram Bot](https://github.com/Ilyuha888/claude-telegram-bot): scheduler registry, notification history, chat session history, and audit log. It's deliberately a **separate repo** from the bot code so:

- State persists independently of bot upgrades — pull a new bot version, your schedules and reminders survive
- You can back it up or rotate it without touching code
- The bot never auto-pushes secrets here (only structural state — see "What's tracked vs gitignored")

This repo ships with **placeholder schedules and zero personal content**. Clone it, point the bot at it, and the bot starts reading and writing here on first message.

---

## Setup

Point the bot at this directory in your `.env`:

```bash
BOT_DATA_DIR=/path/to/your/bot-data
```

Default if not set: `~/bot-data`. The Docker setup bind-mounts `${BOT_DATA_DIR}` into the container at `/data/bot-data`.

---

## What's in this repo

### Tracked (committed, ships with the scaffold)

| File | Purpose |
|---|---|
| `schedules.json` | Scheduler registry — cron schedules + one-shot reminders |
| `notifications.json` | Delivered notification history — capped at 200 rows |

### Gitignored (created at runtime, never committed)

| File | Purpose |
|---|---|
| `chat-session-history.json` | Last 5 chat sessions (Mode 1) — used by `/resume` and auto-resume after restart |
| `sessions.json` | Mode 2 remote-control work sessions (`/work`, `/sessions`, `/attach`, `/close`) |
| `audit.log` | Append-only audit log — every tool invocation, permission grant, denied request |

The bot creates the gitignored files on first use. They're gitignored because:

- `chat-session-history.json` and `sessions.json` contain conversation context
- `audit.log` contains tool inputs (some of which include text from your messages)

You should still **back this repo up** privately (e.g., to a private remote) — the gitignored files matter for continuity even if they don't ship.

---

## File schemas

### `schedules.json`

```json
{
  "schedules": [
    {
      "id": "daily-focus",
      "cron": "0 9 * * *",
      "tz": "Europe/Moscow",
      "prompt_key": "daily_focus",
      "last_fired": "2026-05-08T08:14:03.437Z",
      "one_shot": false
    },
    {
      "id": "remind-abc12345",
      "cron": "",
      "tz": "Europe/Moscow",
      "prompt_key": "scribe_reminder",
      "last_fired": null,
      "fire_at": "2026-05-09T15:00:00.000Z",
      "one_shot": true,
      "payload": { "reminder_message": "follow up", "note_path": "inbox/2026-05-08-follow-up.md" }
    }
  ]
}
```

| Field | Description |
|---|---|
| `id` | Unique schedule identifier |
| `cron` | 5-field cron expression (`node-cron` syntax). Empty string for one-shots. |
| `tz` | IANA timezone for the cron schedule |
| `prompt_key` | Key into the PROMPTS record in the bot's `scheduler-prompts.ts` (or override file in your vault's `prompts/scheduler/`) |
| `last_fired` | ISO timestamp of the **cron** schedule's last fire (or `null`). Used by boot catch-up logic. **Unused for one-shots — keep `null`.** |
| `fire_at` | ISO timestamp of when a **one-shot** should fire. Required for one-shots; unused for cron schedules. |
| `one_shot` | If `true`, the entry is deleted after firing. Used for `/scribe` reminders. |
| `payload` | (one_shot only) — `{reminder_message, note_path}` for scribe reminders |

> **Schema migration.** Older one-shot rows stored their fire-at timestamp in `last_fired`. The bot migrates them on load (`last_fired → fire_at`, then `last_fired = null`) and persists the rewrite. No manual action required.

### `notifications.json`

```json
{
  "notifications": [
    {
      "id": "uuid",
      "fired_at": "ISO timestamp",
      "prompt_key": "daily_focus",
      "title": "Daily focus",
      "content": "...",
      "telegram_message_id": 12345,
      "status": "delivered"
    }
  ]
}
```

Capped at 200 rows. Do not hand-edit — the bot writes this; manual changes risk being overwritten.

---

## Customizing schedules

Edit `schedules.json` directly. The bot watches the file via `fs.watch` and picks up changes without restart.

**Common edits:**

- Change time: edit the `cron` field (5-field syntax: `min hour day month weekday`)
- Change timezone: edit `tz` to your IANA zone (e.g., `America/New_York`, `Europe/Berlin`)
- Disable a routine: remove its entry
- Add a new routine: append an entry with a `prompt_key` matching one in the bot's PROMPTS record (or a vault override in `prompts/scheduler/<key>.md`)

**Don't:** add a `one_shot: true` entry by hand — those are managed by the `/scribe` skill which writes them with the right schema (including `payload.note_path` for the [Log outcome] callback).

---

## Backup

The simplest backup: push to a private GitHub repo of your own. The scaffold ships with no credentials and no real reminders, so the public repo is safe; your fork after using the bot will have personal state and should stay private.

```bash
cd ~/repos/bot-data
git remote set-url origin git@github.com:YOUR-USERNAME/bot-data-personal.git
git push -u origin main
```

The bot's auto-push (configured in your vault's `CLAUDE.md`) will keep this remote in sync after every state change.

---

## License

MIT — paired with the bot's [LICENSE](https://github.com/Ilyuha888/claude-telegram-bot/blob/main/LICENSE).
