# AILLM / Maeve — Migration Summary (Updated)

**Last Updated:** 2026-01-27  
**Project:** Maeve (Discord-connected LLM agent)  
**Current Status:** Stable; memory extractor running; hybrid global + per-guild memory; envs separated for future split-hardware deployment.

---

## 1) What Maeve is
Maeve is a Discord-connected LLM agent designed to:
- Respond naturally in Discord (short, casual, emojis, non-corporate).
- Persist conversation continuity across restarts (JSONL + summaries + archives).
- Build memory (facts/preferences) automatically over time with conservative extraction + conflict handling.
- Avoid ID leakage (Discord IDs scrubbed / roster anonymization).
- Be tunable mostly via `.env` without code edits.

---

## 2) Canonical paths (current)
Root:  
`/media/altiazio/Storage/AI/`

Key data:
- `data/`  
  - `entities.json` (global + scoped entity memory store)
  - `conversations/` (per-user JSONL + summaries + archives)
  - `discord/channels/active/` (per-channel JSONL used for context & memex)
  - `profiles/` (preferred names / per-user facts, if enabled/used)

Services:
- `services/llm_api/`  
  - `main.py` (FastAPI endpoints, prompt build, token budgeting hooks)
  - `memory.py` (memory, summarization, extractor + hybrid scoping)
  - `llama_runner.py` (model load + runtime tuning)
- `services/orchestrator/`  
  - `discord_bot.py` (Discord ingest + routing + commands)

---

## 3) Services & endpoints
### 3.1 LLM API (Backend)
Service: `aillm-api.service`  
Stack: FastAPI + Uvicorn + llama.cpp

Endpoints:
- `POST /generate` (reply generation)
- `POST /ingest` (channel message ingest)
- `POST /admin/summarize`
- `GET /debug/state` (optional, for debug visibility)

### 3.2 Discord Bot (Orchestrator)
Service: `aillm-discord.service`  
Stack: discord.py

Responsibilities:
- Capture Discord messages + mentions + author names
- Send every message to API `/ingest`
- Route bot mentions / wakewords to `/generate`
- Apply channel enable/disable rules + command parsing

---

## 4) Environment configuration (IMPORTANT)
### Goal: split-hardware ready
Soon: Orchestrator stays on current server; AILLM API moves to separate “AI box”.

**Rule:** API must be self-contained and not depend on orchestrator env.

### 4.1 API env (authoritative for model + memory extractor)
File:  
`/media/altiazio/Storage/AI/services/llm_api/.env`

Contains:
- model path / backend parameters (CPU/GPU, n_ctx, n_batch, threads, gpu layers)
- **MEMORY_EXTRACTOR_* settings** (now API-local)
- data paths (DATA_DIR, ENTITIES_PATH, etc.)

Systemd: `aillm-api.service` loads only:
- `EnvironmentFile=/media/altiazio/Storage/AI/services/llm_api/.env`

### 4.2 Orchestrator env (Discord + routing only)
File:  
`/media/altiazio/Storage/AI/services/orchestrator/.env`

Contains:
- `DISCORD_TOKEN`
- `LLM_API_URL`, `LLM_INGEST_URL`, etc.
- wakewords / command prefix
- channel allow/ignore lists

---

## 5) Memory model (current)
### 5.1 Channel context
- Discord user↔user messages are appended to channel JSONL (`/ingest`).
- Channel roster anonymization can be used in prompt (“u1/u2 mapping”) to avoid raw IDs.

### 5.2 Entities store (global + guild scope)
- `entities.json` stores:
  - global user facts/preferences/lists (portable across servers)
  - per-guild overlays for tone/humor/emoji/nicknames/inside jokes

### 5.3 Extractors
There are two layers:
1) **Rule-based extraction on every `/ingest`** (cheap, always-on)
2) **LLM memory extractor (“memex”)** (runs when enabled/gated by thresholds)

Recent fixes/changes:
- `resolve_display_name()` added/standardized to prevent memex NameError crash.
- Hybrid scoping logic supports global + per-guild memory.
- Pronoun/subject anchoring improvements to attach follow-ups to the correct referenced user (work-in-progress; still improving attribution heuristics).

---

## 6) UX / behavior notes
- A “repeat conversation” preamble was removed/filtered so Maeve doesn’t claim repetition unless strongly supported.
- Placeholder “…” message editing was intentionally skipped (Discord default typing indicator is acceptable for now).

---

## 7) Performance notes
- Load time and latency are increasing as features accumulate and model/context grow.
- Immediate priorities (next phase) for performance:
  - reduce prompt size earlier (token budgeting/trim order)
  - lower memex frequency (MEMORY_EXTRACTOR_EVERY_N) or min-chars gate
  - ensure summarization/archive keeps channel logs bounded
  - optional: move AILLM to stronger GPU box

---

## 8) Operational commands
Restart:
```bash
sudo systemctl restart aillm-api.service
sudo systemctl restart aillm-discord.service
```

Logs:
```bash
sudo journalctl -u aillm-api.service -n 80 --no-pager
sudo journalctl -u aillm-discord.service -n 80 --no-pager
```

---

## 9) Migration checklist (new chat OR new hardware)
1) Copy API service folder + data:
   - `services/llm_api/`
   - `data/` (especially `entities.json`, conversations, channel logs)
2) Ensure API `.env` is present and correct on the AI box:
   - model path
   - GPU/CPU parameters
   - MEMORY_EXTRACTOR_* enabled + tuned
3) On orchestrator host: update API URLs to the new AI box:
   - `LLM_API_URL`, `LLM_INGEST_URL`, etc.
4) Validate end-to-end:
   - Discord messages append to channel JSONL
   - `entities.json` updates after a few messages
   - `/generate` returns replies
   - no memex NameError crashes in logs

---

## 10) Next development goals (in order)
1) Dynamic token budgeting (preflight size + trim order)
2) UX polish (latency masking, better status/debug commands)
3) Memory quality (promotion/decay/conflict, better attribution)
4) Hardware scaling (AI box GPU + RAM)
5) Audit token flow / exact trim logic
6) Review `data/personalities/base.yaml` for future tuning
7) Add metrics + debug (simple dashboard; later Grafana)
