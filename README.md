# bot-data

Runtime state for the Claude Telegram Bot. This repo holds the three JSON files the bot reads and writes at runtime.

## Files

### schedules.json

Scheduler registry. The bot reads this on startup and via `fs.watch` for live updates.

| Field | Description |
|---|---|
| `id` | Unique schedule identifier |
| `cron` | Cron expression (uses `node-cron` syntax) |
| `tz` | Timezone for the cron schedule |
| `prompt_key` | Key into the PROMPTS record in `scheduler-prompts.ts` |
| `last_fired` | ISO timestamp of last fire (or `null`). Used for boot catch-up. |
| `one_shot` | If `true`, the entry is deleted after firing. Used for scribe reminders. |

### notifications.json

Delivered notification history. Capped at 200 rows. Do not edit manually.

### sessions.json

**Never committed.** Contains active conversation history — gitignored. Created automatically by the bot.

## Setup

Point the bot at this directory with `BOT_DATA_DIR` in your `.env`:

```
BOT_DATA_DIR=/path/to/your/bot-data
```

Default if not set: `~/bot-data`.

## Customizing schedules

Edit `schedules.json` — the bot picks up changes without restart via `fs.watch`.

To disable a routine, remove its entry. To change the time, update the `cron` field.
Cron syntax: `minute hour day month weekday` (standard 5-field).
