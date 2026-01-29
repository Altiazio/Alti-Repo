AILLM / Maeve – Migration Summary

Date: 2026-01-25
Project: Maeve (Discord-connected LLM agent)
Primary Goal: Migrate and stabilize a multi-layer AI system with persistent personality, Discord ingestion, and safe adaptive behavior.

1. Project Overview

Maeve is a Discord-connected LLM agent designed to:

Learn casual human conversation patterns from Discord user↔user message history

Respond naturally (short messages, emojis, slang, non-corporate tone)

Evolve personality safely over time without drifting out of bounds

Maintain continuity across restarts/crashes

Support manual controls, opt-outs, and filters

Eventually become fully event-driven (not just ping-based)

The system is designed to be modular, restart-safe, and expandable across multiple servers and channels.

2. Directory Structure (Current)

Root:

/media/altiazio/Storage/AI/

Personalities
data/personalities/
├─ base.yaml                # Stable personality anchor (hard rules, defaults)
├─ system.yaml              # Runtime persona config + channel stats
├─ overrides.yaml           # Manual overrides (optional / possibly removable)
├─ state.json               # Learned global state (machine-written)
├─ blocked_words.yaml       # Word/phrase censor list

Conversation Memory
data/conversations/
├─ active/                  # Per-user JSONL transcripts
├─ summaries/               # Rolling summaries (reboot continuity)
├─ archived/                # Summary overflow archive
├─ profiles/                # Per-user facts/preferences/meta
│  ├─ <userid>.json
│  └─ <userid>.personality.json
└─ memory/                  # Episodic memories

Discord-Specific Logs
data/discord/
├─ channels/
│  ├─ active/               # Channel JSONL logs (user↔user context)
│  ├─ summaries/
│  └─ archived/
└─ settings.json            # Per-guild/channel/user flags

3. Services
API (LLM backend)

Service: aillm-api.service

FastAPI + Uvicorn

Endpoint: POST /generate, POST /ingest

Path:

services/llm_api/
├─ main.py
├─ memory.py
└─ llama_runner.py

Discord Bot (Orchestrator)

Service: aillm-discord.service

discord.py client

Handles:

Discord events

Commands

Ingest → API

Output sanitation

Path:

services/orchestrator/
└─ discord_bot.py

4. Key Technical Decisions
4.1 Personality Safety Model

base.yaml = immutable anchor personality

Learned behavior stored separately (state.json, profiles)

Style sliders bounded (0-1)

Emotional shifts are subtle and decay to baseline

No direct self-rewriting of base persona

4.2 Memory Model

Active JSONL → append-only transcripts

Rolling summary → persistent continuity across restarts

Archived overflow → long-term memory without prompt bloat

User facts only extracted via explicit phrases (“my name is”, “call me”)

5. Discord Integration Details
5.1 Message Ingestion

All user↔user messages ingested via /ingest

Author display names included

Mentions included but author IDs never surfaced to the model

5.2 Channel Context (“Roster” Model)

Channel history anonymized to u1, u2, …

System prompt includes:

Roster: u1=Alex, u2=Preston, u3=Jules


Preferred names from user profiles take precedence

Allows “What did Alex say?” without requiring pings

6. Fixed Issues (Important)
6.1 Privileged Discord Intents Crash

Error: PrivilegedIntentsRequired
Fix: Removed intents.members = True
No privileged intents required for current functionality.

6.2 Bot Triggering on Any Mention

Bug: Bot thought any ping was a ping to itself
Fix: Trigger only on literal bot mention tokens:

<@bot_id> or <@!bot_id>

6.3 Discord ID Leakage

IDs appeared as [u:123...] or raw numbers

Fixes:

IDs never injected into prompts

Roster anonymization

Hard system rule: never output IDs

Output scrubber replaces 15–20 digit numbers with [id]

6.4 blocked_words.yaml Ignored

Root cause: Schema mismatch

File used blocked: key

Loader expected blocked_words: or list

Resolution:

Loader updated to support:

blocked

blocked_words

words

blocklist

top-level list

Masking enforced server-side + last-chance in Discord bot

Whole words only; phrases supported

7. Commands (Current / Planned)

Prefix: mae!

Command	Scope	Description
mae!status	User	Show current bot/channel/user state
mae!enable / disable	Channel	Enable/disable responses
mae!setai / unsetai	Channel	Mark channel as AI-chat
mae!optin / optout	User	Control message ingestion
mae!summarize	User	Manual summary trigger
mae!help	User	Command list

Commands are never forwarded to the LLM.

8. Name Handling Logic

Bot asks exactly once:

“What name should I call you?”

Controlled via profile.meta.asked_name

Once set, never repeated

Preferred name stored in:

profiles/<userid>.json → facts.name

9. Performance Notes

Slow Discord responses tied to limited hardware

Mitigations applied:

Increased timeouts

Reduced max tokens in Discord context

Planned migration to stronger hardware

10. Migration Checklist

Verify blocked_words.yaml schema compatibility

Confirm roster mapping works across channels

Confirm ID scrubber prevents all numeric leakage

Validate command routing vs prompt routing

Confirm /generate and /ingest endpoints

Restart both services:

sudo systemctl restart aillm-api.service
sudo systemctl restart aillm-discord.service


Verify via:

sudo journalctl -u aillm-api.service -n 50
sudo journalctl -u aillm-discord.service -n 50

11. Future Work (Post-Migration)

Fully event-driven response model

Emoji/context weighting via channel stats

Multi-message output splitting for natural flow

Long-term preference learning (hobbies, likes)

Optional fine-tuning from curated Discord history

End of Migration Summary
