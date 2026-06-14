# Technical Decisions

This document goes deeper than the README's summary table — it walks through the *reasoning* behind the choices that shaped this system, including the alternatives considered and the trade-offs accepted. The goal is to show how decisions were actually made, not just what was picked.

---

## 1. Orchestration platform: n8n vs. a hand-written backend

**The choice:** Run the production system on self-hosted n8n rather than a custom Express/FastAPI server.

**The context:** The repository actually contains two complete, working reference implementations of this exact agent — one in Node.js/Express, one in Python/FastAPI — both using OpenAI's function-calling API directly against a webhook server. They were built first, as a way to prove out the conversation design and tool-calling pattern in a familiar, fully-controlled environment.

**Why n8n won for production:**
- **Iteration speed on conversation design.** The single biggest cost in building a voice agent isn't writing the initial code — it's the dozens of rounds of "the agent misunderstood that, let me adjust the system prompt / tool description / business rule and try again." In a hand-rolled server, each of those is a code change, a restart, and a test call. In n8n, it's an inline edit and an instant re-run with a full execution trace showing exactly what the agent saw, thought, and did at each step.
- **Debuggability in production.** When something goes wrong on a live call, n8n's execution history shows the *exact* data that flowed through every node for that specific call — the webhook payload, the agent's reasoning, every tool call and its result, the final TwiML. Reproducing that level of visibility in a custom server means building a bespoke logging and tracing system.
- **Lower operational surface area.** No server process to keep alive, no deploy pipeline, no process manager — n8n is already a mature platform handling all of that.

**The trade-off, honestly stated:** Visual workflow tools are worse than code for genuinely complex logic — branching, loops, and anything stateful gets unwieldy fast. The mitigation is visible in the design itself: the moment logic gets non-trivial (the menu engine, opening-hours math, ledger folding, fuzzy matching), it's pushed into a `Code` node and written as plain JavaScript, rather than fighting the visual canvas. n8n is used for what it's good at — orchestration, integration, and rapid iteration — and code is used for what *it's* good at. The two reference servers remain in the repo, proving the logic is portable to a conventional backend if the project ever needs to scale beyond what a visual tool comfortably supports.

---

## 2. Data persistence: Google Sheets (event-sourced) vs. PostgreSQL

**The choice:** Store reservation data in a Google Sheet, accessed as an append-only event log — despite a complete, production-grade PostgreSQL schema existing in the repository (`database/schema.sql`, including a `reservations` table with status enums, check constraints, triggers, and a `call_logs` table for analytics).

**The context:** A relational schema was designed first — it's the "obviously correct" engineering choice for structured, relational data with state transitions (confirmed → modified → cancelled → completed → no-show).

**Why Sheets won for production anyway:**
- **The client needs direct access.** A restaurant's front-of-house staff need to glance at tonight's bookings between seating guests — not file a support ticket to the engineer who built the system. A Google Sheet they already know how to use, that updates in real time as the agent books tables, is worth more to them than a "proper" database they'd never be able to open.
- **Zero hosting burden.** No database server to provision, secure, back up, or pay for.

**The hard problem this creates — and how it's solved:** Spreadsheets have no transactions and no row-level locking. A naive "find the row for this booking ID and update it" pattern is a race condition waiting to happen the moment two operations touch the same record close together (e.g., an agent re-confirming a booking while a modification is mid-flight). 

