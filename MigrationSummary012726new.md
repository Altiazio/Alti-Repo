# Maeve / AILLM Migration Summary (Checkpoint – Repo Split Complete)

**Date:** 2026-01-27  
**Status:** Architecture stabilized; physical LLM box pending hands-on recovery

---

## 1. High-level architecture

### Current
- **VM (Proxmox)**
  - Role: Orchestrator only
  - Runs: Discord bot (LLM-BOT)
  - Purpose: Ingress, command handling, API calls
  - Status: Operational

- **Physical machine (AI box)**
  - Role: LLM runtime + API host
  - Intended to run: LLM-API, llama.cpp, memory + summarization
  - Status: Boot/installer failure, not remotely accessible
  - Next action: Local helper required to finish Ubuntu install and enable SSH

### Target
- VM = Orchestrator
- Physical = Inference + memory
- Communication via HTTP (generate / ingest / summarize)

---

## 2. Repository layout

### LLM-API
- Repo: Altiazio/LLM-API
- Runtime root: /media/altiazio/Storage/AI
- Contains: API, llama runner, memory logic
- Excludes:
  - Orchestrator
  - Models
  - Logs (ignored)

Status:
- SSH auth verified
- main branch pushed
- test branch available

---

### LLM-BOT
- Repo: Altiazio/LLM-BOT
- Path: /media/altiazio/Storage/AI/services/orchestrator
- Contains:
  - discord_bot.py
  - .gitignore
  - .env.example
  - scripts/bootstrap_runtime_dirs.sh

Runtime expectations (outside repo):
/media/altiazio/Storage/AI/data/discord/settings.json  
/media/altiazio/Storage/AI/data/personalities/blocked_words.yaml  
/media/altiazio/Storage/AI/logs  
/media/altiazio/Storage/AI/debug  

Status:
- main + test branches created
- One-time force push completed
- SSH key added globally

---

## 3. Environment variables

### Orchestrator
- .env local only (ignored by git)
- Contains DISCORD_TOKEN and API URLs

### API
- .env will live on physical box
- Contains model paths and runtime tuning

---

## 4. Data and models

- Models: manual transfer only (not in git)
- Logs/debug/data: runtime files, ignored long-term
- Bootstrap script ensures required directories exist

---

## 5. Disk layout intent

**NVMe**
- OS
- Python venvs
- LLM-API
- llama.cpp
- Hot models
- Temp/cache

**SATA SSD**
- Archived models
- Conversations
- Logs
- Audio (future STT/TTS)
- Vision frames
- Backups

Initial bring-up will place everything on NVMe.

---

## 6. Physical box status (blocking)

- Clonezilla restore failed
- Ubuntu installer partially ran then errored
- System not in clean boot state
- No SSH / remote desktop access

Next requirement:
- Local user finishes Ubuntu install
- Networking verified
- SSH enabled (NoMachine optional)

---

## 7. What has NOT happened yet

- No repos cloned to physical box
- No API service deployed on hardware
- No model transfer
- No systemd services recreated

Intentionally deferred until physical box is stable.

---

## 8. Next steps (next chat)

1. Physical install completion
2. Enable SSH / remote access
3. Clone LLM-API onto physical box
4. Run bootstrap_runtime_dirs.sh
5. Install deps, copy models
6. Start API
7. Point orchestrator VM to physical API
8. End-to-end Discord test

---

This document represents a clean migration checkpoint.
