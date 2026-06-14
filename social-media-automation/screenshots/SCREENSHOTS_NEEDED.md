# Screenshots Needed — Social Media Automation

This folder is referenced from the README but intentionally ships without images yet. Adding visuals is the highest-leverage way to make this showcase land. None of these require exposing source code, system prompts, or credentials.

Suggested shots, roughly in order of impact:

1. ✅ **`workflow-canvas.png` — the full 67-node workflow** *(captured — now embedded at the top of the README)*
   A zoomed-out screenshot of the whole pipeline in the n8n editor, ideally showing the 6 sticky-note sections (Overview, Sourcing, AI Content, Video, Approval, Platforms) that document it in-canvas. Communicates scope and structure at a glance.

2. **`generated-reel.png` — a frame from a generated video**
   A still from a Higgsfield Seedance 9:16 output for one product. Proof the pipeline produces a real, postable vertical video.

3. **`content-sheet.png` — the Google Sheet job queue (synthetic data)**
   The "Content" tab showing the row lifecycle columns (`status`, `priority`, `product_name`, `affiliate_url`, `pinterest_status`, `instagram_status`, asset URLs…) populated with **clearly fake** product rows. Makes the "sheet is queue + DB + log" decision tangible.

4. **`approval-email.png` — the Approve/Reject email**
   The HTML approval email with video/cover preview and the two buttons. Demonstrates the human-in-the-loop gate. Use a test product, not a real brand campaign.

5. **`run-summary.png` — a per-run summary email**
   The Pinterest ✅/⏭/❌ · Instagram ✅/⏭/❌ report with the consecutive-failure count. Shows the dual-platform verdict model.

6. **`config-node.png` — the Config control panel**
   The `Config` Set node with its ops toggles (`approvalEnabled`, `igDailyCap`, `circuitBreakerThreshold`, etc.). Shows the "live ops panel" design — blur any real `igUserId` / email.

---

## What NOT to include

- The **Claude content system prompt** in full (it encodes the copywriting + compliance logic).
- Any **API keys, OAuth tokens, the real Sheet ID, `igUserId`, or affiliate URLs** — blur/crop these out.
- Raw **source code** from Code nodes (compliance backstop, circuit-breaker logic, polling loops) — the architecture doc already describes them.
- **Real customer/brand campaign data** — use synthetic test products for any sheet or email screenshot.

---

*Once captured, reference these from the README and either delete this file or keep it as a record of the visual-documentation process.*
