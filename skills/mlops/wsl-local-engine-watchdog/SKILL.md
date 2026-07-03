---
name: wsl-local-engine-watchdog
description: "WSL2 local engine watchdog — STRICT EXCLUSIVITY arbiter: only one local LLM engine runs at a time (llama.cpp, vLLM, etc.) with mmproj (vision) toggle support"
version: 6.0.0
author: Hermes Agent
tags: [wsl, llama.cpp, vllm, watchdog, vram, provider-switch, local-engine, systemd, arbiter, exclusivity, mmproj, vision, multimodal]
metadata:
  hermes:
    related_skills: [llama-cpp-wsl-ops]
---

# WSL Local Engine Watchdog (Strict Exclusivity + mmproj Toggle)

A Python watchdog for WSL2 that **auto-starts and auto-stops local inference engines** based on which Hermes provider you select. **Enforces strict exclusivity — only one engine service is ever active at a time** to guarantee VRAM safety.

**mmproj (vision) toggle:** When a local model supports multimodal, the watchdog checks a flag file at `/tmp/<provider>_mmproj` to decide whether to load the vision mmproj. If present, it uses the multimodal service variant. If absent, it uses the standard (VRAM-efficient) service.

## Behavior

- **Strict exclusivity** — Only one engine service is ever active at a time
- **mmproj toggle** — `/tmp/<provider>_mmproj` flag file controls whether vision mmproj is loaded on next start
- **Auto-starts/stops** on provider switch (agent.log tail)
- **Auto-stops on app close** (Hermes.exe tasklist check)
- **Multi-session safe** — checks state.db, won't stop if another session needs it
- **Idle timeout (15 min)** — safety-net fallback
- **Does NOT manage TTS/STT** — those are independent

## Architecture

```
Session: /model provider switch
        │
        ▼
  Hermes agent.log ──► "Model switched in-place: ... (X) -> ... (llama-qwen)"
                             │
                        every 5s tail
                             │
                             ▼
  local-engine-watchdog (systemd on WSL) — ARBITER + MMPROJ AWARE
     ├─ agent.log tail — THE trigger
     ├─ local_engines.json — engine configs (service, service_mmproj, mmproj, log)
     ├─ /tmp/<provider>_mmproj flag — if exists → use service_mmproj variant
     ├─ ARBITER — stop every other running engine before starting
     ├─ tasklist.exe — app close → stop all
     ├─ state.db — multi-session guard
     └─ log mtime — idle timeout
             │
      start / stop
             ▼
  llama-server.service  (no mmproj, -1~2GB VRAM)
  ─ OR ─  (mutually exclusive)
  llama-server-multimodal.service  (with mmproj vision, +1~2GB VRAM)
```

## Components

### 1. Engine config: `~/scripts/local_engines.json`

```json
{
  "llama-qwen": {
    "engine": "llama.cpp",
    "service": "llama-server.service",
    "service_mmproj": "llama-server-multimodal.service",
    "mmproj": "/home/vibrationall/ai/llama.cpp/models/gemma-4/mmproj-Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-BF16.gguf",
    "log": "/home/vibrationall/ai/llama.cpp/server.log"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `engine` | Yes | Human-readable name |
| `service` | Yes | Standard systemd service (no mmproj) |
| `service_mmproj` | No | Multimodal systemd service (with --mmproj flag) |
| `mmproj` | No | Path to mmproj GGUF file (for documentation) |
| `log` | No | Server log path for idle timeout |

### 2. Script: `~/scripts/local_engine_watchdog.py`

Full script deployed on WSL. Core logic:

```python
def get_service_to_start(provider_name, cfg):
    """Pick mmproj-enabled service if flag exists and config supports it."""
    flag = f"/tmp/{provider_name}_mmproj"
    if cfg.get("service_mmproj") and os.path.exists(flag):
        return cfg["service_mmproj"]
    return cfg.get("service", "")

# ARBITER main loop
while True:
    time.sleep(5)
    event = parse_agent_log()
    action, old_p, new_p = event["action"], event["old"], event["new"]

    if action in ("start","switch"):
        _, new_cfg = get_engine(new_p)
        if new_cfg:
            target_svc = get_service_to_start(new_p, new_cfg)
            # Stop every OTHER engine service (both variants)
            for prov, cfg in engines.items():
                if prov != new_p:
                    for key in ("service","service_mmproj"):
                        svc = cfg.get(key)
                        if svc and service_running(svc):
                            stop_service(svc)
            # Kill target if alive, then start fresh
            if service_running(target_svc): stop_service(target_svc)
            time.sleep(1)
            start_service(target_svc)
            current_provider = new_p
        continue

    if action == "stop":
        _, old_cfg = get_engine(old_p)
        if old_cfg and not sessions_still_need_engine(old_p):
            # Stop BOTH service variants (safety sweep)
            for key in ("service","service_mmproj"):
                svc = old_cfg.get(key)
                if svc and service_running(svc): stop_service(svc)
            if old_p == current_provider: current_provider = None
        continue

    # App close → stop all
    if not hermes_app_running() and not any_session_uses_local_engine():
        stop_all_engines()
        current_provider = None
        continue

    # Idle timeout per engine
    for prov, cfg in engines.items():
        if engine_idle(cfg):
            for key in ("service","service_mmproj"):
                svc = cfg.get(key)
                if svc and service_running(svc):
                    if not sessions_still_need_engine(prov):
                        stop_service(svc)
