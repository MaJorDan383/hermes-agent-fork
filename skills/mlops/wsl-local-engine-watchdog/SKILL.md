---
name: wsl-local-engine-watchdog
description: "WSL2 local engine watchdog — STRICT EXCLUSIVITY arbiter. mmproj prompt IN Hermes desktop chat via clarfy. ASKED EVERY TIME a local model is loaded. No timeout, no skip memory."
version: 10.0.0
author: Hermes Agent
tags: [wsl, llama.cpp, vllm, watchdog, vram, provider-switch, local-engine, systemd, arbiter, exclusivity, mmproj, vision, multimodal]
metadata:
  hermes:
    related_skills: [llama-cpp-wsl-ops]
---

# WSL Local Engine Watchdog (Strict Exclusivity + Agent mmproj Prompt)

A Python watchdog for WSL2 that **auto-starts and auto-stops local inference engines** based on which Hermes provider you select. **Enforces strict exclusivity — only one engine service is ever active at a time.**

**Dynamic mmproj (vision):** When a local model supports multimodal, the watchdog **waits indefinitely** for a `/tmp/<provider>_mmproj` flag file. The flag is set by the Hermes agent (inside the desktop chat) **every single time** — no skip memory, no exceptions.

---

## How the mmproj Prompt Works

### EVERY TIME you switch to a local model with mmproj:

1. **Agent uses `clarify`** immediately in the Hermes desktop chat:
   > "Load vision mmproj for [provider]? Uses ~1-2GB more VRAM."
   > [Yes] [No]

2. **Based on your answer:**
   - **Yes** → Agent runs: `wsl touch /tmp/<provider>_mmproj`
   - **No** → Agent does nothing (flag stays absent)

3. **Watchdog** is waiting indefinitely. It sees the flag (or not) within 1s:
   - Flag exists → starts engine WITH mmproj
   - No flag → starts engine WITHOUT mmproj

**Next time you load the same model: the agent asks again.** Always.

---

## Architecture

```
Provider switch in Hermes desktop
        │
   ┌────┴────┐
   │         │
   ▼         ▼
watchdog   agent loads skill
tails      → USES CLARIFY IN CHAT
agent.log    "Load mmproj? [Yes] [No]"
every 5s          │       │
   │              │       │
   │              ▼       ▼
   │         agent sets  agent does
   │         flag touch  nothing
   │              │       │
   └──────────┬───┘       │
              │           │
              ▼           ▼
      engine starts  engine starts
      WITH mmproj    WITHOUT mmproj
                      (asked again
                       next time)
```

## Behavior

- **Agent asks EVERY TIME** — no skip memory, no exceptions
- **Prompt appears IN the Hermes desktop chat** via `clarify`
- **Works with ANY model** — `clarify` is a built-in tool
- **Infinite wait** — watchdog waits forever for the flag (no timeout)
- **No skip memory** — every switch = fresh question
- **Strict exclusivity** — only one engine active at a time
- **Dynamic mmproj** via `EnvironmentFile` + `$MMPROJ_ARGS`
- **Multi-session safe** — checks state.db before unloading

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

### 2. Systemd service

```ini
[Service]
EnvironmentFile=-/home/vibrationall/scripts/mmproj_env.conf
ExecStart=/path/to/llama-server $MMPROJ_ARGS ...
```

### 3. Watchdog key logic

```python
def wait_for_mmproj_decision(provider, cfg):
    """Wait indefinitely. ASKED EVERY TIME — no skip memory."""
    if not cfg.get("mmproj"):
        return None
    path = get_mmproj_path(provider, cfg)
    if path:
        return path
    print(f"waiting for /tmp/{provider}_mmproj ...")
    while True:
        time.sleep(1)
        if os.path.exists(f"/tmp/{provider}_mmproj"):
            return get_mmproj_path(provider, cfg) or cfg["mmproj"]
```

## Quick Reference

```bash
# Agent asks in chat every time. Manual override:
wsl touch /tmp/llama-gemma-4_mmproj
wsl sh -c 'echo "/path/mmproj.gguf" > /tmp/llama-gemma-4_mmproj'

# Watchdog logs
wsl journalctl --user -u llama-watchdog.service -n 20 --no-pager
```

## Pitfalls

- **state.db does NOT track mid-session switches** — use agent.log tailing
- **SQLite over /mnt/c/ 9p** — copy DB to /tmp before querying
- **Cold load time** — 17GB GGUF ~36s; set gateway_timeout >180s
- **LD_LIBRARY_PATH** — must be explicit in each service unit
- **exit.target must be masked** — `systemctl --user mask exit.target --now`
- **No `on_provider_change` hook** — workaround: tail agent.log
- **clarfy works with ANY model** — it's a tool, not a model capability
- **Flag is ephemeral** — lives in /tmp; clears on WSL restart
- **Agent asks EVERY TIME** — no skip memory, no exceptions
