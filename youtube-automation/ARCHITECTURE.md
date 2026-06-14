[← Back to YouTube Automation](README.md) · [← Portfolio](../README.md)

# Architecture — YT Master Orchestrator

This document describes how the pipeline is put together: the master orchestrator, the 8 sub-workflows it drives, how data moves between stages, and how failures are contained. It describes the system *as-built* and deliberately stops short of source code.

---

## System Overview

The system is a **master/sub-workflow architecture** running entirely on self-hosted n8n. A single orchestrator workflow drives 8 specialized sub-workflows in sequence, each invoked via an `executeWorkflow` node. Heavy media never travels between nodes as payloads — each stage writes files to a per-run temp tree and passes small JSON manifests of *paths* downstream.

| Workflow | n8n ID | Nodes | Role |
|---|---|---|---|
| **YT Master Orchestrator** | `vkNcC763jfdeyH6b` | 29 | Drives the whole pipeline + error rail |
| Topic Research | `cDA5jO6Ux2YlkEMM` | 35 | Mine + score + expand a topic |
| Script Writing | `Fg1kmV8cW2s8rMZc` | 15 | Write + self-heal the documentary script |
| TTS Generation | `TWrNS6E1sINnFZXk` | 16 | Narration → master voiceover + timing |
| Music and SFX | `FlGmudBhQZnWnvmM` | 17 | Mood-grouped music beds + per-cue SFX |
| Image Generation | `GubavwiHta4ndDWJ` | 15 | Scene images + 3 thumbnails |
| Video Clip Generation | `EqqOoIsOPJJLANVo` | 14 | Ken Burns / Kling clip per scene |
| Video Assembly | `fcLQ25XIXh8f6fT4` | 28 | Normalize → concat → mix → subtitle |
| YouTube Upload | `xsBSZvawOxc8qIQR` | 16 | Resumable upload + scheduling |

Every sub-workflow has its own **Manual Trigger + Execute-Workflow Trigger merged at the top**, so each can be run standalone for development and testing, yet still be orchestrated end-to-end.

---

## The Master Orchestrator (29 nodes)

Linear flow with an error rail. Three triggers merge into one entry point; the only seed data is a generated `video_id` — everything else is fetched or generated.

```
Cron Trigger ─┐
Manual Trigger ─┼→ Merge Triggers → Generate Video ID → Create Folder Structure
Webhook Trigger ┘
   → Topic Research          → Log State: topic_research
   → Script Writing          → Log State: script_writing
   → TTS Generation          → Log State: tts_generation
   → Music & SFX             → Log State: music_sfx
   → Image Generation        → Log State: image_generation
   → Video Clip Generation   → Log State: video_clip_generation
   → Video Assembly          → Log State: video_assembly
   → YouTube Upload          → Log State: youtube_upload
   → Send Notification       → Log State: send_notification
   → Mark Pipeline Complete  → Discord Success → Log Completion

Every executeWorkflow node's error output ─→ Handle Error → Discord Notify
```

| Node | Type | What it does |
|---|---|---|
| Cron Trigger | `scheduleTrigger` | Fires Mon/Wed/Fri 09:00 UTC |
| Manual Trigger | `manualTrigger` | On-demand run |
| Webhook Trigger | `webhook` | `POST /webhook/yt-pipeline` |
| Merge Triggers | `merge (chooseBranch)` | Unifies the 3 triggers |
| Generate Video ID | `code` | Mints `vid_YYYYMMDD_HHMMSS` |
| Create Folder Structure | `code` | `fs.mkdirSync` builds the per-run temp tree + `output/` + `tracking/` |
| Topic Research … YouTube Upload, Send Notification | `executeWorkflow` ×9 | Calls each sub-workflow, waits for its result |
| Log State: `<stage>` ×9 | `code` | Updates `pipeline_state.json` stage timings |
| Mark Pipeline Complete | `code` | Sets state `completed` |
| Discord Success | `httpRequest` | Posts the ✅ embed |
| Log Completion | `code` | Appends `logs/completed_runs.json` |
| Handle Error | `code` | Writes `logs/errors.json` from any stage failure |
| Discord Notify | `httpRequest` | Posts the failure message |

---

## Sub-Workflow Internals

### 1 · Topic Research (35 nodes) — multi-source fan-out → AI funnel

```
(Manual / Execute Workflow Trigger) → Trigger Merge → fan out to 11 HTTP calls:
  → 5× Reddit subreddit  → 5× Filter (score ≥ 500) ─┐
  → HN topstories → Extract IDs → Split ⇄ HN Fetch → Filter HN (score ≥ 100) ─┤→ Merge Results (7 inputs)
  → 5× YT Suggest → 5× Parse JSONP → YT Suggestions Merge ────────────────────┘
  → Combine Results (dedupe) → Claude Score Topics → Parse Claude Response
  → Claude Expand Topic → Parse Topic Expansion
```

