---
name: wsl-local-engine-watchdog
description: "WSL2 local engine watchdog — STRICT EXCLUSIVITY arbiter: only one local LLM engine runs at a time. Dynamic mmproj for ANY model with dual-prompt mode (agent + local fallback)."
version: 7.1.0
author: Hermes Agent
tags: [wsl, llama.cpp, vllm, watchdog, vram, provider-switch, local-engine, systemd, arbiter, exclusivity, mmproj, vision, multimodal, offline]
metadata:
  hermes:
    related_skills: [llama-cpp-wsl-ops]
---

# WSL Local Engine Watchdog (Strict Exclusivity + Dynamic mmproj)

A Python watchdog for WSL2 that **auto-starts and auto-stops local inference engines** based on which Hermes provider you select. **Enforces strict exclusivity — only one engine service is ever active at a time** to guarantee VRAM safety.

**Dynamic mmproj (vision):** The watchdog writes an `EnvironmentFile` (`~/scripts/mmproj_env.conf`) before starting the engine. If `/tmp/<provider>_mmproj` exists, it includes `--mmproj <path>`; otherwise the engine starts without vision. **Works for ANY provider, ANY model.**

**Dual-Prompt Mode:** The mmproj question is asked in two ways:
1. **Agent prompt** — When a capable (cloud) agent is available, it uses `clarify` to ask before the switch
2. **Watchdog self-prompt** — When running locally (no agent), the watchdog **waits 15 seconds** after detecting the switch, giving you time to type the toggle command

---

## mmproj Prompt Modes

### Mode 1: Agent Prompt (cloud model available)
When this skill is loaded and the agent detects you're about to switch to a local provider with mmproj support:

1. Agent uses `clarify` to ask: *"Would you like to load the vision mmproj for [provider]? This uses ~1-2GB more VRAM."*
2. Set the flag **before** switching:
   - **Yes:** `wsl touch /tmp/<provider>_mmproj`
   - **Custom path:** `wsl sh -c 'echo "/path/to/mmproj.gguf" > /tmp/<provider>_mmproj'`
   - **No:** `wsl rm -f /tmp/<provider>_mmproj`
3. Switch providers normally

### Mode 2: Watchdog Self-Prompt (local-only, no agent)
When you switch providers and no cloud agent is available to prompt you:

1. You switch to a local model that has `mmproj` configured
2. Watchdog detects the switch and **waits 15 seconds** before starting the engine
3. During those 15 seconds, it prints instructions to `journalctl` and polls for the flag:
   ```
   [MMPROJ] Model [provider] has vision mmproj available.
   [MMPROJ] Run: wsl touch /tmp/[provider]_mmproj  (within 15s)
   [MMPROJ] Waiting 15s for your decision...
   ```
4. **If you type the command** in time → engine starts WITH mmproj
5. **If timeout** → engine starts WITHOUT mmproj (you can restart later with mmproj if desired)

**15s window tip:** Keep a terminal open to WSL and type the toggle right after you switch models.

---

## Behavior

- **Strict exclusivity** — Only one engine service is ever active
- **Dynamic mmproj** — `/tmp/<provider>_mmproj` flag + EnvironmentFile for ANY model
- **Dual prompt** — Agent asks (cloud) OR watchdog waits 15s (local)
- **Auto-starts/stops** on provider switch (agent.log tail)
- **Auto-stops on app close** (Hermes.exe tasklist check)
- **Multi-session safe** — checks state.db, won't unload if gateway needs it
- **Idle timeout (15 min)** — safety-net fallback
- **Does NOT manage TTS/STT** — independent services

## Architecture

```
  Hermes agent.log ──► "Model switched in-place: ... (X) -> ... (llama-gemma-4)"
                             │
                        every 5s tail
                             ▼
  local-engine-watchdog (systemd on WSL)
     ├─ agent.log tail — THE trigger (ANY provider name)
     ├─ wait_for_mmproj_decision() — 15s prompt window if mmproj available
     │    ├─ flag appears → writes ~/scripts/mmproj_env.conf with --mmproj
     │    └─ timeout     → clears env file (no vision, -1~2GB VRAM)
     ├─ ARBITER — stop every other running engine first
     ├─ tasklist.exe — app close → stop all
     ├─ state.db — multi-session guard
     └─ log mtime — idle timeout (15min)
             │
             ▼
  Single systemd service with $MMPROJ_ARGS via EnvironmentFile
  └── Same ExecStart, arg injected dynamically
```

