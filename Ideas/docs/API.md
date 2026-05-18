# Flask Tool API Reference

The `integrations_api.py` Flask app exposes the local tool surface that
ZeroClaw skills call. **Bound to `127.0.0.1:7878` only** — not reachable from
LAN or internet.

---

## Authentication

Every request requires:
X-Auth-Token: <FLASK_AUTH_TOKEN>



The token is a 32-byte hex string from `/etc/zeroclaw/secrets.env`.
Requests without the header (or with a wrong token) return `401 Unauthorized`.

---

## Content Type

- Requests with bodies: `Content-Type: application/json`
- All responses: `application/json` (except `/tts/cached` which streams a file)

---

## Error Format

All errors return:

```json
{
  "error": "machine_readable_code",
  "message": "Human-readable description",
  "details": { ... }
}
Standard Error Codes
HTTP	error	Meaning
400	invalid_payload	Body missing required field or schema mismatch
400	invalid_datetime	fire_at_utc not valid ISO-8601 or in the past
400	invalid_rrule	RRULE string failed to parse
401	unauthorized	Missing or wrong X-Auth-Token
404	not_found	Reminder / event ID does not exist
409	conflict	Duplicate detected (idempotency)
500	internal_error	Unhandled exception; check logs
502	upstream_error	Google / Notion / Discord returned an error
503	upstream_unavailable	Service temporarily down; queued for retry
Endpoints
Reminders
POST /reminder
Create a new reminder.

Request:


{
  "content": "Call the dentist",
  "fire_at_utc": "2026-05-19T08:00:00Z",
  "rrule": null,
  "priority": "normal",
  "add_to_calendar": false
}
Field	Type	Required	Notes
content	string	yes	1–500 chars
fire_at_utc	string	yes	ISO-8601 with Z; must be future
rrule	string|null	no	iCalendar RRULE (FREQ=...;BYDAY=...)
priority	string	no	normal (default) or high
add_to_calendar	bool	no	If true, also creates Google Calendar event
Response 201:


{
  "id": 42,
  "fire_at_local": "2026-05-19 09:00 BST",
  "google_event_id": null,
  "notion_page_id": "abc-123-def",
  "queued_for_notion": false
}
Errors: invalid_payload, invalid_datetime, invalid_rrule

GET /reminder/list
List reminders.

Query params:

status (optional): pending (default) | fired | cancelled | failed | all
limit (optional): default 100, max 500
Response 200:


{
  "count": 3,
  "reminders": [
    {
      "id": 42,
      "content": "Call the dentist",
      "fire_at_utc": "2026-05-19T08:00:00Z",
      "fire_at_local": "2026-05-19 09:00 BST",
      "status": "pending",
      "priority": "normal",
      "rrule": null,
      "google_event_id": null,
      "notion_page_id": "abc-123-def"
    }
  ]
}
DELETE /reminder/<id>
Cancel a reminder. Cascades:

Sets status='cancelled' in SQLite
Deletes linked Google Calendar event (if any)
Updates linked Notion page status
Response 200:


{
  "id": 42,
  "cancelled": true,
  "calendar_event_deleted": false,
  "notion_updated": true
}
Errors: not_found

Calendar
GET /calendar/check
List events in a window. Used for context-aware reminder creation.

Query params:

start (required): ISO-8601 UTC
end (required): ISO-8601 UTC
Response 200:


{
  "events": [
    {
      "id": "gcal-event-id-xyz",
      "summary": "Team standup",
      "start_utc": "2026-05-19T08:00:00Z",
      "end_utc": "2026-05-19T08:30:00Z",
      "description": null
    }
  ]
}
Errors: upstream_unavailable (Google API down)

POST /calendar/event
Create a Google Calendar event.

Request:


{
  "summary": "Dentist appointment",
  "start_utc": "2026-05-19T09:00:00Z",
  "end_utc": "2026-05-19T09:30:00Z",
  "description": "Created by AI assistant"
}
Response 201:


{
  "id": "gcal-event-id-xyz",
  "html_link": "https://calendar.google.com/calendar/event?eid=..."
}
If Google API is down: returns 503 with error: upstream_unavailable and queues the op in pending_calendar_ops for retry.

DELETE /calendar/event/<id>
Delete a Google Calendar event by Google's event ID.

Response 200:


{ "deleted": true }
System
GET /status
System health snapshot. Used by /status Discord command and external monitors.

Response 200:


{
  "timestamp_utc": "2026-05-18T14:23:01Z",
  "daemon": {
    "last_poll_utc": "2026-05-18T14:22:55Z",
    "seconds_since_last_poll": 6,
    "health": "healthy",
    "last_fire_utc": "2026-05-18T14:00:00Z",
    "ntp_offset_sec": 0.012
  },
  "queues": {
    "pending_deliveries": 0,
    "pending_calendar_ops": 0,
    "pending_notion_ops": 1
  },
  "reminders": {
    "pending": 12,
    "fired_today": 4,
    "failed_total": 0
  },
  "uptime_sec": 432189,
  "version": "0.3.1"
}
Health values:

healthy — last poll < 3 min ago
degraded — 3–15 min ago
critical — > 15 min ago (also triggers Resend alert)
GET /tts/cached
Look up a pre-rendered audio file. Used by speak_cached skill.

Query params:

key (required): cache key, e.g. done_reminder_set
Response 200:

Content-Type: audio/ogg
Body: raw OGG Opus file bytes
Response 404: if key doesn't exist in ~/assistant/audio_cache/

Idempotency
POST /reminder is not idempotent. Each call creates a new row.
Caller (ZeroClaw skill) should not retry without checking, or it duplicates.
DELETE /reminder/<id> IS idempotent. Cancelling an already-cancelled
reminder returns 200 with cancelled: true.
Rate Limits
No internal rate limit (single-user, localhost). External calls inside the
endpoint are rate-limited:

External	Limit	Handling
Google Calendar	1M req/day	Won't hit; no special handling
Notion	3 req/sec	400 ms spacing + 429 retry with backoff
Discord (daemon)	5 req/2s per channel	1s sleep between sends + tenacity
OpenRouter	Account-dependent	Surfaced as 502 to caller
Versioning
Currently unversioned (single client, no third-party consumers).
If a breaking change is made, document in CHANGELOG.md and update both
the API and all calling skills in the same commit.

Example: curl Smoke Test

TOKEN=$(grep FLASK_AUTH_TOKEN /etc/zeroclaw/secrets.env | cut -d= -f2)
BASE=http://127.0.0.1:7878

# Health
curl -s -H "X-Auth-Token: $TOKEN" $BASE/status | jq

# Create reminder for 2 minutes from now
FIRE=$(date -u -d "+2 minutes" +"%Y-%m-%dT%H:%M:%SZ")
curl -s -X POST -H "X-Auth-Token: $TOKEN" -H "Content-Type: application/json" \
  -d "{\"content\":\"smoke test\",\"fire_at_utc\":\"$FIRE\"}" \
  $BASE/reminder | jq

# List pending
curl -s -H "X-Auth-Token: $TOKEN" $BASE/reminder/list | jq

# Cancel
curl -s -X DELETE -H "X-Auth-Token: $TOKEN" $BASE/reminder/42 | jq


---

# 📄 5. `.env.example`

```bash
# ============================================================================
# AI Assistant — Environment Variables Template
# ============================================================================
# 
# Copy this file to /etc/zeroclaw/secrets.env on the Pi, then fill in real values.
#
#   sudo mkdir -p /etc/zeroclaw
#   sudo cp .env.example /etc/zeroclaw/secrets.env
#   sudo nano /etc/zeroclaw/secrets.env       # fill in values
#   sudo chown root:pi /etc/zeroclaw/secrets.env
#   sudo chmod 0600 /etc/zeroclaw/secrets.env
#
# Loaded automatically by systemd via `EnvironmentFile=/etc/zeroclaw/secrets.env`.
# NEVER commit the real file to git. NEVER share these values.
# ============================================================================

