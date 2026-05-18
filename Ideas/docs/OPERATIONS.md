# Operations Runbook

What to do when things break, what each alert means, and routine maintenance.

This document is for **the person who runs the system** (you). It assumes the
system is already built — for setup see [`INSTALL.md`](../INSTALL.md), and for
disaster recovery see [`RECOVERY.md`](../RECOVERY.md).

---

## Table of Contents

1. [Daily checks (30 seconds)](#1-daily-checks)
2. [Alert dictionary](#2-alert-dictionary)
3. [Common problems & fixes](#3-common-problems--fixes)
4. [Maintenance schedule](#4-maintenance-schedule)
5. [Key rotation](#5-key-rotation)
6. [Debugging recipes](#6-debugging-recipes)
7. [Performance tuning](#7-performance-tuning)
8. [How-to recipes](#8-how-to-recipes)

---

## 1. Daily Checks

Do this once a day, takes 30 seconds:

In Discord `#chat`:
/status



Expect: all green, daemon `last_poll` < 3 min, queues at 0.

If anything is yellow or red, jump to § 2.

---

## 2. Alert Dictionary

Every alert sent to `#alerts` (and optionally Resend) has an `alert_key`. This
table maps keys → meaning → action.

| `alert_key` | Severity | Meaning | Immediate action |
|---|---|---|---|
| `daemon_poll_stale` | 🔴 critical | Daemon hasn't polled in > 15 min | `systemctl restart reminder-daemon`; check logs |
| `daemon_channel_404` | 🔴 critical | A configured Discord channel doesn't exist | Verify channel IDs in `/etc/zeroclaw/secrets.env` |
| `discord_token_revoked` | 🔴 critical | Discord 401 on bot token | Regenerate token in Dev Portal; update secrets |
| `openrouter_quota` | 🟡 high | OpenRouter returning 429 / payment required | Top up balance at openrouter.ai |
| `openrouter_down` | 🟡 high | OpenRouter unreachable > 5 min | Check status.openrouter.ai; wait |
| `google_oauth_expired` | 🟡 high | Refresh token rejected | Re-run `scripts/setup_google_oauth.py` |
| `notion_auth_failed` | 🟡 high | Notion 401 | Check integration token; verify DB still shared |
| `rclone_sync_failed_3x` | 🟡 high | 3 consecutive backup failures | `rclone config reconnect dropbox-raw:` |
| `backup_integrity_failed` | 🔴 critical | Weekly verify found corrupt DB | Investigate — possibly restore from older backup |
| `ntp_drift` | 🟡 high | Clock offset > 10s | `sudo chronyc makestep`; check network |
| `crashloop_<service>` | 🔴 critical | Service restarted > 3x in 5 min | `journalctl -u <svc> -n 100` for root cause |
| `reminder_send_failed` | 🟢 info | A single reminder failed 5x to send | Check Discord status; daemon retries automatically |
| `calendar_orphan` | 🟢 info | Google event linked to deleted reminder | Daily reconciliation will clean up |
| `temp_high` | 🟡 high | CPU > 75°C sustained | Check fan, dust, ambient temp |
| `disk_low` | 🟡 high | < 1 GB free on `/` | `du -sh ~/.zeroclaw ~/assistant /tmp` to find culprit |
| `ups_battery_low` | 🟡 high | Waveshare UPS HAT (E) battery < 25% | Investigate power supply |
| `ups_on_battery` | 🟢 info | Mains lost, running on battery | If sustained, plan shutdown |

---

## 3. Common Problems & Fixes

### "I sent a message but ZeroClaw didn't reply"

```bash
# Is ZeroClaw alive?
sudo systemctl status zeroclaw

# Recent logs
sudo journalctl -u zeroclaw -n 50

# Common causes:
# - OpenRouter outage: curl -I https://openrouter.ai
# - Wrong channel: ZeroClaw only listens on $DISCORD_CHANNEL_CHAT
# - Token revoked: check Discord Developer Portal
"Reminder didn't fire"

# Check daemon
sudo systemctl status reminder-daemon
curl -s -H "X-Auth-Token: $TOKEN" http://127.0.0.1:7878/status | jq .daemon

# Check the row directly
sqlite3 ~/reminders.db "SELECT id, content, fire_at_utc, status FROM reminders WHERE id=<id>;"

# Check Discord delivery queue
sqlite3 ~/reminders.db "SELECT * FROM pending_deliveries;"

# Force-fire (for debugging only)
curl -X POST -H "X-Auth-Token: $TOKEN" http://127.0.0.1:7878/_admin/fire_reminder/<id>
"Backup not running"

sudo systemctl status rclone-backup
sudo journalctl -u rclone-backup -n 50

# Trigger manually
sudo systemctl restart rclone-backup
touch ~/reminders.db   # forces inotify event (after debounce window)

# Check last sync time in Dropbox
rclone lsl dropbox-crypt:/db/ | head
"Notion isn't updating"

# Check queue depth
curl -s -H "X-Auth-Token: $TOKEN" http://127.0.0.1:7878/status | jq .queues.pending_notion_ops

# Check the queue contents
sqlite3 ~/reminders.db "SELECT * FROM pending_notion_ops;"

# Test Notion connectivity
cd ~/assistant && venv/bin/python -c "
import os
from notion_client import Client
c = Client(auth=os.environ['NOTION_API_KEY'])
print(c.databases.retrieve(os.environ['NOTION_DATABASE_ID'])['title'])
"

# If db not shared with integration → add via Notion UI
# If credentials wrong → check secrets file
"Calendar conflicts not being detected"

# Direct test
curl -s -H "X-Auth-Token: $TOKEN" \
  "http://127.0.0.1:7878/calendar/check?start=$(date -u +%Y-%m-%dT00:00:00Z)&end=$(date -u -d '+1 day' +%Y-%m-%dT00:00:00Z)" | jq

# If 503 → Google API issue or OAuth expired
# If empty list but you know there are events → wrong GOOGLE_CALENDAR_ID
"Pi feels slow / temperature warning"

vcgencmd measure_temp                # current
vcgencmd get_throttled               # 0x0 = healthy; non-zero = recent throttling
top                                  # what's hot

# Common culprits:
# - rclone running on every file change (debounce broken?)
# - SQLite VACUUM mid-query (check cron)
# - Excessive logging (rotate journald)
4. Maintenance Schedule
Daily (automated)
Watchdog cron checks restart counts (every 5 min)
Calendar orphan reconciliation (daily at 03:30)
NTP drift check on boot
Weekly (automated)
Backup integrity verification (Sunday 04:00)
Review #alerts channel for accumulated info-level alerts
Monthly (manual — set calendar reminder)
 Top up OpenRouter if needed (/status in chat shows monthly spend)
 rclone config reconnect dropbox-raw: (refresh Dropbox token)
 Review docs/BUDGET.md; update with actuals
 Check Pi temps under load (vcgencmd measure_temp during a chat)
 Check disk usage (df -h)
 sudo apt update && sudo apt upgrade -y && sudo reboot (security updates)
Quarterly (automated + manual)
[auto] Memory archive + VACUUM (1st of Jan/Apr/Jul/Oct at 03:00)
[manual] Review and rotate non-critical API keys (OpenRouter, Resend)
[manual] Update Python deps: pip list --outdated; test in branch first
Annually (manual)
 Test-restore drill using only RECOVERY.md and a spare device
 Rotate critical API keys (Discord bot token, Notion)
 Update RECOVERY.md paper copy
 Replace UPS battery if degraded
 Review plan.md — what should change for v2?
5. Key Rotation
Standard procedure for ANY key (except rclone crypt — never rotate):


# 1. Generate new key in provider dashboard
# 2. Edit secrets
sudo nano /etc/zeroclaw/secrets.env
# 3. Restart affected services
sudo systemctl restart zeroclaw reminder-api reminder-daemon
# 4. Smoke test
# 5. Revoke old key in provider
# 6. Update RECOVERY.md (both paper and password manager)
# 7. Update § 8 rotation log in RECOVERY.md
Rotation-specific notes
Key	Notes
rclone crypt password	NEVER ROTATE — destroys backup
DISCORD_BOT_TOKEN	All gateway clients disconnect; ZeroClaw + daemon will reconnect with new token
GOOGLE_REFRESH_TOKEN	Run scripts/setup_google_oauth.py to get a new one
NOTION_API_KEY	Don't forget to re-share the database with the new integration if you regenerate the whole integration
OPENROUTER_API_KEY	Old key can be revoked immediately — no graceful handoff needed
FLASK_AUTH_TOKEN	Must match between Flask and ZeroClaw skill config; rotate atomically
6. Debugging Recipes
Get the last 100 entries from every service

sudo journalctl -n 100 \
  -u zeroclaw -u reminder-api -u reminder-daemon -u rclone-backup \
  --output=short-iso
Filter logs by event type (structured JSON)

sudo journalctl -u reminder-daemon -o json --since "1 hour ago" \
  | jq 'select(.event=="reminder_fired")'

sudo journalctl -u reminder-api -o json --since "1 day ago" \
  | jq 'select(.level=="ERROR")'
Inspect SQLite under load (safe — read-only)

sqlite3 ~/reminders.db "
SELECT status, COUNT(*) FROM reminders GROUP BY status;
SELECT * FROM daemon_health;
SELECT COUNT(*) FROM pending_deliveries;
SELECT COUNT(*) FROM pending_notion_ops;
"
Test Discord delivery directly (bypassing daemon)

source /etc/zeroclaw/secrets.env
curl -X POST \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"test from operations runbook"}' \
  "https://discord.com/api/v10/channels/$DISCORD_CHANNEL_REMINDERS/messages"
See what ZeroClaw sees (memory inspection)

# Read-only; safe during operation
sqlite3 ~/.zeroclaw/memory.db ".tables"
sqlite3 ~/.zeroclaw/memory.db "SELECT COUNT(*) FROM <main_memory_table>;"
Trace one specific request
Use a unique marker in your test message: "test-abc123 hello"


sudo journalctl --since "5 minutes ago" -o cat \
  -u zeroclaw -u reminder-api -u reminder-daemon \
  | grep abc123
7. Performance Tuning
If the system gets slow as data accumulates:

Symptoms & responses
Symptom	Likely cause	Fix
Chat replies > 5s	OpenRouter slow OR ZeroClaw memory huge	Quarterly archive runs already; force one manually
Reminders fire late	Daemon poll lag from DB contention	Check pragma busy_timeout; review queue sizes
Backups taking minutes	DBs grew large	Run quarterly archive + VACUUM early
Pi temp climbing	Fan failure or dust	Clean fan; check ambient temp
SD/NVMe write errors in dmesg	Disk wear	Time to replace NVMe; restore from backup
Force quarterly maintenance early

sudo /home/pi/assistant/scripts/quarterly_maintenance.sh
Vacuum reminders DB manually

# Stop daemon first to avoid lock contention
sudo systemctl stop reminder-daemon
sqlite3 ~/reminders.db "VACUUM;"
sudo systemctl start reminder-daemon
8. How-To Recipes
How do I add a new pre-rendered TTS phrase?
Edit scripts/generate_audio_cache.py — add ("new_key", "Phrase text")
Run venv/bin/python scripts/generate_audio_cache.py --only new_key
Edit docs/PROMPTS.md — add new_key to the LLM's allowed list
Restart ZeroClaw: sudo systemctl restart zeroclaw
Test
How do I change the LLM system prompt?
Edit the prompt source (location depends on your ZeroClaw setup; typically in ~/.zeroclaw/workspace/)
Update docs/PROMPTS.md with the new version + date + reason
Restart ZeroClaw
Test with at least 3 different message types
Commit prompt change to git with a meaningful message
Watch #alerts for 24h — silent breakage from prompt change is the #1 risk
How do I add a new Discord slash command?
This is a ZeroClaw skill addition. Outline:

Add Flask endpoint in integrations_api.py
Add skill in ~/.zeroclaw/workspace/skills/
Update docs/SKILLS.md and docs/API.md
Register the slash command in Discord Dev Portal (or via API)
Restart reminder-api and zeroclaw
How do I pause the assistant temporarily?

# Stop accepting new messages but keep firing existing reminders
sudo systemctl stop zeroclaw

# Stop firing reminders too (they'll fire when restarted)
sudo systemctl stop reminder-daemon
To resume:


sudo systemctl start reminder-daemon zeroclaw
How do I drain all queues before maintenance?

# Watch queues
watch -n 5 'curl -s -H "X-Auth-Token: $TOKEN" http://127.0.0.1:7878/status | jq .queues'

# Wait until all 0, then proceed
How do I bulk-export reminders?

sqlite3 -header -csv ~/reminders.db \
  "SELECT id, content, fire_at_utc, status, rrule FROM reminders;" \
  > ~/reminders_export_$(date +%F).csv
How do I temporarily disable a noisy alert?

# Pre-populate the dedup table with a recent timestamp
sqlite3 ~/reminders.db \
  "INSERT INTO sent_alerts (alert_key, sent_at_utc) VALUES ('noisy_key', datetime('now'));"

# This suppresses it for 60 min. For longer, repeat hourly or fix the root cause.
How do I upgrade ZeroClaw?
Back up current state manually (don't wait for debounce):

sqlite3 ~/.zeroclaw/memory.db ".backup /tmp/memory_pre_upgrade.db"
rclone copy /tmp/memory_pre_upgrade.db dropbox-crypt:/backups/manual/
Read ZeroClaw release notes for breaking changes
Stop service: sudo systemctl stop zeroclaw
Install new binary per official upgrade docs
Start: sudo systemctl start zeroclaw
Smoke test (see § 10 of INSTALL.md)
If broken, restore: cp /tmp/memory_pre_upgrade.db ~/.zeroclaw/memory.db, rollback binary
9. Escalation Decision Tree

Is the system reachable via SSH?
├── No  → Physical access. Check power, UPS, network cable.
│         If Pi truly dead → use RECOVERY.md
└── Yes
    │
    Are all 4 services running?
    ├── No  → systemctl status <svc>; journalctl -u <svc>
    │         Fix root cause; restart
    └── Yes
        │
        Can you chat in #chat?
        ├── No  → OpenRouter? Discord? Check § 3
        └── Yes
            │
            Are reminders firing?
            ├── No  → Check daemon poll; check NTP; check § 3
            └── Yes
                │
                Is Notion / Calendar / Backup working?
                ├── No  → Check § 2 alert table for that subsystem
                └── Yes → System is healthy. Investigate the specific
                           complaint or alert that brought you here.
10. When to Page Yourself vs. Wait
Resend already filters to "things that need attention." But within #alerts:

Severity	Response time
🔴 critical	Same-day investigation; system functionality reduced
🟡 high	Within 24 hours; risk of degradation or data loss
🟢 info	Weekly review; no immediate action needed
If you're on holiday: critical alerts will email via Resend (rate-limited).
You can SSH in from anywhere via your normal SSH key.

11. Glossary of Service States

sudo systemctl status reminder-daemon
Reported state	Meaning
active (running)	Healthy
active (exited)	One-shot finished successfully
activating	Starting up
inactive (dead)	Stopped intentionally
failed	Crashed; check logs
activating (auto-restart)	Crashed; systemd retrying
A service in failed state with a recent crash typically means: read the logs.

12. Contact Sheet
Within this document because you'll have it open during incidents.

What	Where
OpenRouter status	https://status.openrouter.ai(opens in new tab)
Discord status	https://discordstatus.com(opens in new tab)
Google Workspace status	https://www.google.com/appsstatus(opens in new tab)
Notion status	https://status.notion.so(opens in new tab)
Dropbox status	https://status.dropbox.com(opens in new tab)
Resend status	https://resend-status.com(opens in new tab)
Pi Foundation forum	https://forums.raspberrypi.com(opens in new tab)
ZeroClaw docs	[look up current URL]
