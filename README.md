# AstroMED VOICE

![AstroMED VOICE interface](https://github.com/user-attachments/assets/6244e522-e1f7-4b97-b988-de490785459c)

**NASA HUNCH Medical Storage Management — LLM Voice Guidance Layer**

A real-time voice guidance interface for the AstroMED medical supply storage kiosk. The system listens to the backend's WebSocket state machine and guides crew members through each protocol step using a locally-running LLM and text-to-speech output.

---

## What it does

AstroMED VOICE is a voice layer that sits on top of the storage management kiosk. It doesn't control any hardware directly — it listens to every state change the backend broadcasts and uses an LLM to generate spoken guidance for whoever is standing at the kiosk.

The LLM only fires on backend protocol events. It doesn't respond to open-ended conversation. The only voice input the system actually acts on is yes/no confirmation during an active protocol step, and an emergency keyword that overrides everything.

---

## How it works

```
Backend emits event → WebSocket → protocol handler fires
    → live state snapshot sent to LLM
    → LLM response spoken via TTS + printed to console

Crew speaks → STT → yes/no gate or emergency override
```

All LLM responses are grounded strictly in the live state data — box IDs, shelf locations, face recognition results, pill scan data — whatever the backend actually sent. The model is instructed never to reference a value that isn't in the data block, so it can't hallucinate shelf names or crew identities.

---

## Protocol states

Every state the backend can broadcast is handled:

| State | What happened | What the system says |
|---|---|---|
| `boxEntered` | Sensor picked up a new box | Asks crew if they want to register it |
| `boxExited` | Box removed before finishing | Acknowledges, resets to idle |
| `boxEnterCancel` | Backend canceled the entry | Tells crew the registration was canceled |
| `face_recognition_boxentry` | ID check started for box entry | Tells crew to look at the camera |
| `face_recognition_boxexit` | ID check started for checkout | Tells crew to look at the camera |
| `faceRecognitionUpdate` | Face scan result came back | Confirms the name or describes the error |
| `boxLocation` | Shelf assigned after registration | Tells crew which shelf to put the box at |
| `shelves_full` | No open shelves | Tells crew to wait |
| `multiple_boxes` | More than one box in frame | Tells crew to remove the extras |
| `pill_checkup` | Pill verification required | Tells crew to point the camera at the pills and capture |
| `pillScanResult` | Pill scan completed | Reads back detected pill types and quantities |
| Checkout initiated | Crew triggers a checkout | Asks yes or no to confirm, waits for voice response |

---

## Requirements

### Primary LLM — Ollama (default)

This is the normal path. Runs locally, no internet needed once set up.

- Install Ollama: https://ollama.com
- Pull the model: `ollama pull gemma3:4b`
- Start the server: `ollama serve`
- If Ollama is running on a different machine, make sure CORS is enabled

### Emergency LLM — WebLLM (optional)

If Ollama goes down during a session, the Emergency LLM toggle loads a backup model that runs entirely in the browser via WebGPU. First load pulls about 800MB and caches it permanently — after that it works offline. Needs Chrome 113+ or Edge 113+ and a GPU with at least 2GB VRAM.

### Browser

Chrome 113+ or Edge 113+. Both WebGPU (for the emergency model) and the Web Speech API (for STT) require a Chromium-based browser. Microphone access is requested once on page load.

### Backend

- WebSocket server at `ws://localhost:3000`
- Shelf data API at `http://localhost:8000/getShelves`

---

## Setup

**1. Get Ollama running**

```bash
ollama pull gemma3:4b
ollama serve
```

**2. Start the AstroMED backend**

See the main AstroMED repository for backend setup. The voice interface needs the backend up before it can do anything useful.

**3. Open the interface**

Open `astromed-voice-recognition.html` in Chrome or Edge. Allow microphone access when the browser asks — it only asks once.

---

## Controls

**Live** — starts and stops continuous speech recognition. Stays grayed out until Ollama is confirmed online.

**Emergency LLM** — flips from Ollama to the in-browser WebLLM backup. Toggle it off to switch back. The first time you turn it on it'll download the model, which takes a moment.

**Debug** — shows raw WebSocket traffic, Ollama connection logs, and speech recognition events in the console. Off by default.

---

## Voice input

Most crew speech just gets transcribed to the console. The LLM reads it and may write a response there too, but it won't speak unless it's a protocol event that calls for it.

The only two voice paths that actually do something:

- **Confirmation** — during a checkout, say `yes`, `confirm`, `no`, or `cancel`
- **Emergency** — say `emergency` at any point to trigger the override regardless of state

---

## WebSocket message reference

```
boxEntered               { type, message: boxId }
boxExited                { type }
boxEnterCancel           { type }
statusUpdate             { type, message: statusString }
boxLocation              { type, message: shelfLocation }
faceRecognitionUpdate    { type, message: crewName | "err_nodetect" | "err_multiple" }
pillScanResult           { type, pills: [{ pill_type, quantity }] }
boxUpdate                { type }
```

---

## File structure

```
astromed-voice-recognition.html   — the whole interface, one file, no build step
README.md
```

Augmented UI and Google Fonts load from CDN. WebLLM is only imported dynamically when the Emergency LLM toggle is switched on.

---

## Part of NASA HUNCH

AstroMED VOICE is part of the AstroMED Medical Storage Management System, built for the NASA High school students United with NASA to Create Hardware (HUNCH) program.