# ----------------------------------------------------------------------------
# OpenRouter (LLM + TTS)
# ----------------------------------------------------------------------------
# Get from: https://openrouter.ai/keys
OPENROUTER_API_KEY=sk-or-v1-REPLACE_ME

# ----------------------------------------------------------------------------
# Discord
# ----------------------------------------------------------------------------
# Bot token from: https://discord.com/developers/applications → Bot → Reset Token
DISCORD_BOT_TOKEN=REPLACE_ME

# Guild (server) ID — right-click your server name with Developer Mode on
DISCORD_GUILD_ID=000000000000000000

# Channel IDs — right-click each channel with Developer Mode on
DISCORD_CHANNEL_CHAT=000000000000000000           # main conversation
DISCORD_CHANNEL_REMINDERS=000000000000000000      # reminder fires land here
DISCORD_CHANNEL_ALERTS=000000000000000000         # system errors / health

# ----------------------------------------------------------------------------
# Resend (email alerts only)
# ----------------------------------------------------------------------------
# Get from: https://resend.com/api-keys
RESEND_API_KEY=re_REPLACE_ME
RESEND_FROM=alerts@yourdomain.com                 # must be verified in Resend
RESEND_TO=you@yourdomain.com                      # where alerts go

# ----------------------------------------------------------------------------
# Internal auth (Flask <-> ZeroClaw)
# ----------------------------------------------------------------------------
# Generate with: openssl rand -hex 32
FLASK_AUTH_TOKEN=REPLACE_ME_WITH_RANDOM_64_HEX_CHARS