```

Key functions:
- `get_engine(provider)` — resolves provider name to engine config (explicit config or auto-detect)
- `get_service_to_start(provider, cfg)` — checks `/tmp/<provider>_mmproj` flag to pick service variant
- `parse_agent_log()` — tails agent.log for `Model switched in-place` events
- `hermes_app_running()` — checks Windows tasklist for Hermes.exe
- `sessions_still_need_engine(provider)` — multi-session safety via state.db
- `service_running(svc)` / `start_service(svc)` / `stop_service(svc)` — systemd wrappers

### 3. Systemd services

**Standard** (`~/.config/systemd/user/llama-server.service`):
```ini
[Unit]
Description=llama.cpp Inference Server (no mmproj)
After=network.target

[Service]
Type=simple
Environment=HOME=/home/vibrationall
Environment=USER=vibrationall
Environment=PATH=/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=LD_LIBRARY_PATH=/home/vibrationall/ai/turboquant_llama/build/bin:/usr/local/cuda/lib64:/usr/lib/wsl/lib
ExecStart=/home/vibrationall/ai/turboquant_llama/build/bin/llama-server \
  -m /home/vibrationall/ai/llama.cpp/models/gemma-4/Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-Q4_K_M.gguf \
  --model-draft /home/vibrationall/ai/llama.cpp/models/gemma-4/mtp-gemma-4-26B-A4B-it.gguf \
  --spec-type draft-mtp --spec-draft-n-max 5 \
  --host 0.0.0.0 --port 8080 \
  -c 140000 -ctk turbo4 -ctv turbo4 -ngl 99 \
  --cont-batching -np 1 --cache-prompt --flash-attn auto
Restart=on-failure
RestartSec=10
TimeoutStartSec=300
TimeoutStopSec=30
StandardOutput=append:/home/vibrationall/ai/llama.cpp/server.log
StandardError=append:/home/vibrationall/ai/llama.cpp/server.log

[Install]
WantedBy=default.target
```

**Multimodal** (`~/.config/systemd/user/llama-server-multimodal.service`):
Same as standard but adds:
```
  --mmproj /home/vibrationall/ai/llama.cpp/models/gemma-4/mmproj-Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-BF16.gguf \
  --no-mmproj-offload \
```

### 4. mmproj toggle script: `~/scripts/toggle_mmproj.sh`

```bash
#!/usr/bin/env bash
PROVIDER="${1:?Usage: toggle_mmproj.sh <provider> [on|off|status]}"
ACTION="${2:-toggle}"
FLAG_FILE="/tmp/${PROVIDER}_mmproj"
case "$ACTION" in
  on)    touch "$FLAG_FILE"; echo "mmproj ENABLED for $PROVIDER" ;;
  off)   rm -f "$FLAG_FILE";  echo "mmproj DISABLED for $PROVIDER" ;;
  status) [ -f "$FLAG_FILE" ] && echo "ENABLED" || echo "DISABLED" ;;
  toggle) [ -f "$FLAG_FILE" ] && rm -f "$FLAG_FILE" || touch "$FLAG_FILE" ;;
esac
```

### 5. Watchdog systemd service: `~/.config/systemd/user/llama-watchdog.service`

```ini
[Unit]
Description=Local engine watchdog (STRICT EXCLUSIVITY arbiter for llama.cpp/vLLM/etc)
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/vibrationall/scripts/local_engine_watchdog.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

## Commands

```bash
# Watchdog status
systemctl --user status llama-watchdog.service --no-pager | head -5

# View logs
journalctl --user -u llama-watchdog.service -n 20 --no-pager

# Toggle vision mmproj
~/scripts/toggle_mmproj.sh llama-qwen on      # enable vision
~/scripts/toggle_mmproj.sh llama-qwen off     # disable vision (save VRAM)
~/scripts/toggle_mmproj.sh llama-qwen toggle  # flip
~/scripts/toggle_mmproj.sh llama-qwen status  # check

# Test switch TO a local engine
echo "Model switched in-place: cloud (opencode) -> local (llama-qwen)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log

# Test switch FROM a local engine
echo "Model switched in-place: local (llama-qwen) -> cloud (opencode)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log
```

## Pitfalls

- **state.db does NOT track `/model` switches mid-TUI-session** — use agent.log tailing
- **SQLite over /mnt/c/ 9p** — file locking broken; always copy DB to /tmp before querying
- **Cold load time** — 17GB GGUF ~36s; set Hermes gateway_timeout >180s
- **LD_LIBRARY_PATH is critical** — must be set in each systemd service unit
- **exit.target must be masked** — `systemctl --user mask exit.target --now`
- **Hermes has NO `on_provider_change` plugin hook** — workaround: tail agent.log
- **Auto-detection is case-insensitive** — "Llama-Pro", "Gemma-4", "vLLM-70B" all work
- **Engine config takes precedence** over auto-detection
- **mmproj flag file is ephemeral** — lives in WSL /tmp; cleared on WSL restart. Set it again after reboot.
- **Toggle script before switch** — set the mmproj flag BEFORE switching providers; the watchdog reads it on the next 5s cycle.