Sources: `r/todayilearned`, `r/history`, `r/UnresolvedMysteries`, `r/technology`, `r/Futurology`; Hacker News top stories; and 5 Google/YouTube autocomplete queries (JSONP). Two Claude calls: one scores every pooled candidate 1–10 on search potential / story strength / visual potential / uniqueness / virality and assigns a niche; the second expands the #1 topic into title, alt titles, hook, unique angle, 10 SEO keywords, and 5 research queries.

### 2 · Script Writing (15 nodes) — AI chain with self-healing branch

```
Trigger Merge → Claude Gather Facts → Store Research Facts → Build Script Prompt
  → Claude Write Script → Store Script → Parse and Validate Script → IF: Needs JSON Fix
        ├─ true  → Claude Fix JSON → Re-parse After Fix ─┐
        └─ false ─────────────────────────────────────────┼→ Merge Fix Paths → Finalize Script → Extract Asset Arrays
```

The system prompt encodes the channel's creative identity — **"Kurzgesagt clarity + thriller tension + podcast warmth"** — with explicit `[VISUAL-FLAT]`, `[VISUAL-CINEMA]`, `[SFX]`, and `[MUSIC]` tag formats that drive every downstream generation step. `Extract Asset Arrays` splits the validated script into `narration_sections`, `visual_prompts`, `sfx_cues`, and `music_directions`.

### 3 · TTS Generation (16 nodes) — two-phase loop

```
Trigger Merge → Prepare TTS Items → Split TTS Sections (batch 1)
   ├ loop: → Prepare Request → ElevenLabs TTS → Save Audio File → Wait 1.5s ↺
   └ done: → Enrich Section Items → Get Audio Duration (ffprobe) → Build Timing Manifest
        → Create Silence → Build Concat List → Concatenate Audio (ffmpeg) → TTS Output
```

The **timing manifest** built here (per-section start/end times + fixed 0.4s gaps) is the pipeline's shared clock — reused for music alignment, subtitle timing, and YouTube chapters.

### 4 · Music and SFX (17 nodes) — two parallel loops

```
Trigger Merge → Group Music Segments → Prepare Music Items → Split Music Segments (batch 1)
   ├ loop A (music): → Prepare Music Request → ElevenLabs Music → Save Music File → Wait 3s ↺
   └ done → Prepare SFX Items → Split SFX Cues (batch 1)
        ├ loop B (sfx): → Prepare SFX Request → ElevenLabs SFX → Handle SFX Result → Wait 2s ↺
        └ done → Build Manifests
```

`Group Music Segments` merges adjacent same-mood sections into a single instrumental bed. `Handle SFX Result` saves or marks-failed each cue so one missing sound effect can't break the run.

### 5 · Image Generation (15 nodes) — batched generate + parallel poll → thumbnails

```
Trigger Merge → Enhance Prompts → Split Image Batches (batch 5)
   ├ loop: → Submit to Fal Queue → Extract Request IDs ↺
   └ done → Poll and Download Images (Promise.all, 40×3s)
        → Prepare Thumbnails → Submit Thumbnails → Extract Thumbnail IDs
        → Poll and Download Thumbnails → Prepare Overlay Command → Add Text Overlays (ImageMagick) → Build Image Manifest
```

Submissions are batched 5-at-a-time to fal.ai's queue, then all request IDs are polled in parallel (`Promise.all`, ~120s budget). Thumbnails get text burned on via ImageMagick `convert`.

### 6 · Video Clip Generation (14 nodes) — per-scene branch with fallback

```
Trigger Merge → Classify Scenes → IF: Ken Burns
   ├ true  → Prepare Ken Burns → Ken Burns FFmpeg → Collect Ken Burns ─┐
   └ false → Upload Image to Fal → Submit to Kling → Collect Kling Submit ─┤→ Merge Clip Results
        → Poll and Download Kling (30×10s; Ken Burns fallback on timeout) → Build Clip Manifest
```

The cost-control heart of the pipeline: flat/long scenes get a free FFmpeg zoompan; cinematic scenes get paid Kling image-to-video; Kling timeouts silently fall back to Ken Burns.

### 7 · Video Assembly (28 nodes) — normalize → concat → audio mix → subtitle burn

```
Trigger Merge → Prepare Clips → Split Clips (batch 1)
   ├ loop: Build Normalize Command → Normalize FFmpeg → Collect Normalized ↺
   └ done: Build Concat File → Concat Videos → Collect Video Only
        → [music loop: Build/Process/Collect] → Build Music Concat → Concat Music → Collect Music Final
        → Create SFX Base → Build and Mix SFX → Build Mix Command → Mix Final Video
        → Collect Mixed Video → Build Drawtext Filter → Apply Overlays → Verify Output → Build Final Output
```

Normalizes every clip to 1080p/30fps, concatenates, fades + ducks music to 15%, mixes voice + music + SFX with `amix`, burns section subtitles with `drawtext`, and ffprobe-verifies the final `<video_id>_final.mp4`.

### 8 · YouTube Upload (16 nodes) — resumable upload + scheduling

```
Trigger Merge → Build YouTube Metadata → Prepare Upload Metadata → Initiate Resumable Upload
   → Extract Upload URL → Upload Video (curl PUT) → Upload Thumbnail → Build Upload Result
   → Calculate Next Publish Slot → Set Video Status (PUT) → Collect Status Result
   → Update Tracking Log → Build Discord Payload → Send Discord Notification
```