# ----------------------------------------------------------------------------
# Google Calendar
# ----------------------------------------------------------------------------
# Setup steps:
#   1. https://console.cloud.google.com/ → create project
#   2. Enable Google Calendar API
#   3. APIs & Services → Credentials → Create OAuth client ID (Desktop app)
#   4. Download client JSON; copy values below
#   5. Run scripts/setup_google_oauth.py → produces GOOGLE_REFRESH_TOKEN
GOOGLE_CLIENT_ID=REPLACE_ME.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=REPLACE_ME
GOOGLE_REFRESH_TOKEN=REPLACE_ME
GOOGLE_CALENDAR_ID=primary                        # or specific calendar ID

# ----------------------------------------------------------------------------
# Notion
# ----------------------------------------------------------------------------
# Setup steps:
#   1. https://www.notion.so/my-integrations → New integration (internal)
#   2. Copy the secret
#   3. Create a database in Notion with the schema in plan.md § 9.2
#   4. Share the database with your integration (⋯ → Connections → Add)
#   5. Copy the database ID from the URL (the 32-char hex before ?v=)
NOTION_API_KEY=secret_REPLACE_ME
NOTION_DATABASE_ID=REPLACE_ME

# ----------------------------------------------------------------------------
# Preferences
# ----------------------------------------------------------------------------
TTS_VOICE=alloy                                   # OpenRouter TTS voice
TTS_MODEL=openai/gpt-4o-mini-tts-2025-12-15       # confirm current dated slug
LLM_MODEL=openai/gpt-4o-mini
USER_TIMEZONE=Europe/London

# ----------------------------------------------------------------------------
# Paths (rarely change)
# ----------------------------------------------------------------------------
ASSISTANT_HOME=/home/pi/assistant
REMINDERS_DB=/home/pi/reminders.db
ZEROCLAW_HOME=/home/pi/.zeroclaw
AUDIO_CACHE_DIR=/home/pi/assistant/audio_cache

# ----------------------------------------------------------------------------
# Tunables (sane defaults; only change if you know why)
# ----------------------------------------------------------------------------
DAEMON_POLL_SECONDS=60
BACKUP_DEBOUNCE_SECONDS=600
TTS_WORD_CAP=300
DISCORD_MESSAGE_CAP=1900
NOTION_RETRY_INTERVAL_SECONDS=300
ALERT_DAILY_CAP=20
ALERT_DEDUP_MINUTES=60
