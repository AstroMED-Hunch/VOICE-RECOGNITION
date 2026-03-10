# VOICE-RECOGNITION
Real-time LLM voice guidance layer for the AstroMED medical storage kiosk -> mirrors WebSocket protocol events and guides crew through shelf registration, face recognition, and checkout via speech.

# AstroMED VOICE

**NASA HUNCH Medical Storage Management — LLM Voice Guidance Layer**

A real-time voice guidance interface for the AstroMED medical supply storage kiosk. The system listens to the physical shelf storage and guides crew members through each protocol step using a locally-running LLM and text-to-speech output.

---

## Overview

AstroMED VOICE sits on top of the storage management kiosk as a voice layer. It does not control the hardware, but it mirrors every state transition broadcast by the backend over WebSocket and uses an LLM to generate concise spoken guidance for the crew member standing at the kiosk in real time.

The LLM is driven entirely by backend events. It does not respond to arbitrary conversation. Voice input from the crew is limited to yes/no confirmation during active protocol steps and an emergency keyword that bypasses all state.

---

## Features

- Real-time protocol guidance via text-to-speech as WebSocket events fire
- LLM responses grounded strictly in live backend data — no hallucinated values
- Continuous speech recognition with phonetic autocorrection for domain terms
- Three-path voice input: confirmation gate, emergency override, silent drop for everything else
- Off-protocol crew speech receives a console-only LLM response — no audio
- Primary LLM: Ollama local inference (default, no internet required after setup)
- Emergency LLM: WebLLM in-browser via WebGPU (toggled on demand, falls back to Ollama when disabled)
- Debug toggle to surface WebSocket traffic and system logs in the console
- Single-file deployment — no build step, no framework, open in any Chromium browser

---

## System Architecture

### Voice Interface (Frontend)

The frontend handles both **voice input and spoken output**, while coordinating with the backend through WebSocket events.

**Recognition Flow**

1. The system listens for WebSocket events from the backend.
2. Incoming events trigger **protocol handlers** in the interface.
3. A **live state snapshot** is sent to the LLM.
4. The LLM generates a response, which is:
   - displayed in the console
   - spoken through **text-to-speech**

### Voice Input

Voice commands are captured using the **Web Speech API**:

- Speech → converted to text (STT)
- Commands are parsed by the voice handler
- Sensitive actions pass through a **Yes/No confirmation gate**
- An **emergency override** allows immediate interruption or control

### Backend Connection

The interface communicates with the **Storage Management Backend** via WebSocket:

```
ws://localhost:3000
```

The backend provides real-time operational data and services including:

- Box detection
- Face recognition
- Shelf operations

### System Interaction Summary

- Backend emits events → frontend receives via WebSocket
- Frontend updates system state → sends context to LLM
- LLM generates responses → spoken via TTS
- User voice commands → processed through STT → validated → executed

---

## Protocol States

The voice layer handles every state the backend can broadcast:

| State | Trigger | Voice Guidance |
|------|------|------|
| `boxEntered` | Sensor detects a new box | Asks crew to confirm registration |
| `boxExited` | Box removed before registration | Acknowledges, confirms system ready |
| `face_recognition_boxentry` | Identity check for box entry | Instructs crew to face the camera |
| `face_recognition_boxexit` | Identity check for checkout | Instructs crew to face the camera |
| `faceRecognitionUpdate` | Face scan result received | Confirms identity or states error |
| `boxLocation` | Shelf assigned after registration | Directs crew to place box at shelf |
| `shelves_full` | No available shelves | Tells crew to wait |
| `multiple_boxes` | Multiple boxes in sensor frame | Tells crew to remove extra boxes |
| Checkout initiated | Crew triggers a shelf checkout | Prompts verbal yes/no confirmation |

---

## Requirements

### Primary LLM (Ollama — default)

- Ollama installed and running: https://ollama.com  
- Model pulled:

```
ollama pull gemma3:4b
```

- Server running:

```
ollama serve
```

- CORS enabled if running on a separate machine

### Emergency LLM (WebLLM — optional)

- Chrome 113+ or Edge 113+ (WebGPU required)
- GPU with at least 2GB VRAM recommended
- First activation downloads ~800MB (Llama 3.2 1B), cached permanently after that
- Works fully offline after initial cache

### Browser

- Chrome 113+ or Edge 113+ required for both WebGPU (Emergency LLM) and Web Speech API (STT)
- Microphone access must be permitted on first load

### Backend

- Storage management WebSocket server at `ws://localhost:3000`
- Shelf data API at `http://localhost:8000/getShelves`

---

## Setup

### 1. Install and start Ollama

```bash
# Install from https://ollama.com
ollama pull gemma3:4b
ollama serve
```

### 2. Start the storage management backend

Refer to the main AstroMED repository for backend setup.

### 3. Open the voice interface

Open `astro-med-voice.html` directly in Chrome or Edge. Grant microphone access when prompted — this happens once and is not requested again.

---

## Usage

**Live button** — toggles continuous speech recognition on and off. Disabled until the AI model is confirmed online.

**Emergency LLM toggle** — switches from Ollama to an in-browser WebLLM model. Use if the local Ollama server becomes unavailable. Toggling off restores Ollama.

**Debug toggle** — surfaces WebSocket message traffic, Ollama connection status, speech recognition events, and system logs in the console.

### Voice Commands

Under normal operation crew speech is transcribed but only two input paths produce a response:

- **Confirmation** — when a checkout is pending, say `yes` / `confirm` or `no` / `cancel`
- **Emergency** — say `emergency` at any time to trigger the emergency protocol override

All other speech is transcribed to the console. The LLM evaluates it and may respond in the console if the input is a relevant question about the system — but does not speak.

---

## File Structure

```
astromed-voice-recognition.html   — complete interface, single file, no dependencies to install
README.md               — this file
```

All vendor dependencies (Augmented UI, Google Fonts) are loaded from CDN. WebLLM is imported dynamically only when the Emergency LLM toggle is activated.

---

## WebSocket Message Reference

Messages the voice layer listens for from the backend:

```
boxEntered               { type, message: boxId }
boxExited                { type }
statusUpdate             { type, message: statusString }
boxLocation              { type, message: shelfLocation }
faceRecognitionUpdate    { type, message: crewName | "err_nodetect" | "err_multiple" }
boxUpdate                { type }  — triggers shelf data refresh from REST API
```

---

## Part of NASA HUNCH

This interface is a component of the AstroMED Medical Storage Management System, developed as part of the NASA High school students United with NASA to Create Hardware (HUNCH) program.
