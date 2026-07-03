---
name: wsl-local-engine-watchdog
description: "WSL2 local engine watchdog — STRICT EXCLUSIVITY arbiter. mmproj prompt IN Hermes desktop chat via clarfy. Works with ANY model (local or cloud). Infinite wait, no timeout."
version: 9.0.0
author: Hermes Agent
tags: [wsl, llama.cpp, vllm, watchdog, vram, provider-switch, local-engine, systemd, arbiter, exclusivity, mmproj, vision, multimodal]
metadata:
  hermes:
    related_skills: [llama-cpp-wsl-ops]
---

# WSL Local Engine Watchdog (Strict Exclusivity + Agent mmproj Prompt)

A Python watchdog for WSL2 that **auto-starts and auto-stops local inference engines** based on which Hermes provider you select. **Enforces strict exclusivity — only one engine service is ever active at a time.**

**Dynamic mmproj (vision):** When a local model supports multimodal, the watchdog **waits indefinitely** for a `/tmp/<provider>_mmproj` flag file. The flag is set by the Hermes agent (inside the desktop chat) **or** manually by the user.

---

## How the mmproj Prompt Works

The prompt appears **IN the Hermes desktop app chat** — not as a Windows popup or desktop file.

### Primary Path: Agent Asks via clarfy (works with ANY model)

When the skill is loaded and a switch to a local provider with mmproj is detected:

1. **The agent IMMEDIATELY uses the `clarify` tool** to ask in the chat:
   > "Load vision mmproj for [provider]? This uses ~1-2GB more VRAM."
   > Choices: ["Yes", "No, skip (remember)"]

2. **Based on the answer:**
   - **Yes** → Agent runs: `wsl touch /tmp/<provider>_mmproj`
   - **No** → Agent runs: `wsl sh -c 'echo SKIP > /tmp/<provider>_mmproj'`

3. The watchdog is waiting **indefinitely** (no timeout) — it sees the flag within 1 second and starts the engine.

**This works with any model** — `clarify` is a built-in Hermes tool, not a model capability. Local models can use it too.

### Fallback: Manual via WSL Terminal

If no agent is active, the watchdog prints instructions to `journalctl`. You can set the flag directly:

```bash
wsl touch /tmp/llama-gemma-4_mmproj                              # enable mmproj
wsl sh -c 'echo "/custom/path/mmproj.gguf" > /tmp/llama-gemma-4_mmproj  # custom path
wsl sh -c 'echo SKIP > /tmp/llama-gemma-4_mmproj                 # skip + save preference
```

### Skip Memory
Once you skip a provider, `~/scripts/mmproj-skip/<provider>` is created. Next time:
- Agent skips the question silently
- Watchdog starts without mmproj immediately

Reset with: `wsl rm ~/scripts/mmproj-skip/llama-gemma-4`

---

## Behavior

- **Prompt via clarfy** — appears IN Hermes desktop chat, works with ANY model
- **Infinite wait** — watchdog waits forever for the flag (no 15s timeout)
- **Skip memory** — never re-asks for providers you've declined
- **Strict exclusivity** — only one engine active at a time
- **Dynamic mmproj** via `EnvironmentFile` + `$MMPROJ_ARGS`
- **Multi-session safe** — checks state.db before unloading
- **Idle timeout (15 min)** — safety-net fallback
- **Does NOT manage TTS/STT**

## Architecture

```
            Provider switch in Hermes desktop TUI
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   Hermes agent.log      Agent (in desktop chat)
   (watchdog sees it)    loads this skill
          │                     │
          ▼                     ▼
   Watchdog waits ─── Agent uses clarfy to ask user
   for flag          │          │
   (no timeout)      ▼          ▼
                    "Yes"     "No"
                     │          │
                     ▼          ▼
              wsl touch     echo SKIP >
              ..._mmproj    ..._mmproj
                     │          │
                     └────┬─────┘
                          ▼
                  Watchdog sees flag
                  → writes $MMPROJ_ARGS → starts engine
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

### 2. Systemd service with dynamic mmproj

```ini
[Service]
EnvironmentFile=-/home/vibrationall/scripts/mmproj_env.conf
ExecStart=/path/to/llama-server $MMPROJ_ARGS ...
```

### 3. Watchdog key logic (infinite wait)

```python
def wait_for_mmproj_decision(provider, cfg):
    if has_user_skipped(provider):
        return None                     # skip memory
    path = get_mmproj_path(provider, cfg)
    if path:
        return path                     # already decided
    if not cfg.get("mmproj"):
        return None                     # no mmproj available
    print(f"waiting for /tmp/{provider}_mmproj ...")
    while True:                         # NO TIMEOUT
        time.sleep(1)
        if flag exists:
            # extract path or SKIP decision...
```

## Quick Reference

```bash
# Agent asks in chat; or set manually:
wsl touch /tmp/llama-gemma-4_mmproj                          # enable
wsl sh -c 'echo "/path/mmproj.gguf" > /tmp/llama-gemma-4_mmproj  # custom
wsl sh -c 'echo SKIP > /tmp/llama-gemma-4_mmproj             # skip + save

# Reset skip memory
wsl rm ~/scripts/mmproj-skip/llama-gemma-4

# Watchdog logs
wsl journalctl --user -u llama-watchdog.service -n 20 --no-pager

# Toggle script
wsl ~/scripts/toggle_mmproj.sh llama-gemma-4 status
wsl ~/scripts/toggle_mmproj.sh llama-gemma-4 on
wsl ~/scripts/toggle_mmproj.sh llama-gemma-4 off
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
- **Agent MUST ask before the watchdog starts** — the skill instruction enforces this
