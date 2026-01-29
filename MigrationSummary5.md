# 🧭 Maeve Project — Migration Summary (Physical Box 24.04 Cutover)

## 📌 Current State (Checkpoint Summary)

### Hardware — Physical AI Box
- **CPU:** AMD Ryzen 5 5500
- **RAM:** 32 GB DDR4 (2×16 GB)
  - Stable configuration: **2933 MHz**
  - Higher speeds caused reproducible memory corruption
  - `memtester` passes clean at 2933 MHz
- **GPU:** AMD Radeon RX 7900 XT (RDNA3 / GFX1100)
- **Storage:** NVMe
  - Project root: `/media/altiazio/Storage/AI`
- **Network:** Reachable from orchestrator VM and local LAN

---

### Operating System Status
- **Previous OS:** Ubuntu 22.04 LTS (Jammy)
  - Significant issues encountered:
    - Vulkan tooling gaps
    - `glslc` / `shaderc` unavailable or outdated
    - GLIBC / libstdc++ mismatches
    - ROCm instability and dpkg corruption
    - Vulkan shader generation hangs
- **Decision:**  
  ✅ **Fresh install of Ubuntu 24.04 LTS on the physical AI box**

Ubuntu 24.04 provides:
- Newer Mesa + RADV (better RDNA3 stability)
- First-class Vulkan tooling (`glslc`, `shaderc`, `spirv-tools`)
- Newer glibc / libstdc++
- Significantly reduced build friction for `llama.cpp` Vulkan backend

---

## 🔥 Root Causes Identified

### Memory Instability (Major Contributor)
- Silent memory errors were corrupting:
  - pip wheel builds
  - Vulkan shader compilation
  - ROCm packages
  - gcc / ninja builds
- Resolution:
  - Disable A-XMP
  - Downclock RAM to 2933 MHz
  - System stability restored

---

### GPU Backend Evaluation
- **ROCm**
  - Attempted on 22.04
  - Caused dpkg failures, segfaults, and incomplete installs
  - Abandoned for now

- **Vulkan (Chosen Path)**
  - Works reliably with RX 7900 XT
  - Actively supported by `llama.cpp`
  - Requires modern userspace (24.04+)
  - Selected as primary GPU backend

- **CPU Backend**
  - Retained as fallback / hybrid mode

---

## 🧠 llama-cpp-python Status

- Target version: **llama-cpp-python 0.3.2**
- Vulkan backend: `ggml-vulkan`
- On 22.04:
  - Build failures traced to tooling + OS age
- Expectation on 24.04:
  - Clean build using native Vulkan tooling
  - No shims or manual shader compilation required

---

## 🤖 Model Strategy (Locked In)

### Initial Deployment
- **Primary model size:** **8B**
  - Best balance of:
    - Speed
    - Stability
    - GPU memory headroom
    - Development velocity
- Intended as the default Maeve runtime model

### Scale-Up Plan
- **Optional upgrade target:** **13B**
  - Will be considered if:
    - 8B shows reasoning or coherence limitations
    - GPU memory usage proves stable
    - Latency remains acceptable
- Model scaling will be done *after* system stability is confirmed

❗ Models larger than 13B are explicitly **out of scope** for early phases.

---

## 🏗️ Architecture Decisions (Now Fixed)

### Physical AI Box
- Runs:
  - LLM inference
  - GPU workloads
  - `LLM-API`
  - Model files
  - User memory / context systems
- OS: **Ubuntu 24.04 LTS**
- Backend: **Vulkan (primary), CPU (fallback)**

---

### Orchestrator VM
- OS: **Ubuntu 22.04 LTS (unchanged for now)**
- Role:
  - Discord bot
  - Orchestration / routing logic
  - Control plane
- Decision:
  - **Do NOT upgrade VM yet**
  - Keeps a known-good baseline
  - Reduces debugging surface
- Planned upgrade:
  - After Maeve is stable
  - Possibly after visual dashboard is complete

---

## 🔗 Integration Notes
- VM contains authoritative project files
- Physical box requires:
  - New SSH keypair
  - GitHub key registration
- VM ↔ Physical communication will use API calls, not shared runtime

---

## 🚀 Next Execution Phases

### Phase 1 — Fresh OS Install (Physical)
- Install Ubuntu 24.04 LTS
- Install required packages:
  - `vulkan-tools`
  - `shaderc`
  - `spirv-tools`
  - `libvulkan-dev`
  - build essentials

### Phase 2 — Maeve Bring-Up
- Restore project to:
  ```
  /media/altiazio/Storage/AI
  ```
- Create fresh virtual environment
- Install `llama-cpp-python` with Vulkan enabled
- Validate:
  - Model load
  - Stable inference
  - API availability

### Phase 3 — Integration
- Create and register SSH keys
- Connect orchestrator VM to physical box
- Validate end-to-end message flow

---

## ⏸️ Explicitly Deferred
- ROCm
- Orchestrator VM OS upgrade
- RAM overclocking
- Models >13B
- Multi-GPU or exotic backends

---

## 🎯 Maeve Project Goals (Unchanged)
- Text ↔ Text interaction
- Speech ↔ Speech interaction
- Long-term user memory
- Persistent emotional state
- Visual context ingestion (emulated display / screenshots)
- Hybrid smart execution (GPU + CPU)

---

## ✅ Final Note
Hardware stability issues are resolved.  
Platform choice is corrected.  
Migration path is clear and staged.

**Maeve is now positioned for a clean 24.04 restart and controlled scale-up.**
