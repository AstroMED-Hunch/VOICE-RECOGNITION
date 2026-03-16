# AIMS VOICE

![AIMS VOICE interface](https://github.com/user-attachments/assets/6244e522-e1f7-4b97-b988-de490785459c)

Voice guidance layer for the AIMS medical storage kiosk. Part of the NASA HUNCH program.

---

The kiosk runs a WebSocket-driven state machine that handles box detection, face recognition, shelf assignment, pill verification, and checkout. This module sits on top of that and speaks to whoever is standing at the kiosk, telling them what to do at each step, reading back scan results, and waiting for verbal confirmation where needed.

It's a single HTML file. No build step, no framework.

---

## How the guidance works

Every time the backend emits a protocol event, the voice layer captures the current system state, strips any null values, and sends it to a local LLM (Ollama running `gemma3:4b`). The model gets the state data and a task, something like "tell the crew member which shelf to place the box at", and produces one or two sentences. Those get spoken aloud and printed to the console.

The model is hard-constrained to only reference values that exist in the state snapshot. It can't guess a shelf name or invent a crew member's identity.

Voice input from the crew is narrow on purpose. The mic stays open, but the system only acts on two things: a yes/no during an active confirmation prompt, and the word "emergency" which overrides everything. Anything else gets transcribed and quietly evaluated. If it's a relevant question the LLM writes a response to the console, but it doesn't speak.

---

## Protocol states

| State | What triggered it | What gets said |
|---|---|---|
| `boxEntered` | Sensor picked up a new box | Asks the crew if they want to register it |
| `boxExited` | Box removed mid-flow | Acknowledges and resets |
| `boxEnterCancel` | Backend canceled the entry | Tells crew the registration was called off |
| `face_recognition_boxentry` | ID check for box entry | Tells crew to look at the camera |
| `face_recognition_boxexit` | ID check for checkout | Tells crew to look at the camera |
| `faceRecognitionUpdate` | Scan result returned | Reads the name back, or describes what went wrong |
| `boxLocation` | Shelf assigned | Tells crew exactly where to put the box |
| `shelves_full` | No space left | Tells crew to wait |
| `multiple_boxes` | Too many boxes in frame | Tells crew to pull extras out |
| `pill_checkup` | Pill scan required | Tells crew to aim the camera at the pills and hit capture |
| `pillScanResult` | Scan completed | Reads back each pill type and quantity |
| Checkout | Crew initiates a shelf checkout | Asks yes or no, holds until crew responds |

---

## Setup

You need Ollama installed and `gemma3:4b` pulled before opening the interface.

```bash
ollama pull gemma3:4b
ollama serve
```

Then start the AIMS backend (see the main repo). If Ollama is running on a separate machine, CORS needs to be enabled on that machine.

---

## The interface

The **Live** button starts and stops the mic. It stays grayed out until Ollama connects.

The **Emergency LLM** toggle is a fallback. If Ollama goes offline mid-session, flipping this on switches to a backup model so guidance can continue. Toggle it back off to return to Ollama when it's back up.

The **Debug** toggle shows raw WebSocket traffic and system logs in the console. Off by default, useful when something isn't behaving.

---

## Backend

Needs the WebSocket server at `ws://localhost:3000` and the shelf API at `http://localhost:8000/getShelves`.

Full message reference:

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

Built for the NASA High school students United with NASA to Create Hardware (HUNCH) program.
