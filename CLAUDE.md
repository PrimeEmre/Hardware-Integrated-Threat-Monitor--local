# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Does

A hardware-integrated cybersecurity threat intelligence dashboard. A Flask backend scrapes live news via DuckDuckGo, feeds it to a local LLM (Ollama) or optionally a CrewAI pipeline, generates a Markdown threat briefing, and signals an Arduino via serial to flip a red/green LED based on threat severity. The web UI renders the briefing and mirrors the hardware state visually.

## Running the App

```bash
pip install flask requests python-dotenv pyserial duckduckgo-search
python app.py
```

`duckduckgo-search` installs as the `ddgs` package (imported via `from ddgs import DDGS`).

App runs at `http://localhost:5000` (port overridable via `PORT` env var). Set `FLASK_DEBUG=1` for debug mode.

## Environment Variables (`.env`)

| Variable | Purpose |
|---|---|
| `ARDUINO_PORT` | Serial port for Arduino (default `COM3`) |
| `OLLAMA_URL` | Ollama chat endpoint (default `http://localhost:11434/api/chat`) |
| `OVERRIDE_MODEL` | Force a specific Ollama model, skipping GPU tier auto-detection |
| `CREWAI_CREW_URL` | Optional CrewAI crew endpoint; if set, tried before Ollama |
| `CREWAI_CREW_TOKEN` | Bearer token for CrewAI |
| `CACHE_TTL` | Cache TTL in seconds (default `3600`) |
| `SECRET_KEY` | Flask secret key |

## Architecture

**Request flow for `/generate-briefing`:**
1. Topic is sanitized and hashed into a cache key (per topic + date).
2. If cached, return immediately and re-trigger hardware signal.
3. If `CREWAI_CREW_URL` is set, kick off a CrewAI crew and poll for result.
4. Fallback: DuckDuckGo search → Ollama local LLM with an adaptive system prompt.
5. `send_hardware_signal()` parses `[HARDWARE_STATUS: CRITICAL/NOMINAL]` tag from LLM output, writes `C` or `O` byte to Arduino serial, and strips the tag before returning to the client.

**GPU tier auto-selection** (`app.py:132–169`): `nvidia-smi` detects VRAM at startup and picks an Ollama model + context window from `GPU_TIERS`. Override with `OVERRIDE_MODEL`.

**Hardware signal protocol**: Arduino (`threat_monitor.ino`) listens at 9600 baud. Byte `C` → red LED (pin 9), byte `O` → green LED (pin 10).

**Frontend** ([static/script.js](static/script.js)): Vanilla JS. Calls `/generate-briefing` (POST) and `/api/test-hardware` (GET). Renders Markdown with `marked.js`. UI LED strip and LCD widget mirror hardware state.

**Rate limiter**: In-memory, IP-keyed, 10 req/60 s. Applied via `@app.before_request` to all `/api/*` and `/generate-briefing` routes. Resets on server restart — not shared across workers.

**Threading**: Flask runs with `threaded=True`. The `LRUCache` and `RateLimiter` classes use `threading.Lock` for safety. The Arduino `serial.Serial` object is a module-level singleton; concurrent writes are not locked, so only one request should write to serial at a time in practice.

## Key API Endpoints

| Route | Method | Purpose |
|---|---|---|
| `/generate-briefing` | POST | Main briefing generation |
| `/api/health` | GET | Health + detected model |
| `/api/gpu-info` | GET | VRAM + selected tier |
| `/api/test-hardware` | GET | Send `C` directly (bypass AI) |
| `/api/clear-cache` | POST | Flush in-memory LRU cache |

## Hardware Testing

`test.py` is a standalone script to verify Arduino serial connectivity on a specific port (hardcoded to `COM9`). Edit the port before running:

```bash
python test.py
```

## Arduino

Flash `threat_monitor.ino` via Arduino IDE. No libraries needed — uses built-in `Serial`. Red LED on pin 9, green LED on pin 10.
