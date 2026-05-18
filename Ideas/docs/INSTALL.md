# Installation Guide (Greenfield)

This is for **first-time setup with no existing backup**. If you're rebuilding
after a hardware failure, use [`RECOVERY.md`](./RECOVERY.md) instead.

**Estimated time:** 3–5 hours, mostly waiting for downloads.

---

## Prerequisites Checklist

Before you start, have ready:

- [ ] Raspberry Pi 5 (4 GB), NVMe SSD + PCIe HAT, Waveshare UPS HAT (E), heatsink + fan, PSU
- [ ] Ethernet cable (or WiFi credentials)
- [ ] A separate computer (for SSH and Discord setup)
- [ ] Accounts created:
  - [ ] [OpenRouter](https://openrouter.ai/) (£5 credit minimum)
  - [ ] [Discord Developer](https://discord.com/developers/applications)
  - [ ] Discord server (your private one-person guild) with three channels: `#chat`, `#reminders`, `#alerts`
  - [ ] [Resend](https://resend.com/) (free tier) + verified sender domain or use their test domain
  - [ ] [Google Cloud Console](https://console.cloud.google.com/) project with Calendar API enabled
  - [ ] [Notion](https://notion.so) workspace + integration token
  - [ ] [Dropbox](https://dropbox.com) account
- [ ] [`RECOVERY.md`](./RECOVERY.md) template printed and ready to fill in

---

## Phase 0 — Account & Service Setup (1 hour)

Do all of these from your main computer before touching the Pi.

### 0.1 Discord

1. Create a private guild (server): right-click server list → "Create a Server" → "For me and my friends"
2. Create three text channels: `#chat`, `#reminders`, `#alerts`
3. Enable **Developer Mode**: User Settings → Advanced → Developer Mode ON
4. Right-click each channel → Copy Channel ID. Save these.
5. Go to https://discord.com/developers/applications → New Application
6. Bot tab → Reset Token → copy it (you only see it once)
7. Bot tab → enable "Message Content Intent" and "Server Members Intent"
8. OAuth2 → URL Generator → scopes: `bot`, `applications.commands`; permissions: `Send Messages`, `Read Message History`, `Attach Files`, `Use Slash Commands`
9. Copy the generated URL, paste in browser, authorise into your guild

### 0.2 OpenRouter

1. Sign up at https://openrouter.ai/
2. Add £5 credit (covers months of single-user use)
3. Settings → Keys → Create Key. Copy it (`sk-or-v1-...`)
4. Verify model access at https://openrouter.ai/openai/gpt-4o-mini
5. **Verify current TTS slug** at https://openrouter.ai/models?q=tts — confirm exact model ID for `gpt-4o-mini-tts`

### 0.3 Resend

1. Sign up at https://resend.com/
2. Add and verify a domain (or use their `onboarding@resend.dev` for testing)
3. API Keys → Create. Copy it (`re_...`)

### 0.4 Google Calendar

1. https://console.cloud.google.com/ → New Project (name: `ai-assistant`)
2. APIs & Services → Library → enable "Google Calendar API"
3. APIs & Services → Credentials → Create Credentials → OAuth client ID
4. Application type: **Desktop app**
5. Download the JSON; note `client_id` and `client_secret`
6. (Refresh token will be generated in Phase 5)

### 0.5 Notion

1. https://www.notion.so/my-integrations → New integration (Internal)
2. Name: "AI Assistant". Copy the secret (`secret_...`)
3. In Notion, create a new database with these properties:
   - `Task` (Title)
   - `Due` (Date)
   - `Status` (Select: `pending`, `fired`, `cancelled`)
   - `Priority` (Select: `normal`, `high`)
   - `Recurring` (Checkbox)
   - `SQLite ID` (Number)
4. Open the database as a full page. Click `⋯` → Connections → Add → select your integration
5. Copy the database ID from the URL (32-char string before `?v=`)

### 0.6 Dropbox

1. Sign up at https://dropbox.com/
2. No further setup needed in browser — `rclone config` handles OAuth on the Pi

### ✏️ Fill In `RECOVERY.md` Now

Take the printed copy and write in (or paste into password manager) all the values you just collected. **Don't skip this** — half of them you'll never see again easily.

---

## Phase 1 — Hardware & OS (45 min)

### 1.1 Assemble Hardware

1. Attach the PCIe HAT and NVMe SSD to the Pi 5
2. Attach the Waveshare UPS HAT (E) per its manual
3. Attach heatsink + fan
4. Connect Ethernet (or have WiFi creds ready)
5. Plug in PSU **last**

### 1.2 Flash OS

On your main computer:

1. Download Raspberry Pi Imager from https://www.raspberrypi.com/software/
2. Insert NVMe via USB enclosure (or use bootable SD card first to copy OS to NVMe)
3. Choose OS: **Raspberry Pi OS (64-bit)** — the regular Bookworm version, not Lite (unless you prefer headless from the start; Lite is fine)
4. Click the gear icon for OS customisation:
   - Hostname: `assistant` (or your choice)
   - Username: `pi`, password: (strong, save it to `RECOVERY.md`)
   - Configure WiFi if using
   - Enable SSH with public-key auth (paste your `~/.ssh/id_ed25519.pub`)
   - Set locale: `Europe/London`, keyboard: `gb`
5. Write to NVMe
6. Install NVMe in PCIe HAT, boot the Pi

### 1.3 First Boot & SSH In

```bash
# From your main computer
ssh pi@assistant.local

# If .local mDNS doesn't work, find the IP from your router
1.4 System Update & Base Packages

sudo apt update && sudo apt full-upgrade -y
sudo apt install -y \
    git python3-venv python3-pip python3-dev \
    rclone inotify-tools \
    sqlite3 chrony \
    jq bc curl \
    build-essential

sudo apt autoremove -y
1.5 Verify Health

vcgencmd measure_temp                          # < 60°C at idle
chronyc tracking | grep "System time"          # offset < 1 second
df -h /                                        # plenty of space
free -h                                        # ~3.5 GB free
1.6 Install Waveshare UPS HAT (E) Driver
Follow current vendor documentation at https://www.waveshare.com/wiki/UPS_HAT_(E)(opens in new tab)

After install, verify:


# Should print battery voltage and status
python3 /home/pi/UPS_HAT_E/INA219.py
Add the shutdown daemon to systemd per their docs.

Phase 2 — Clone Repo & Python Setup (15 min)

cd /home/pi
git clone <your-repo-url> assistant
cd assistant

# Create virtualenv
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install --upgrade pip
pip install --require-hashes -r requirements.txt

# Sanity check
python -c "import flask, dateutil, googleapiclient, notion_client, resend; print('OK')"
Phase 3 — Secrets Configuration (15 min)

# Create secrets file
sudo mkdir -p /etc/zeroclaw
sudo cp /home/pi/assistant/.env.example /etc/zeroclaw/secrets.env
sudo nano /etc/zeroclaw/secrets.env
Fill in all the values from § 0. For FLASK_AUTH_TOKEN, generate:


openssl rand -hex 32
Leave GOOGLE_REFRESH_TOKEN blank for now (Phase 5).

Lock down permissions:


sudo chown root:pi /etc/zeroclaw/secrets.env
sudo chmod 0640 /etc/zeroclaw/secrets.env   # 0640 because pi user reads it via systemd; not 0600
Phase 4 — ZeroClaw Install & First Boot (30 min)
Install ZeroClaw per its official documentation
Run onboarding:

zeroclaw onboard
Provider: OpenRouter (paste API key from secrets)
Channel: Discord (paste bot token)
Restrict to #chat channel only
Verify ZeroClaw starts:

zeroclaw service start
Test from Discord #chat: send "hello" — expect a reply within 5 seconds
Stop it: zeroclaw service stop (we'll run it via systemd next)
Phase 5 — Google Calendar OAuth (10 min)

cd /home/pi/assistant
source venv/bin/activate
python scripts/setup_google_oauth.py
This will:

Print a URL
You paste the URL into a browser on your main computer
Authorise → copy back the verification code → paste into terminal
The script prints GOOGLE_REFRESH_TOKEN=...
Edit /etc/zeroclaw/secrets.env and paste it in. Save and exit.

Test:


python -c "
import os, sys
sys.path.insert(0, '.')
from integrations_api import google_calendar_check
from datetime import datetime, timezone, timedelta
now = datetime.now(timezone.utc)
print(google_calendar_check(now, now + timedelta(days=1)))
"
Phase 6 — Install systemd Services (15 min)

sudo cp /home/pi/assistant/systemd/*.service /etc/systemd/system/
sudo systemctl daemon-reload

# Start in order
sudo systemctl enable --now reminder-api.service
sleep 5
sudo systemctl status reminder-api.service     # should be active (running)

sudo systemctl enable --now reminder-daemon.service
sleep 5
sudo systemctl status reminder-daemon.service

sudo systemctl enable --now zeroclaw.service
sleep 5
sudo systemctl status zeroclaw.service
Watch logs:


sudo journalctl -f -u zeroclaw -u reminder-api -u reminder-daemon
Phase 7 — Backup Setup (20 min)
7.1 Configure rclone

rclone config
n → New remote
Name: dropbox-raw
Type: dropbox
Follow OAuth prompts (it will print a URL; paste in browser; copy code back)
Then create the crypt wrapper:

n → New remote
Name: dropbox-crypt
Type: crypt
Remote: dropbox-raw:/pi-assistant-encrypted/
Filename encryption: standard
Directory name encryption: true
Password: generate a strong one — openssl rand -base64 32
Salt: generate another one — openssl rand -base64 32
⚠️ WRITE BOTH PASSWORDS INTO RECOVERY.md IMMEDIATELY. If you lose them, every backup becomes garbage forever.

Test:


rclone ls dropbox-crypt:
echo "test" | rclone rcat dropbox-crypt:/test.txt
rclone cat dropbox-crypt:/test.txt          # should print "test"
rclone delete dropbox-crypt:/test.txt
7.2 Enable Backup Service

sudo systemctl enable --now rclone-backup.service
sudo systemctl status rclone-backup.service
Wait 11 minutes, then check Dropbox app/web — you should see encrypted files appearing in pi-assistant-encrypted/.

7.3 Schedule Verification Cron

sudo cp /home/pi/assistant/scripts/cron.d/backup-verify /etc/cron.d/
sudo cp /home/pi/assistant/scripts/cron.d/quarterly-maintenance /etc/cron.d/
sudo cp /home/pi/assistant/scripts/cron.d/watchdog /etc/cron.d/
Phase 8 — TTS Audio Cache (10 min)

cd /home/pi/assistant
source venv/bin/activate
python scripts/generate_audio_cache.py
Generates ~30 OGG files in audio_cache/ using voice=alloy.

Verify count:


ls audio_cache/*.ogg | wc -l   # should be ~30
Phase 9 — Notion Backfill (5 min)

python scripts/notion_backfill.py
(Will be a no-op on first install since reminders.db is empty — but it
verifies Notion connectivity.)

Phase 10 — Smoke Test (15 min)
Run through all 7 tests from RECOVERY.md Phase F:

✅ Send "hello" in #chat → reply within 5s
✅ "remind me in 2 minutes to test" → fires in #reminders within 2–3 min
✅ "what's on my calendar tomorrow?" → returns a list
✅ Check Notion DB → new reminder row appears
✅ "say hello aloud" → OGG attachment in #chat
✅ Type /status → all green
✅ Wait 11 min, verify new files in Dropbox
If all pass: you're live. 🎉

If any fail: see docs/OPERATIONS.md.

Phase 11 — Lock Down RECOVERY.md (5 min)
Update RECOVERY.md with EVERY value you used
Print fresh paper copy
Update password-manager secure note
Set Google Calendar reminder: "Annual RECOVERY test-restore" (1 year out)
Set Google Calendar reminder: "Monthly: rclone token reconnect check" (monthly)
Set Google Calendar reminder: "Quarterly: rotate API keys" (quarterly)
Phase 12 — Update CHANGELOG.md
Record the initial deployment:


## [0.1.0] — YYYY-MM-DD

Initial deployment.
- Hardware: Pi 5 4GB, NVMe, Waveshare UPS HAT (E)
- OS: Raspberry Pi OS 64-bit Bookworm
- Python: 3.11.x
- ZeroClaw: x.y.z
- All services healthy, smoke tests pass.
Common First-Time Gotchas
Problem	Fix
zeroclaw onboard doesn't see the API key	source /etc/zeroclaw/secrets.env before running
Discord bot offline after invite	Check Message Content Intent enabled in Dev Portal
Google OAuth "Access blocked: not verified"	OAuth consent screen → Publishing → "In production" OR add yourself as test user
Notion 401 from backfill script	Database not shared with integration; redo step 0.5 #4
rclone Dropbox auth fails on headless Pi	Run rclone authorize "dropbox" on your main computer with rclone installed, paste result on Pi
systemd service immediately fails	journalctl -u <svc> -n 50 — usually a missing env var
Flask returns 401 to everything	FLASK_AUTH_TOKEN in secrets doesn't match what ZeroClaw is sending
Reminder fires hours late	Check chronyc tracking — NTP offset > 10s is the culprit
Next Steps After Install
Read docs/OPERATIONS.md so you know what to do when alerts fire
Test the recovery procedure once before you trust it (use a spare SD card)
Set up monthly cost tracking in docs/BUDGET.md
Consider hardening: SSH key-only, fail2ban, automatic security updates

