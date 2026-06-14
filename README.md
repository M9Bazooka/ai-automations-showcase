# AI Automations Portfolio — Mohammed Waliuddin / Aether AI

**Production AI agents and automations for real businesses — voice agents, content pipelines, and multi-platform publishing systems that run end-to-end with zero human steps in between.**

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![Anthropic Claude](https://img.shields.io/badge/Claude-D97757?style=for-the-badge&logo=anthropic&logoColor=white)
![OpenAI](https://img.shields.io/badge/GPT--4-412991?style=for-the-badge&logo=openai&logoColor=white)
![ElevenLabs](https://img.shields.io/badge/ElevenLabs-000000?style=for-the-badge&logo=elevenlabs&logoColor=white)
![Twilio](https://img.shields.io/badge/Twilio-F22F46?style=for-the-badge&logo=twilio&logoColor=white)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-34A853?style=for-the-badge&logo=googlesheets&logoColor=white)
![FFmpeg](https://img.shields.io/badge/FFmpeg-007808?style=for-the-badge&logo=ffmpeg&logoColor=white)

---

## 👋 About

I'm **Mohammed Waliuddin** — an MSc Artificial Intelligence student at **Aston University** and the builder behind **Aether AI**, where I design and ship production AI automations for real businesses.

These aren't demos — they're complete, working systems I've built end to end: a phone agent that runs a full reservation flow, a pipeline that produces finished videos, and a publisher that ships to two platforms. Each one is a full build — conversation design, API orchestration, generative-media generation, error handling, compliance, and the data layer — done solo.

This repository is a **showcase**: architecture, design decisions, and the engineering reasoning behind each system. It deliberately contains **no source code, credentials, or client data** — the goal is to show *how the pieces fit together*, not to hand over the implementation.

---

## 🗂️ The Projects

| Project | What it is | Stack | Status |
|---|---|---|---|
| **[🎙️ Voice Agent — "Prawn"](voice-agent/)** | An AI phone agent that handles a restaurant's calls, runs the full reservation lifecycle, and answers any menu question — in natural, low-latency speech. | Twilio · GPT-4.1-mini · ElevenLabs · n8n · Google Sheets · LangChain | ✅ **Built & working** |
| **[🎬 YouTube Automation — "YT Master Orchestrator"](youtube-automation/)** | An end-to-end pipeline that researches a trending topic, writes a documentary script, generates voiceover/music/SFX/images/AI video, assembles a 1080p film, and schedules it to YouTube. | n8n · Claude · ElevenLabs · fal.ai (Flux + Kling) · FFmpeg · YouTube Data API | 🟡 **Built — in development** |
| **[📌 Social Media Automation — "Beauty Affiliate Pipeline"](social-media-automation/)** | A scheduled content factory that turns one spreadsheet row into an AI-written, AI-generated vertical video and publishes it as both a Pinterest pin and an Instagram Reel — with built-in affiliate compliance. | n8n · Claude (Fable 5 + Sonnet) · Higgsfield · Pinterest API · Instagram Graph API · Google Sheets | 🟡 **Built — awaiting activation** |

> Each folder has its own full write-up: a `README.md` (the story + features), an `ARCHITECTURE.md` (the system internals), and a `screenshots/` guide.

---

## 🧠 What These Projects Have in Common

These are different domains — telephony, video production, social publishing — but they share a deliberate engineering philosophy:

- **LLM output is treated as untrusted.** Every system that asks a model for structured data validates the result and, where it matters, runs a dedicated repair pass or a deterministic code backstop before anything ships. A sloppy model response never reaches production.
- **Orchestration in n8n, hard logic in code.** Visual workflows handle integration, branching, and rapid iteration; the moment logic gets genuinely complex (capacity math, ledger folding, audio timing, compliance clamping), it moves into a Code node written as plain JavaScript.
- **Manifest-driven, debuggable-by-stage design.** Long pipelines emit small JSON manifests and write heavy artifacts to disk, so each stage can be run, tested, and inspected in isolation — and so a failure is diagnosable rather than mysterious.
- **Failure is a designed path, not an afterthought.** Circuit breakers, bounded polling loops, timeout fallbacks, error rails, and notification channels are load-bearing parts of the design — because these systems run autonomously, with no human watching each run.
- **Self-contained, reusable sub-workflows.** Every pipeline is decomposed into modular sub-workflows that can be developed and tested standalone, then orchestrated by a master workflow.

---

## 🛠️ Core Toolkit

| Layer | Technologies |
|---|---|
| **Orchestration** | n8n (self-hosted, Docker) — sub-workflow patterns, `executeWorkflow`, `splitInBatches` loops, Switch/Wait polling |
| **LLMs** | Anthropic Claude (Sonnet, Fable 5), OpenAI GPT-4.1-mini — function-calling agents, structured generation, self-healing JSON |
| **Voice & Audio** | ElevenLabs (TTS, Music, Sound-Generation), Twilio Voice & Messaging |
| **Generative Media** | fal.ai (Flux Pro, Kling Video), Higgsfield (Soul, Seedance), ImageMagick |
| **Media Processing** | FFmpeg / FFprobe — normalization, Ken Burns, audio mixing, subtitle burning |
| **Data & Publishing** | Google Sheets (event-sourced ledgers + job queues), YouTube Data API v3, Pinterest API v5, Instagram Graph API |
| **Notifications** | Discord webhooks, Gmail (approval flows + alerts) |

---

## 📬 Get in Touch

I'm available for freelance and contract work building AI agents and automations for businesses.

- **GitHub:** [@M9Bazooka](https://github.com/M9Bazooka)
- **Email:** [woven.cookie.1612@gmail.com](mailto:woven.cookie.1612@gmail.com)
- **Agency:** Aether AI

---

*Built by Mohammed Waliuddin — solo design, build, and deployment. Every project here is a real system, not a tutorial.*
