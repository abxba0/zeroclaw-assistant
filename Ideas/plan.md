# AI Assistant Project Plan

## Overview
This project aims to develop a voice-enabled AI assistant that interacts with the user via Discord, handles notifications through email, stores data on Dropbox, performs AI inference using OpenRouter, and can respond via TTS through speakers. The system will be deployed on a Raspberry Pi. Zeroclaw will be used for assistant orchestration.

---

## 1. Core Components

### 1.1 Voice Input
- Capture user voice through microphone.
- Transcribe voice using **Whisper Large v3** via OpenRouter:  
  - URL: [https://openrouter.ai/openai/whisper-large-v3](https://openrouter.ai/openai/whisper-large-v3)

### 1.2 AI Inference
- Process transcribed text using **OpenRouter AI models**.
- Model: `gpt-4o-mini` for core inference tasks.
- **Multi-Model Setup**:
  - Configure multiple AI models for different tasks:
    - Primary inference: `gpt-4o-mini`
    - TTS: `gpt-4o-mini-tts`
    - Optional fallback or specialized models for high load or specific tasks.
  - Dynamically switch models based on task, priority, or resource usage.

### 1.3 Text-to-Speech (TTS)
- Convert AI responses to audio using **gpt-4o-mini-tts** via OpenRouter:  
  - URL: [https://openrouter.ai/openai/gpt-4o-mini-tts-2025-12-15](https://openrouter.ai/openai/gpt-4o-mini-tts-2025-12-15)
- Output audio through connected speakers.

### 1.4 Discord Integration
- AI assistant communicates with the user through Discord.
- Features:
  - Respond to text messages.
  - Initiate voice messages (TTS).
  - Receive voice clips for transcription.
  - **Commands via Zeroclaw**:
    - `/new` – Clear conversation history to free token usage.
    - Additional commands can be added for system controls or advanced actions.

### 1.5 Email Notifications
- Use **Resend API** to send notifications to user via email:  
  - URL: [https://resend.com/](https://resend.com/)
- Use cases:
  - Notify about AI actions.
  - Alerts for system events or reminders.

### 1.6 Storage
- Store voice clips, transcripts, and logs on **Dropbox** via Dropbox API.
- Ensure persistent storage for long-term usage and historical data.

---

## 2. Hardware
- **Raspberry Pi**: Main device for hosting the assistant.
- **Speakers**: For audio output using TTS.
- Optional: Microphone for voice input (if not built-in).

---

## 3. Software / APIs
| Component            | Service / API                                 | Purpose                                  |
|----------------------|-----------------------------------------------|------------------------------------------|
| Voice Transcription   | OpenRouter Whisper Large v3                   | Transcribe voice to text                 |
| AI Inference          | OpenRouter GPT-4o-mini                        | Process text input and generate response|
| Text-to-Speech (TTS) | OpenRouter GPT-4o-mini-tts                     | Convert text to audio                    |
| Notifications         | Resend.com                                   | Send email notifications                 |
| Storage               | Dropbox API                                   | Store transcripts, audio files, logs    |
| Voice / Messaging     | Discord API                                   | Communicate with the user                |
| Orchestration         | Zeroclaw                                      | Manage sessions, commands, and context  |

---

## 4. Workflow

1. User speaks into microphone or sends a voice clip on Discord.
2. Voice clip is transcribed to text using Whisper Large v3.
3. Transcribed text is sent to AI (GPT-4o-mini) for inference.
4. AI generates a response.
5. Response is:
   - Sent as a text message on Discord.
   - Converted to speech via GPT-4o-mini-tts and played through speakers.
6. Relevant actions or alerts trigger email notifications via Resend.
7. All voice clips, transcripts, and logs are stored on Dropbox for future reference.
8. **Command Handling**:
   - `/new` – Clears conversation history in Zeroclaw to reset context and free tokens.
   - Future commands can include model switching, system status checks, or admin tasks.

---

## 5. Future Plans
- Expand AI capabilities with additional OpenRouter models.
- Implement conversation history and context retention across sessions.
- Enhance Discord integration with interactive commands and reactions.
- Integrate with other home automation tools or IoT devices.
- Improve TTS naturalness and support multiple languages.
- Optimize multi-model selection dynamically based on load and task type.
- Consider local AI inference optimizations for Raspberry Pi performance.

---

## 6. Notes
- The system is modular: voice input, AI inference, and output are separate services.
- Emphasis on Discord as primary interface, but modular for future expansion.
- Cloud APIs (OpenRouter, Resend, Dropbox) handle heavy processing and storage.
- Zeroclaw provides orchestration for commands, multi-model routing, and session management.
