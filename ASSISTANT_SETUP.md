# Ollama Vast.ai Assistant Plan

This guide outlines a practical architecture for running an Ollama instance on Vast.ai with a static IP, a remote custom frontend, Whisper (ASR), TTS, and integrations for Google Calendar + Gmail.

## 1) High-level architecture

- **Compute**: Vast.ai GPU instance running Docker.
- **Ollama**: Runs on the GPU box with a static IP and exposed API port.
- **Speech services**:
  - **Whisper** for speech-to-text (ASR).
  - **TTS** engine for speech synthesis (e.g., Piper, Coqui-TTS, or ElevenLabs if you want hosted).
- **Frontend**: Static web UI served from any CDN/static host; it connects to the Ollama API via your static IP (or via a reverse proxy with TLS).
- **Assistant Orchestrator**: A small service that:
  - Receives microphone audio from the frontend.
  - Calls Whisper to transcribe.
  - Calls Ollama for reasoning/response.
  - Calls TTS to synthesize audio.
  - Calls Google APIs for Calendar and Gmail via OAuth.

## 2) Container strategy

- **Single docker-compose** recommended for quick deployment, with services:
  - `ollama` (GPU)
  - `whisper` (GPU or CPU, depending on model size)
  - `tts` (CPU or GPU based on engine)
  - `orchestrator` (Node.js/Python)
  - `reverse-proxy` (Caddy or Nginx for TLS + routing)
- If you want to minimize complexity on Vast.ai, keep `ollama` + `orchestrator` on the GPU box and run the frontend elsewhere.

## 3) Networking and security

- Use TLS termination with a reverse proxy (Caddy/Nginx) and **only expose** required ports (e.g., 443).
- Lock down access with an auth layer (JWT or API keys) to avoid open access to your Ollama API.
- If hosting the frontend separately, configure CORS in the reverse proxy (or in the orchestrator) to allow only your frontend domain.

## 4) Whisper and TTS choices

- **Whisper**:
  - `openai/whisper` or `faster-whisper` for better performance.
  - For real-time, you can do streaming ASR in the orchestrator.
- **TTS**:
  - **Piper**: lightweight, local, good for offline.
  - **Coqui TTS**: better quality, heavier.
  - **ElevenLabs**: hosted, easiest quality, but not local.

## 5) Google Calendar + Gmail integration

- You will need a **Google Cloud project** with:
  - **Calendar API** enabled.
  - **Gmail API** enabled.
  - OAuth 2.0 Client ID.
- Your orchestrator should:
  - Perform OAuth login flow and store refresh tokens securely.
  - Use least-privilege scopes:
    - Calendar: `https://www.googleapis.com/auth/calendar`
    - Gmail: `https://www.googleapis.com/auth/gmail.modify` (or `gmail.readonly` if you don’t want to send mail).
- Consider using a secrets manager or encrypted storage for refresh tokens.

## 6) Recommended next steps (based on your preferences)

1. **Model target**: plan for **Ollama3** as the primary model; size GPU VRAM accordingly.
2. **Hosted TTS**: select a hosted provider (e.g., ElevenLabs or PlayHT) and configure API keys in the orchestrator.
3. **Domain + TLS**: terminate TLS at the reverse proxy and configure CORS to allow only your domain.
4. **Autonomous actions**: implement policy guards for Google Calendar/Gmail actions (rate limits, logging, and optional confirmation thresholds).
5. **Orchestrator**: implement the audio pipeline (audio in → Whisper → LLM → tools → TTS → audio out).

## 7) Assumptions confirmed

- **Model**: Ollama3.
- **Latency**: no strict latency target.
- **TTS**: hosted provider.
- **Domain**: available for TLS/redirect URIs.
- **Autonomy**: assistant may act autonomously.
