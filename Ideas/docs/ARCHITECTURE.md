# Architecture

Visual companion to [`plan.md`](../plan.md). Read `plan.md` for decisions and rationale; read this for diagrams and data flows.

---

## 1. System Context (C4 Level 1)

        ┌───────────────────┐
        │       User        │
        │  (you, in #chat)  │
        └─────────┬─────────┘
                  │
           Discord client
                  │
                  ▼
        ┌───────────────────┐
        │   AI Assistant    │
        │     (Pi 5)        │
        └─┬───────┬───────┬─┘
          │       │       │
  ┌───────▼─┐ ┌───▼───┐ ┌─▼──────┐
  │OpenRouter│ │Google │ │ Notion │
  │ LLM+TTS  │ │  Cal  │ │   DB   │
  └──────────┘ └───────┘ └────────┘
          │       │
  ┌───────▼───────▼──────┐
  │  Discord │ Dropbox  │
  │  Resend  │          │
  └──────────────────────┘

**External dependencies:**

- **Discord** — sole user interface (input + output)
- **OpenRouter** — LLM inference (`gpt-4o-mini`) + TTS
- **Google Calendar** — event read/write for context
- **Notion** — task mirror (read-only from user perspective)
- **Resend** — high-priority email alerts only
- **Dropbox** — encrypted backup storage

---

## 2. Container Diagram (C4 Level 2)


┌─────────────────────────────────────────────────────────────────┐
│ Raspberry Pi 5 │
│ │
│ ┌──────────────────┐ ┌──────────────────────────┐ │
│ │ ZeroClaw │ │ integrations_api │ │
│ │ (Rust binary) │────▶│ (Flask + gunicorn) │ │
│ │ systemd svc │ HTTP│ 127.0.0.1:7878 │ │
│ │ │ │ systemd svc │ │
│ │ - Discord │ └────┬────────────────┬────┘ │
│ │ gateway │ │ │ │
│ │ - LLM calls │ │ │ │
│ │ - memory.db │ ▼ ▼ │
│ └──────────────────┘ ┌──────────┐ ┌──────────────┐ │
│ │reminders │ │ pending_* │ │
│ │ .db │ │ queue tables │ │
│ │ (WAL) │ │ │ │
│ └────▲─────┘ └──────────────┘ │
│ │ │
│ ┌──────────────────────┐ │ │
│ │ reminder_daemon │─────┘ │
│ │ (Python) │ reads pending │
│ │ systemd svc │ │
│ │ 60s poll │ │
│ └──────────────────────┘ │
│ │
│ ┌──────────────────────┐ │
│ │ rclone_watch.sh │ │
│ │ inotify, debounced │ │
│ │ systemd svc │ │
│ └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘


---

## 3. Key Data Flows

### 3.1 User Sends a Message


User in #chat ──▶ Discord ──▶ ZeroClaw gateway
│
▼
ZeroClaw loads context from memory.db
│
▼
POST to OpenRouter (gpt-4o-mini)
│
▼
Parse JSON response: { reply_text, actions[], tts_requested }
│
┌─────────┼─────────┐
▼ ▼ ▼
Post reply Invoke Optional TTS
in #chat skills → audio attachment
│
▼
HTTP to integrations_api → DB writes, calendar, notion


### 3.2 Reminder Fires


Time passes...
│
▼
reminder_daemon polls (every 60s):
│
▼
SELECT * FROM reminders WHERE status='pending' AND fire_at_utc <= NOW()
│
├─ 0 rows ─▶ update daemon_health.last_poll_utc, sleep 60s
│
└─ 1+ rows ──▶ For each (batched if same minute):
│
▼
POST to Discord REST → #reminders channel
│
├─ success ──▶ UPDATE status='fired' (if rrule: compute next, reset pending)
│
└─ fail ─────▶ INSERT pending_deliveries (exponential backoff, after 5x: status='failed' + alert)


### 3.3 Context-Aware Reminder Creation


User: "Remind me to call dentist at 9am tomorrow"
│
▼
LLM emits actions: [{ type: "check_calendar" }, { type: "create_reminder" }]
│
▼
ZeroClaw skill: calendar_check → GET /calendar/check?start=...&end=...
│
▼
Flask → Google Calendar API
│
▼
Conflict found? → reply_text becomes confirmation question
needs_user_confirmation: true, reminder NOT yet inserted
│
▼
User confirms "yes" → ZeroClaw re-invokes create_reminder
→ POST /reminder → INSERT into reminders.db → Notion upsert → Discord confirmation


### 3.4 Backup Flow


File change in ~/.zeroclaw/ or ~/reminders.db
│
▼
inotifywait fires close_write event
│
▼
Debounce check: NOW - LAST_SYNC < 600s? ──▶ skip
│
▼
sqlite3 .backup → /tmp/memory.db (atomic snapshot)
sqlite3 .backup → /tmp/reminders.db
│
▼
rclone copy /tmp/.db ──▶ dropbox-crypt:/db/ (encryption transparent)
│
▼
rm /tmp/.db
LAST_SYNC = NOW


---

## 4. Trust Boundaries


┌───────────────────────────────┐
│ TRUSTED: Local Pi (127.0.0.1) │
│ │
│ ZeroClaw ←──HTTP+token──→ Flask API │
│ reminder_daemon ──reads── SQLite │
└───────────────────────────────┘
│
▼
┌───────────────────────────────┐
│ UNTRUSTED: Internet │
│ │
│ OpenRouter │ Discord │ Google │ Notion │
│ Resend │ Dropbox │ │
└───────────────────────────────┘


**Security properties:**

- Flask binds to `127.0.0.1` only — never exposed to LAN
- All requests require `X-Auth-Token` header (defense in depth)
- Secrets in `/etc/zeroclaw/secrets.env` with `0600 root:pi`
- All Dropbox files encrypted via `rclone crypt` before upload
- No inbound ports open on the Pi (except SSH if configured)

---

## 5. Failure Isolation

| Component fails | Impact | What still works |
|---|---|---|
| ZeroClaw | Cannot chat in #chat | Reminders still fire; backups still run |
| reminder-api | Cannot create new reminders | Existing reminders fire; chat works (degraded) |
| reminder-daemon | Reminders don't fire | Chat works; new reminders are accepted but queued |
| rclone-backup | No new backups | Everything else works; last backup remains valid |
| Internet down | Chat + reminders queue | Local state preserved; resumes on reconnect |
| OpenRouter outage | Cannot chat | Reminders still fire (no LLM needed for firing) |
| Discord down | Cannot send or receive | Daemon queues sends; resumes on reconnect |
| Google Cal API down | New reminders skip conflict check | Local reminders + chat unaffected |
| Notion down | New reminders queue for Notion | Local reminders + chat unaffected |
| SQLite locked | Brief retry (5s timeout) | Resolves automatically |
| Power loss | UPS triggers graceful shutdown | WAL journal recovers on boot |

---

## 6. Data Lifecycle


Birth Active Archive Death
│ │ │ │
▼ ▼ ▼ ▼
User says Stored in After 6 Hard delete
"remind me" reminders.db months: (never, by policy)

Notion + memory_archive.db
(opt) GCal + still in backup

- Backup every 10 min (debounced) → Dropbox encrypted
- Weekly cron → `verify_backup.sh` → integrity check
- Quarterly → VACUUM + archive → smaller live DB

---

## 7. Component Versions Map

| Component | Version | Last upgraded |
|---|---|---|
| Raspberry Pi OS | Bookworm 64-bit | YYYY-MM-DD |
| Python | 3.11.x | YYYY-MM-DD |
| ZeroClaw | x.y.z | YYYY-MM-DD |
| rclone | x.y.z
