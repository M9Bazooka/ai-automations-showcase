# API & Tool Interface Documentation

This system doesn't expose a conventional REST API to end users — its "interface" is a phone call. But internally, the conversational agent communicates with the rest of the system through a well-defined **tool-calling interface**, and the system as a whole communicates with the outside world through a small number of **webhook contracts**. This document describes both, since together they're the closest thing this project has to a public API surface.

---

## 1. The Agent's Tool Interface

The LangChain agent at the core of the system has access to **seven tools**. Each is described to the model with a name, a natural-language description of when and how to use it, and a structured parameter schema — exactly the same shape as OpenAI/Anthropic function-calling definitions. The agent decides autonomously, per conversational turn, whether and which tools to invoke, in what order, and how to use their results in its spoken response.

### `get_current_datetime`

Returns the current date and time, plus "tomorrow's" date, formatted naturally for the restaurant's timezone (Europe/London) — e.g., resolving relative phrases like "today," "tomorrow," or "this Friday" into concrete calendar dates the other tools can work with.

| | |
|---|---|
| **Inputs** | none |
| **Returns** | structured object with today's date/time and tomorrow's date, in both machine-readable and natural-language forms |
| **Execution** | sandboxed JS (`toolCode`), pure computation, no network access |

---

### `query_menu`

The restaurant's full menu, exposed as a queryable tool rather than baked into the agent's static instructions. Covers 100+ items across starters, sharing platters, seafood boils, signature shrimp dishes, land & sea mains, burgers, pasta, sides, drinks, kids' meals and desserts — including prices, descriptions, spice indicators, dietary flags, and allergen information.

| | |
|---|---|
| **Inputs** | a natural-language query (e.g., "what shrimp dishes do you have," "anything vegetarian," "how spicy is the cajun boil," "what's in the kids' menu") |
| **Returns** | a structured, conversational answer: either a specific category's items, an answer to a cross-cutting question (allergens / price range), or — for queries that don't match a known pattern — the best-matching items found via fuzzy text search |
| **Execution** | sandboxed JS (`toolCode`), pure computation over an embedded structured dataset, no network access |

**Why this is a tool and not just "knowledge" in the prompt:** the agent is explicitly instructed never to describe menu items from memory. Every answer about food is sourced live from this tool — which means prices, descriptions, and availability are always exactly what the engine returns, never an invented or misremembered approximation. This is the difference between "an AI that knows about a menu" and "an AI that correctly reads a menu out loud."

---

### `check_availability`

Determines whether the restaurant can seat a party at a requested date and time.

