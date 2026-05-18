# AI Assistant

A single-user, Discord-native productivity assistant running on a Raspberry Pi 5.
Text-first conversation, proactive reminders, Google Calendar awareness, Notion
task mirror, optional audio replies, and encrypted cloud backup.



---

## 30-Second Pitch

You talk to it in a Discord channel (`#chat`). It remembers context, sets
reminders that fire at the right time (delivered to `#reminders`), checks your
Google Calendar before scheduling, mirrors everything to a Notion database for
search, and can read replies aloud as audio attachments if you ask. System
alerts go to `#alerts`. Backups are encrypted to Dropbox every 10 minutes.

No microphone. No speaker. No buttons. Discord is the only interface.

---

## Quick Start (after install)

```bash
# Health check
curl -H "X-Auth-Token: $FLASK_AUTH_TOKEN" http://127.0.0.1:7878/status | jq

# In Discord #chat, type:
"hi"                                         # smoke test
"remind me in 2 minutes to check the laundry" # reminder test
"what's on my calendar tomorrow?"            # calendar test
"say hello aloud"                            # TTS test
"/status"                                    # full system status
