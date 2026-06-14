# Screenshots Needed

This folder is referenced from `README.md` but intentionally ships without images yet — adding real visuals is the highest-leverage thing left to make this showcase land. None of these require exposing source code; they're all either UI views, sanitized diagrams, or your own voice/writing.

Suggested shots, roughly in order of impact:

1. **`hero.png` — a recording of a real call** (or a realistic re-creation)
   The single most convincing artifact a voice-agent project can have is *hearing it work*. A 30–60 second Loom/screen recording of an actual call — checking availability, booking a table, getting the SMS confirmation — linked from the top of the README, does more than any amount of written description. If using a real recorded call, make sure the guest's personal details (name, number) are clearly anonymized or it's a call you placed yourself for demo purposes.

2. **`n8n-workflow.png` — the main workflow canvas**
   A screenshot of the main conversation workflow's node graph in the n8n editor (zoomed out enough to show the overall shape: webhook → prepare input → agent → tools → TwiML → response). This is a visual that immediately communicates "this is a real, structured system" to a technically literate viewer — and it shows orchestration skill without revealing any node *contents* (system prompts, credentials, business logic). Collapse or blur node parameter panels before capturing.

3. **`execution-trace.png` — a single call's execution trace**
   n8n's per-run execution view, showing the sequence of nodes that fired for one real call and the high-level shape of data flowing between them (timing, success/failure status). This demonstrates the operational transparency described in the architecture doc — "I can see exactly what happened on every call" is a strong signal of production maturity. Redact any payload contents that include real customer data (names, phone numbers, addresses) before sharing.

4. **`sheet-ledger.png` — the reservation ledger (with synthetic data)**
   A view of the Google Sheet showing the event-sourced row structure (`ID | Name | Phone | Date | Time | Party Size | Notes | Status | Created At`) — but populated with **clearly fake** entries (e.g., "Test Guest", "+44 7000 900000", obviously placeholder dates) rather than real customer data. This makes the "event log, not a database" design decision tangible rather than abstract.

5. **`sms-confirmation.png` — a booking confirmation text**
   A screenshot of the SMS a guest receives after booking — again, sent to your own number as a test, not a real customer's. This closes the loop visually: "the agent didn't just say it booked the table, here's the proof that landed in someone's pocket."

6. **`twilio-console.png` — the call log in the Twilio console**
   A view of the Twilio dashboard showing real inbound call volume/duration over time (numbers and content blurred as needed). This is strong evidence of the "currently live, handling real traffic" claim — concrete proof beats a status badge.

---

## What NOT to include

- Anything showing the **system prompt** in full (it's the most carefully engineered artifact in the project and is effectively the "secret sauce" of the conversation design)
- Anything showing **credentials, API keys, account IDs, or webhook URLs** — blur or crop these out of every screenshot before saving
- **Real customer names, phone numbers, or booking details** — use synthetic test data for any data-layer screenshot
- Raw **source code** from the Code nodes (menu engine, TwiML builder, ledger-folding logic) — the architecture and decisions documents already describe what these do; showing the code itself defeats the purpose of a no-source-code showcase

---

*Once captured, reference these images from `README.md` (the commented-out hero section at the top is already wired up for `hero.png` and `n8n-workflow.png`) and remove this file, or leave it as a record of the visual-documentation process — either is a reasonable choice for a public-facing repo.*
