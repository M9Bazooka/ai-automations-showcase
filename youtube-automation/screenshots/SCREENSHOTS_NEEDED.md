# Screenshots Needed — YouTube Automation

This folder is referenced from the README but intentionally ships without images yet. Adding visuals is the highest-leverage way to make this showcase land. None of these require exposing source code, system prompts, or credentials.

Suggested shots, roughly in order of impact:

1. **`orchestrator-canvas.png` — the master orchestrator node graph**
   A zoomed-out screenshot of the YT Master Orchestrator in the n8n editor, showing the overall shape: 3 triggers → merge → the 8 sequential `executeWorkflow` stages → the error rail. This instantly communicates "real, structured system" without revealing any node contents.

2. **`finished-video.png` — a frame from a generated video**
   A still from an assembled `<video_id>_final.mp4` showing the 1080p output with burned subtitle overlay. Proof the pipeline produces a real, watchable film — not just JSON.

3. **`sub-workflow.png` — one sub-workflow expanded** (e.g. Video Assembly)
   The normalize → concat → mix → subtitle-burn graph. Shows the FFmpeg-orchestration depth that the prose describes.

4. **`pipeline-state.png` — a `pipeline_state.json` run record (synthetic)**
   The per-stage timing/state object for one run, demonstrating the manifest-driven, stage-by-stage observability. Use a sample/test run, not anything tied to a real channel.

5. **`discord-notification.png` — the "Ready for Review" Discord embed**
   The success/ready embed posted at the end of a run, with the scheduled time and Studio link. Closes the loop visually.

6. **`thumbnail-variants.png` — the 3 generated thumbnails**
   The Flux-generated + ImageMagick-overlaid thumbnail options for one video, side by side.

---

## What NOT to include

- The **script system prompt** in full (the "Kurzgesagt + thriller + podcast" creative-direction source of truth) — it's the most engineered artifact here.
- Any **API keys, OAuth tokens, webhook URLs, or `$env` values** — blur/crop these out of every screenshot.
- Raw **source code** from Code nodes (FFmpeg command builders, polling logic, manifest construction) — the architecture doc already describes what they do.
- Any **real YouTube channel identity / analytics** you don't want associated with this showcase.

---

*Once captured, reference these from the README and either delete this file or keep it as a record of the visual-documentation process.*