| | |
|---|---|
| **Inputs** | requested date, requested time, party size |
| **Logic applied** | per-weekday opening hours, a "last seating" cutoff buffer before closing time (so the kitchen isn't still serving a multi-course meal at closing), a maximum-covers ceiling for that time slot, and natural-language date parsing (so "the first of March" and "01/03" both resolve correctly) |
| **Returns** | availability decision plus, where relevant, a human-readable explanation (e.g., why a requested time falls outside service hours) |
| **Execution** | delegated sub-workflow (`toolWorkflow`) — reads the current reservation ledger from Google Sheets to compute live capacity for the requested slot |

---

### `make_booking`

Creates a new reservation and confirms it to the guest.

| | |
|---|---|
| **Inputs** | guest name, phone number, date, time, party size, optional notes (dietary requirements / special requests) |
| **Process** | generates a unique booking reference, appends a confirmed reservation record to the ledger, formats the guest's number to E.164, and sends an SMS confirmation via the Twilio Messaging API containing the booking reference, party size, and a number to call to cancel |
| **Returns** | success confirmation with the booking reference, or a structured failure reason |
| **Execution** | delegated sub-workflow (`toolWorkflow`) — the only tool in the system that triggers an outbound communication (SMS) independent of the live call |

---

### `lookup_booking`

Finds an existing reservation.

| | |
|---|---|
| **Inputs** | any combination of: booking reference, phone number, guest name |
| **Logic applied** | exact match on booking reference or phone number where provided; fuzzy matching on guest name to handle minor spelling/pronunciation variance from speech-to-text |
| **Returns** | the matching reservation's current details (folded from the ledger to its latest state), or a clear "not found" result |
| **Execution** | delegated sub-workflow (`toolWorkflow`) |

This is also the tool invoked proactively at the start of a call when a caller's number matches a known booking — see [Conversation Design](README.md#conversation-design) for how that shapes the greeting itself.

---

### `modify_booking`

Changes the details of an existing reservation.

| | |
|---|---|
| **Inputs** | booking reference (to identify the reservation), plus any of: new date, new time, new party size, new notes |
| **Process** | appends a new ledger row under the *same* booking ID with the updated fields — the original entry is left untouched |
| **Returns** | confirmation of the updated details, or a structured failure reason (e.g., booking not found, new slot unavailable) |
| **Execution** | delegated sub-workflow (`toolWorkflow`) |

---

### `cancel_booking`

Cancels an existing reservation.

| | |
|---|---|
| **Inputs** | booking reference and/or phone number to identify the reservation |
| **Process** | appends a new ledger row for that booking ID with `status: cancelled` — nothing is deleted or overwritten |
| **Returns** | cancellation confirmation, or a structured failure reason if no matching active booking is found |
| **Execution** | delegated sub-workflow (`toolWorkflow`) |

---

## 2. Webhook Contracts

The system exposes two webhook endpoints. Both are internal integration points (Twilio → n8n), not public APIs — but they define the actual "wire format" the system operates on, which is worth documenting precisely.

### `POST /voice/handler` — Main Conversation Webhook

Receives every event for the lifetime of a call from Twilio Voice: the initial connection, every transcribed utterance, and every silence-timeout re-invocation.

**Representative inbound payload (Twilio's standard voice webhook fields):**
```
CallSid        — unique identifier for this call (used as the memory key)
From           — caller's phone number, in E.164 format
SpeechResult   — the transcribed text of what the caller just said (absent on first contact)
(additional Twilio-standard fields for call status, confidence scores, etc.)
```

**Response:** TwiML (XML) — either:
- a `<Gather>` wrapping a `<Play>` (speak a response, then re-open the microphone for the caller's next turn), or
- a `<Play>` followed by `<Hangup/>` (deliver a final message and end the call — used when the agent has detected the conversation has reached a natural close)

**Turn classification logic** (performed by the `Prepare Input` step before anything reaches the agent):

| Incoming signal | Classified as | Resulting instruction to the agent |
|---|---|---|
| First request for this `CallSid`, caller's number present | First contact | "A new call has connected from `<number>` — check whether this number matches an existing booking before greeting them, and shape your greeting accordingly" |
| `SpeechResult` present | Caller spoke | "The caller just said: `<transcribed text>` — respond appropriately" |
| No `SpeechResult`, not first contact | Caller went silent | "The caller hasn't said anything — check in naturally, don't assume the call has ended" |

### `POST /tts-proxy` — Text-to-Speech Proxy Webhook

Fetched by Twilio when it encounters a `<Play>` URL pointing at this system (rather than a static audio file).

**Inbound:** the text to be synthesized (passed through from the `Build TwiML` step's audio URL)

**Process:** forwards the request to ElevenLabs' streaming text-to-speech endpoint with tuned parameters (see [Technical Decisions §3](TECHNICAL_DECISIONS.md#3-voice-synthesis-elevenlabs-vs-twilios-built-in-tts) for the specifics and reasoning behind each one):
- Model: `eleven_flash_v2_5` (low-latency, conversational)
- Streaming MP3 output
- Latency optimization set to maximum
- Voice settings tuned for consistency over expressiveness (style processing disabled)

**Response:** a streamed `audio/mpeg` response, played directly by Twilio to the caller

---

## 3. Reference Implementations (Non-Production)

The repository also contains two complete, independent implementations of the same agent logic built directly against the OpenAI function-calling API, without n8n:

- **Node.js / Express** — a single-file server wiring together Twilio webhook handling, OpenAI's chat completions API with function calling, and an ElevenLabs TTS proxy route, with an in-memory reservation store
- **Python / FastAPI** — a structural equivalent of the above, demonstrating the same conversation loop, tool schema definitions, and TwiML assembly in an async Python idiom

These exist primarily as a **portability proof**: they show the conversation design and tool-calling contract translate directly onto a conventional backend stack, and they served as the original proving ground for the agent's behavior before the n8n implementation became the production system. Both define the *same* tool schemas described above (`create_reservation`, `lookup_reservation`, `update_reservation`, `cancel_reservation`, `transfer_to_human` in their naming convention), confirming the interface design is the stable, portable part of the system — independent of which orchestration platform runs it.