The fix is to stop trying to make the spreadsheet behave like a database, and instead treat it as what it actually is well-suited to be: **an append-only log**.
- Every operation — create, modify, cancel — *appends* a new row. Nothing is ever found-and-updated.
- "Current state" for a given booking ID is derived, not stored: read every row with that ID, and the most recent one wins.
- This sidesteps concurrent-write hazards entirely (there's no shared mutable state to race over), and produces a complete audit trail as a *side effect* of the storage model rather than as a deliberately bolted-on feature.

This is the same pattern that underlies event sourcing and CQRS in much larger systems — applied here because it happens to be exactly the right fit for "a spreadsheet that needs to behave safely under concurrent access," not because it was fashionable.

**What was kept from the relational design:** The PostgreSQL schema remains in the repo as the natural migration target if/when call volume outgrows what a spreadsheet can comfortably index and search — at which point the event-log *pattern* carries over directly (it maps cleanly onto an append-only table with a materialized "current state" view), so the migration is a storage-layer swap, not a redesign.

---

## 3. Voice synthesis: ElevenLabs vs. Twilio's built-in TTS

**The choice:** Route all spoken responses through ElevenLabs' streaming text-to-speech API, with Twilio's native `<Say>` kept only as a silent fallback.

**The context:** Twilio can synthesize speech natively with zero extra integration work — call `<Say>` with text and it speaks immediately. The catch: it sounds synthetic enough that callers notice they're talking to a machine within the first sentence, which undermines the entire premise of a *natural-feeling* phone agent.

**Why ElevenLabs, and why it took real tuning to make it work for this use case:**
ElevenLabs produces dramatically more natural speech — but its default settings are optimized for *quality* (e.g., narration, content creation), not for the *latency* a live phone conversation demands. Simply switching providers would have traded "sounds robotic" for "sounds great but the caller sits in silence for two seconds wondering if the call dropped" — arguably a worse outcome.

The actual work was identifying and tuning the specific levers that trade a small amount of voice fidelity for a large amount of speed:
- **Model choice: `eleven_flash_v2_5`** — ElevenLabs' model line built specifically for real-time / conversational latency, as opposed to their higher-fidelity narration-oriented models
- **`optimize_streaming_latency` set to its maximum value** — instructs the API to prioritize time-to-first-byte over absolute audio quality
- **Streaming MP3 output** rather than waiting for a complete file — audio starts reaching Twilio (and therefore the caller's ear) before synthesis has finished
- **Voice settings tuned for speech, not performance** — `style` processing disabled (it adds latency for an effect that's wasted on a phone line's compressed audio), speaker-boost disabled, stability and similarity tuned for consistency turn-over-turn rather than expressive range

**The fallback:** if the TTS proxy or the ElevenLabs API has a bad moment, Twilio's `<Say>` kicks in so the *call* never fails — only, briefly, its voice quality. A robotic sentence is a far better outcome than a dropped call.

---

## 4. Tool execution model: sandboxed code vs. delegated sub-workflows

**The choice:** Split the agent's seven tools into two execution models based on what each one actually needs to do — pure computation in sandboxed code nodes, I/O-bound operations delegated to full sub-workflows.

**The context (a real platform constraint, not a design preference):** n8n's `toolCode` nodes execute JavaScript inside a sandbox that explicitly blocks outbound networking — `fetch`, the internal `$helpers` HTTP utilities, and even `require('http')` are all unavailable. This is a deliberate security boundary on n8n's part, and it's non-negotiable from inside that node type.

**Why the split is the right answer (not a workaround):**
- Two of the seven tools — `get_current_datetime` and `query_menu` — are pure computation. They take an input, run logic over static or derived data, and return a result. They have *no reason* to need network access, and the sandbox is actually a good fit: fast, isolated, nothing to configure.
- The other five — every reservation operation — fundamentally cannot be pure computation. They need to read and write a live data store and, in one case, send an SMS through a third-party API. That requires running outside the sandbox.

The solution is structural rather than a workaround: those five tools are implemented as `toolWorkflow` nodes (n8n's mechanism for an agent to invoke an entire sub-workflow as a callable tool). The sub-workflow runs as a normal n8n workflow — full network access, full node library — and returns a clean structured result back to the calling agent.

**The payoff:** from the agent's perspective, all seven tools look identical — a name, a description, and a parameter schema. It has no awareness of, or need to care about, which ones are "just code" and which ones orchestrate multi-step operations against external services. The complexity is fully contained at the layer where it belongs.

---

## 5. Conversation control flow: one webhook vs. per-state endpoints

**The choice:** Route every event for every call — the very first ring, every spoken turn, and every silence timeout — through a single webhook endpoint, rather than building separate endpoints for "start of call," "ongoing conversation," and "timeout handling."

**Why this matters more than it sounds like it should:** Telephony platforms generate a surprising number of distinct event shapes for what a human would describe as "one phone call" — initial connection, speech results, no-input timeouts, hangup notifications. Building separate handlers for each is the natural first instinct, and it's a trap: the conversation logic (what should the agent know, what should it say) ends up duplicated — or worse, subtly diverging — across multiple code paths.

**The actual design:** A single, small `Prepare Input` step looks at whatever payload just arrived and reduces it to exactly one of three classifications — *first contact*, *caller said something*, or *caller went silent* — and emits a single, clean instruction for the agent either way. Every subsequent step in the pipeline (the agent, the tools, the TwiML builder) is completely unaware of which of the three cases produced its input; it just sees "here's what's happening this turn, respond appropriately."

**The payoff:** there is exactly **one** place in the entire system where "what kind of moment is this in the conversation?" gets decided. Adding a new case (say, handling a caller who presses a DTMF key) means extending that one classification step — not threading a new code path through the agent, the tools, and the response builder.

---

## 6. Returning-caller recognition: proactive lookup vs. reactive Q&A

**The choice:** Extract the caller's phone number from the very first webhook payload and instruct the agent — as part of its *initial* system instruction, before it has said a single word — to check whether this number matches an existing booking, and to shape its greeting accordingly.

**The alternative considered (and rejected):** Let the conversation start generically ("Hi, thanks for calling Shrimp & Co — how can I help?") and let the agent discover a returning caller reactively, e.g., if they mention an existing booking. This is simpler to implement and technically "works" — but it wastes the single biggest opportunity a phone system has to feel personal: the moment the call connects, the system *already knows* who's most likely calling. Making the caller introduce themselves anyway, when their number is sitting right there in the connection metadata, is a small thing that adds up to "feels like a call center" rather than "feels like they know me here."

**The implementation insight:** this didn't require a special "returning caller" code path at all — it required getting the *information* (the caller's number) into the agent's hands at the *right time* (before the greeting is generated) and trusting the agent's existing tool-calling and reasoning ability to do the rest. The "feature" is really just good sequencing of information flow, not new logic.

---

## What I'd Improve With More Time

Being candid about the current edges of the system:

- **Observability** — execution traces exist per-call inside n8n, but there's no aggregated view (call volume over time, common failure patterns, escalation frequency) that would help spot trends without manually reviewing individual runs
- **Load testing** — the system has been validated against real call patterns for a single-location restaurant; behavior under significantly higher concurrent-call volume (where Sheets' lack of true transactions would be tested harder) hasn't been stress-tested
- **Automated regression testing for conversation quality** — changes to the system prompt or tool descriptions are currently validated by manual test calls; a suite of recorded scenarios with automated transcript assertions would catch regressions faster as the prompt evolves
