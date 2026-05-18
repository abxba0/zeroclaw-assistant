# AI Assistant Project Plan

**Status:** Locked spec — ready to build
**Target:** Single-user Discord-native productivity assistant on Raspberry Pi 5
**Last updated:** 2026-05-18

---

## 1. Overview

A text-first personal productivity assistant that lives entirely on Discord. It runs on a Raspberry Pi 5 in a fixed room install, orchestrated by ZeroClaw, with a separate scheduler daemon for proactive reminders, Google Calendar for context-awareness, and Notion as a searchable task mirror. TTS is opt-in and delivered as Discord audio attachments. Backups are encrypted to Dropbox.

### Design Principles
- **Pragmatic over pretty** — every component must have a defined failure mode.
- **Defence in depth** — localhost-bound services still require auth.
- **Recoverability** — assume the NVMe will fail; document and test restore.
- **Observability** — every service emits structured logs and a heartbeat.
- **No voice hardware** — Discord text is the only user interface.

---

## 2. Architecture
