AILLM / Maeve — Migration Summary (Updated)

Last Updated: 2026-01-26
Project: Maeve (Discord-connected LLM agent)
Current Status: Stable, GPU-assisted, context-safe
Primary Goal: Seamlessly migrate, iterate, and extend a multi-layer AI system with persistent memory, safe personality evolution, and Discord-native behavior.

1. Project Overview

Maeve is a Discord-connected LLM agent designed to:

Learn casual human conversation patterns from Discord user↔user history

Respond naturally (short messages, emojis, slang, non-corporate tone)

Maintain continuity across restarts, crashes, and chat migrations

Evolve personality safely without drifting or self-rewriting core rules

Support manual controls, opt-outs, filters, and admin overrides

Operate reliably on constrained hardware (currently Quadro P400)

Be tunable without code changes via environment variables

The architecture prioritizes stability first, then quality, then performance.

2. Directory Structure (Current, Canonical)

Root

/media/altiazio/Storage/AI/

Personalities
data/personalities/
├─ base.yaml                # Immutable personality anchor (hard rules)
├─ system.yaml              # Runtime persona config + channel stats
├─ overrides.yaml           # Optional manual overrides
├─ state.json               # Learned global behavior state
├─ blocked_words.yaml       # Word/phrase censorship list

Conversation Memory
data/conversations/
├─ active/                  # Per-user JSONL transcripts
├─ summaries/               # Rolling summaries (persistent continuity)
├─ archived/                # Summary overflow (long-term archive)
├─ profiles/                # Per-user facts & preferences
│  ├─ <userid>.json
│  └─ <userid>.personality.json
└─ memory/                  # Episodic memories

Discord-Specific Data
data/discord/
├─ channels/
│  ├─ active/               # Channel JSONL context (user↔user)
│  ├─ summaries/            # Channel summaries
│  └─ archived/             # Channel overflow archive
└─ settings.json            # Per-guild/channel/user flags

3. Services
3.1 LLM API (Backend)

Service: aillm-api.service

Stack: FastAPI + Uvicorn + llama.cpp

Endpoints:

POST /generate

POST /ingest

POST /admin/summarize

Location:

services/llm_api/
├─ main.py        # API + generation logic (context-safe)
├─ memory.py      # Memory, summaries, profiles
└─ llama_runner.py# Model loading + hardware tuning

3.2 Discord Bot (Orchestrator)

Service: aillm-discord.service

Stack: discord.py

Responsibilities:

Discord event handling

Command routing

Ingest → API forwarding

Output sanitation & safety

Location:

services/orchestrator/
└─ discord_bot.py

4. Runtime / Hardware Configuration (Current Stable)
Environment-Based Tuning (.env)
MODEL_PATH=/media/altiazio/Storage/AI/models/llm/model3BQ4.gguf

N_CTX=8192
N_THREADS=12
N_GPU_LAYERS=17
N_BATCH=24

Notes

Quadro P400 (2 GB VRAM) confirmed stable with above values

Lowering N_BATCH from 64 → 24 eliminated intermittent HTTP 500 errors

All tuning is deliberately externalized to .env

No code changes required to retune hardware

5. Key Technical Decisions (Confirmed)
5.1 Personality Safety Model

base.yaml is immutable and never self-modified

Learned behavior stored separately (state.json, profiles)

Style sliders are bounded and decay toward baseline

Emotional shifts are subtle and controlled

No direct LLM self-rewriting allowed

5.2 Memory Model

Append-only JSONL for active conversations

Rolling summaries for restart continuity

Archived overflow prevents prompt bloat

User facts extracted only via explicit phrases

Channel context anonymized (u1, u2, …)

6. Discord Integration Details
6.1 Message Ingestion

All user↔user messages sent to /ingest

Author display names included

Mentions included (never raw IDs)

Discord IDs never surfaced to the model

6.2 Channel Context (“Roster” Model)

Users anonymized per channel

Example system prompt fragment:

Roster:
u1 = Alex
u2 = Preston
u3 = Jules


Preferred names override display names

Enables natural references (“What did Alex say?”)

7. Fixed Issues (Critical History)
7.1 Context Window Crashes

Issue: Requested tokens exceed context window

Fix: Dynamic max_tokens clamping based on actual prompt size

Result: No more context overflow crashes

7.2 GPU Instability

Issue: Random API 500 errors under load

Root Cause: Excessive N_BATCH for P400 VRAM

Fix: Reduced to N_BATCH=24

7.3 Privileged Discord Intent Crash

Removed intents.members = True

No privileged intents required

7.4 ID Leakage

IDs scrubbed from input and output

Roster anonymization enforced

Numeric ID patterns masked server-side and in Discord adapter

7.5 blocked_words.yaml Schema Mismatch

Loader updated to support:

blocked

blocked_words

words

blocklist

top-level list

8. Commands (Current)

Prefix: mae!

Command	Scope	Description
mae!status	User	Show user/channel state
mae!enable / disable	Channel	Enable/disable responses
mae!setai / unsetai	Channel	Mark channel as AI-chat
mae!optin / optout	User	Control ingestion
mae!summarize	User	Manual summarization
mae!help	User	Command list

Commands never forwarded to LLM

Command parsing occurs before prompt construction

9. Name Handling Logic

Bot asks exactly once: “What name should I call you?”

Controlled via profile.meta.asked_name

Stored in profile facts

Never repeated once set

10. Current Performance Notes

Long generations (60–100s) expected on P400

Stability prioritized over latency

Summarization prevents runaway prompt growth

GPU acceleration confirmed active

11. Continuing Development Goals (Next Phases)
Phase 1 — Dynamic Token Budgeting

Pre-flight token estimation before prompt assembly

Trim channel context → user context → summary (in that order)

Guarantees no overflow regardless of history size

Phase 2 — Discord UX Improvements

“Typing…” / thinking indicator

Placeholder message + edit-on-response

Reduced perceived latency

Phase 3 — Memory Quality (Not Quantity)

Episodic memory tagging (emotion, preference, decision)

Contextual injection based on relevance

Promotion of repeated episodic memories to profile facts

Phase 4 — Hardware Scaling

Drop-in support for stronger GPUs (RTX / P-series)

No code changes required; .env retuning only

12. Migration Checklist (For New Chat or Host)

Verify .env values (especially N_BATCH)

Confirm llama_runner config log line

Confirm summarization + archive working

Verify Discord ingest/output flow

Restart services:

sudo systemctl restart aillm-api.service
sudo systemctl restart aillm-discord.service


Verify:

sudo journalctl -u aillm-api.service -n 50
sudo journalctl -u aillm-discord.service -n 50


End of Migration Summary