## Components

### 1. Engine config: `~/scripts/local_engines.json`

```json
{
  "llama-qwen": {
    "engine": "llama.cpp",
    "service": "llama-server.service",
    "mmproj": "/home/vibrationall/ai/llama.cpp/models/gemma-4/mmproj-Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-BF16.gguf",
    "log": "/home/vibrationall/ai/llama.cpp/server.log"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `engine` | Yes | Human-readable engine name |
| `service` | Yes | systemd service unit name (with `.service`) |
| `mmproj` | No | Default mmproj GGUF path for this model |
| `log` | No | Server log path for idle timeout |

### 2. Systemd service with EnvironmentFile

`~/.config/systemd/user/llama-server.service`:
```ini
[Service]
EnvironmentFile=-/home/vibrationall/scripts/mmproj_env.conf
ExecStart=/path/to/llama-server \
  $MMPROJ_ARGS \
  ...other args...
```

### 3. Watchdog script: `~/scripts/local_engine_watchdog.py`

The core mmproj decision loop:
```python
def wait_for_mmproj_decision(provider_name, cfg, timeout=15):
    """If model has mmproj available but no flag, wait briefly for user."""
    if not cfg.get("mmproj"):
        return None
    if os.path.exists(f"/tmp/{provider_name}_mmproj"):
        return get_mmproj_path(provider_name, cfg)
    print(f"[MMPROJ] Waiting {timeout}s for your decision...", flush=True)
    for i in range(timeout):
        time.sleep(1)
        if os.path.exists(f"/tmp/{provider_name}_mmproj"):
            return get_mmproj_path(provider_name, cfg)
    return None  # timeout, start without mmproj
```

### 4. mmproj toggle script: `~/scripts/toggle_mmproj.sh`

```bash
./toggle_mmproj.sh <provider> on                                       # enable with default path
./toggle_mmproj.sh <provider> off                                      # disable
./toggle_mmproj.sh <provider> /custom/mmproj/MyModel.gguf              # enable with custom path
./toggle_mmproj.sh <provider> toggle                                   # flip state
./toggle_mmproj.sh <provider> status                                   # check
```

## Quick Reference

```bash
# Check watchdog logs
wsl journalctl --user -u llama-watchdog.service -n 30 --no-pager

# Enable mmproj before switching (agent tells you, or do it yourself)
wsl touch /tmp/llama-gemma-4_mmproj

# Enable with custom path
wsl sh -c 'echo "/home/vibrationall/ai/models/mmproj-MyModel.gguf" > /tmp/llama-gemma-4_mmproj'

# Disable
wsl rm -f /tmp/llama-gemma-4_mmproj

# Check flag status
wsl [ -f /tmp/llama-gemma-4_mmproj ] && echo ON || echo OFF

# Restart engine WITH mmproj after starting without it:
wsl touch /tmp/llama-gemma-4_mmproj && wsl systemctl --user restart llama-server.service
```

## Pitfalls

- **state.db does NOT track `/model` switches mid-TUI-session** — use agent.log tailing
- **SQLite over /mnt/c/ 9p** — file locking broken; copy DB to /tmp before querying
- **Cold load time** — 17GB GGUF ~36s; set Hermes gateway_timeout >180s
- **LD_LIBRARY_PATH is critical** — must be set in each service unit
- **exit.target must be masked** — `systemctl --user mask exit.target --now`
- **Hermes has NO `on_provider_change` plugin hook** — workaround: tail agent.log
- **Auto-detection is case-insensitive** — any provider name with "llama", "gemma", "qwen", "heretic", "local", or "wsl"
- **mmproj flag file is ephemeral** — lives in WSL /tmp; cleared on WSL restart. Re-set after reboot.
- **15s window starts when watchdog detects the switch** — type `wsl touch /tmp/...` immediately
- **$MMPROJ_ARGS must be unquoted in ExecStart** — systemd expands the variable; double-quoting makes it a single arg
- **To add mmproj after starting without:** touch the flag and restart the engine service