Builds SEO metadata + chapter timestamps + tags, performs a resumable upload via the YouTube Data API v3, sets the custom thumbnail, computes the next conflict-free publish slot, sets privacy + `publishAt`, records to `tracking/log.json`, and posts a "🎬 Video Ready for Review" Discord embed with a Studio link.

---

## Data Flow

**Input:** a trigger event. The only seed data is the generated `video_id`; everything else is fetched or generated.

| Stage | In | Out |
|---|---|---|
| Init | trigger | `{ video_id }` + working directories |
| Topic Research | — | expanded topic object (title, hook, angle, keywords, research queries) |
| Script Writing | topic | structured script JSON → `narration_sections`, `visual_prompts`, `sfx_cues`, `music_directions` |
| TTS | narration sections | per-section MP3s, `timing_manifest`, `master_voiceover.mp3` |
| Music & SFX | music directions + timings, sfx cues | `music_manifest`, `sfx_manifest` |
| Images | visual prompts | scene PNGs + 3 thumbnails, `image_manifest` |
| Clips | image manifest + prompts | per-scene MP4s, ordered `clip_manifest` |
| Assembly | clips + voiceover + music + sfx + timing | `<video_id>_final.mp4` (+ duration/size) |
| Upload | final video + metadata + timing (chapters) + thumbnail | YouTube video ID/URL, scheduled slot, tracking log, Discord embed |

**Output:** a scheduled/private YouTube video, the finished MP4 in `output/`, tracking-log entries, and Discord notifications.

---

## Error Handling & Resilience

- **Pipeline-level error rail.** Every `executeWorkflow` node's secondary (error) output routes to `Handle Error`, which logs to `logs/errors.json` (video_id, stage, message, timestamp) and posts a Discord failure message.
- **Stage state tracking.** `pipeline_state.json` records `current_stage`, `completed_stages`, and per-stage timings — so a stopped run is diagnosable. (The orchestrator does not auto-resume; you re-run from the start.)
- **Script JSON self-repair.** `IF: Needs JSON Fix` → `Claude Fix JSON` → re-parse, with warnings appended.
- **Kling timeout fallback.** If Kling exceeds ~5 min (30×10s polls), the node auto-generates a Ken Burns clip and tags a warning — the pipeline keeps moving.
- **fal.ai polling budgets.** ~120s (40×3s) per image/thumbnail; failures are counted into manifest warnings rather than thrown.
- **SFX resilience.** Failed SFX fall back to a silent base (`copyFileSync`) so the mix never breaks.
- **Per-shell guards.** Every FFmpeg/ffprobe shell-out captures `exitCode`/`stderr` in a try/catch and reports status downstream instead of crashing.
- **Upload guards.** Explicit error if the resumable upload returns no `Location` header; thumbnail upload skipped gracefully if no video ID.

---

## Interesting Technical Decisions

1. **Hybrid cost-aware visuals** — free FFmpeg Ken Burns for flat/long scenes, paid Kling for cinematic ones, Ken Burns again as the timeout fallback. Caps cost and guarantees a clip per scene.
2. **LLM output as untrusted, with a repair loop** — validate structure/word count, run a dedicated Claude "fix JSON" pass only when needed (IF-gated), then re-validate.
3. **Manifest-driven, file-on-disk** — each stage emits a small JSON manifest and writes heavy media to a `video_id`-scoped temp tree; downstream stages read paths, not payloads. The shared `timing_manifest` is reused for audio gaps, music alignment, subtitle timing, and YouTube chapters.
4. **Self-contained sub-workflows** — every sub-workflow has its own Manual + Execute-Workflow trigger merged at the top, so each is independently developable yet orchestratable.
5. **Conflict-free auto-scheduling** — scans up to 21 days ahead for the next open Mon/Wed/Fri slot, honors a 15-minute buffer, persists taken slots, and falls back to "+7 days".

---

## Host Dependencies

The machine running n8n must have **ffmpeg, ffprobe, ImageMagick (`convert`), and curl** on `PATH` — the Code nodes shell out to all of them via `execSync`. Node built-ins used in Code nodes: `fs`, `https`, `child_process`, `path`.

## Secrets Model

Every external call injects `$env.*` directly into HTTP headers/bodies — `ANTHROPIC_API_KEY`, `ELEVENLABS_API_KEY`, `ELEVENLABS_VOICE_ID`, `FAL_API_KEY`, `YOUTUBE_ACCESS_TOKEN`, `DISCORD_WEBHOOK_URL`, `OUTPUT_DIR`, `AUTO_PUBLISH`. No n8n credential objects are used by these workflows. The `YOUTUBE_ACCESS_TOKEN` is a manually-pasted OAuth token that expires hourly — flagged as the #1 likely failure (401); the long-term fix is to switch the upload nodes to a proper n8n Google OAuth2 credential.

---

*Part of the [Aether AI automations portfolio](../README.md).*
