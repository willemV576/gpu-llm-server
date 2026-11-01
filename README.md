# GPU LLM Server

Local LLM box running Ollama, serving models to other machines on the LAN.

> **Recovery note (2026-07-07):** gpu-llm has **no automated backup**. The reproducible config artifacts that otherwise live only on the box are mirrored under [`config/`](config/) in this repo — the Ollama systemd drop-in ([`config/ollama/override.conf`](config/ollama/override.conf)) and the voice-pipeline compose ([`config/voice-pipeline/docker-compose.yml`](config/voice-pipeline/docker-compose.yml)). Firewall rules are documented in the hsc-project `network.md §8`. The netplan WiFi PSK is deliberately **not** committed — it belongs in Bitwarden. Until a real backup exists, this repo is the primary rebuild source. Still on-box-only and needing sudo to capture: `/etc/iptables/rules.v4`, `/etc/netplan/*.yaml`.

## Hardware (assembled 2026-04-08)

| Component | Model |
|---|---|
| GPU | PNY GeForce RTX 5060 Ti Dual OC 16GB GDDR7 |
| CPU | AMD Ryzen 5 5600 |
| Motherboard | MSI B550M PRO-VDH WIFI (mATX, AM4) |
| RAM | Corsair Vengeance LPX 32GB (4x8GB) DDR4-3200 CL16 |
| SSD | Crucial E100 480GB PCIe Gen4 NVMe M.2 |
| Case | FSP CUT593P ATX Mid Tower |
| PSU | Cooler Master MWE 750W Gold V2 Modular White (ATX 3.0) |
| Total | ~AUD $1,713 |

**First boot 2026-04-08:** CPU 41°C idle, board 33°C, all 32GB RAM (4x8GB, all 4 DIMM slots populated), SSD detected, GPU + all 4 case fans spinning.

## OS Install

- **OS:** Ubuntu Server 24.04 LTS
- **Disk:** entire 480GB SSD, default LVM layout
- **Network:** DHCP on enp42s0 (got 192.168.1.200 on first boot). **Primary/active address is Ethernet `192.168.1.200`** — this is the IP the Beelink HA integrations (Ollama `:11434`, Wyoming `:10200/10300/10400`) and the firewall rules point at; WiFi `192.168.1.102` (below) is secondary. ⚠️ Neither is a DHCP *reservation* (see hsc-project `network.md §3`) — a rebuilt box may not get `.200` back; reserve it on the AX53/FastMile first or the HA integrations break.
- **No GUI at install time** — added manually post-install

## Post-Install Status (2026-04-08)

### Done
- **Ubuntu Server 24.04.4 LTS** installed, full disk, no LUKS
- **Hostname:** `gpu-llm`
- **User:** `adminwill` (sudo, SSH key auth via GitHub import)
- **SSH:** OpenSSH server enabled, key-based auth working from M3 Mac
- **System updated:** `apt update && apt upgrade -y` clean
- **NVIDIA driver:** `nvidia-driver-580-open` (recommended), driver 580.159.03, CUDA 13.0
  - `nvidia-smi` confirms RTX 5060 Ti, 16311 MiB VRAM, ~31°C idle, 5W
  - **Upgraded 580.126.09 → 580.159.03 on 2026-05-27** (apt, same 580-open branch, CUDA still 13.0). All tuning settings verified intact post-upgrade. One cosmetic boot artifact: `nvidia-persistenced` fails its first start attempt (device nodes `/dev/nvidia*` not yet created) and systemd auto-restarts it ~2 s later — self-healing, no action needed.
- **GUI:** XFCE4 + lightdm installed, boots to text mode by default
  - Default session set to xfce via `/etc/lightdm/lightdm.conf.d/50-xfce.conf` (`user-session=xfce`)
  - Launch on demand: `sudo systemctl isolate graphical.target`
  - Back to text: `sudo systemctl isolate multi-user.target`
  - Default boot target: `multi-user.target` (locked in via `systemctl set-default`)
- **WiFi:** Intel AX200 (`wlo1`) on TP-Link_5G via netplan
  - Config: `/etc/netplan/60-wifi.yaml` (chmod 600), single quotes for SSID/password
  - Connected at WiFi 6 / 5 GHz / 80 MHz / 1.2 Gbit/s, signal -22 dBm, IP `192.168.1.102`
  - `systemd-networkd-wait-online.service` **disabled** so boot doesn't hang without ethernet
- **SSH config on M3:** `~/.ssh/config` host `gpu-llm` → `192.168.1.102` (wifi IP, no port, key auth)
- **VS Code Remote-SSH:** working from M3, host alias `gpu-llm`
- **Firewall (UFW):** active, default deny incoming / allow outgoing
  - Allow `22/tcp` (SSH) from `192.168.1.0/24`
  - Allow `11434/tcp` (Ollama) from `192.168.1.0/24`
- **Disk (LVM):** root LV extended to use full SSD
  - `ubuntu-vg/ubuntu-lv` grown from 100 GB → 444 GB (`lvextend -l +100%FREE` + `resize2fs`)
  - `/` now 437 GB usable
- **GitHub repo published:** [willemV576/gpu-llm-server](https://github.com/willemV576/gpu-llm-server)
  - `gh` CLI installed and authenticated as `willemV576` (SSH key uploaded as `gpu-llm`)
- **Performance tuning:**
  - NVIDIA persistence daemon (`nvidia-persistenced`) hooked into `multi-user.target` via `systemctl add-wants` (2026-04-11) — unit is marked `static` (no `[Install]` section), so plain `systemctl enable` won't work; `add-wants` is the supported workaround.
    - **Gotcha**: Ubuntu ships the daemon with the `--no-persistence-mode` flag, which is the *modern* mechanism — the daemon holds a device reference to keep the driver resident. `nvidia-smi --query-gpu=persistence_mode` will still report `Disabled` because that reads the legacy flag, which is intentionally unused. Practical effect is identical to the legacy `nvidia-smi -pm 1` approach.
  - CPU governor set to `performance` via `cpufrequtils` (`/etc/default/cpufrequtils`, persistent)
- **Ollama installed and operational (2026-04-11):**
  - Installed via the official script (`curl -fsSL https://ollama.com/install.sh | sh`) — binary at `/usr/local/bin/ollama`, systemd unit auto-generated
  - Ollama running as systemd service, auto-start enabled. Originally v0.20.4; **upgraded to v0.40.13 on 2026-11-01** by re-running the install script (the `override.conf` drop-in survives). v0.40 converts each model to its llama.cpp format on first run (slow first load) and keeps the original as a backup — see the v0.40.2 release notes for the cleanup command.
  - `OLLAMA_NUM_THREADS=6` set via `/etc/systemd/system/ollama.service.d/override.conf` (physical cores only, no oversubscribe)
  - GPU detected: `NVIDIA GeForce RTX 5060 Ti`, CUDA 13.0, 15.9 GiB VRAM, `default_num_ctx=4096`
  - Three models pulled and smoke-tested — all responding correctly on GPU:
    - `llama3.2:3b-instruct-q8_0` (3.4 GB) — original voice/HA intent model; **replaced by `qwen3:8b` on 2026-11-01**, kept as fallback
    - `qwen3:8b` (5.2 GB, ~5.9 GB VRAM loaded) — **current voice/HA model** (added 2026-11-01). Mini HA tool-calling test: 12/12 correct at ~0.5 s vs 10/12 for llama3.2. HA Ollama integration: Think OFF, Keep alive `-1` (always loaded), Control Home Assistant = Assist, max history 5.
    - `qwen2.5:14b-instruct-q6_K` (12 GB) — general-purpose / research / Linux / HA design specialist
    - `qwen2.5-coder:14b-instruct-q6_K` (12 GB) — coding specialist
    - `deepseek-r1:14b` — reasoning / R1-distill specialist (pulled after the original three; present on box as of 2026-07-07)
  - **Verified performance on `qwen2.5:14b-instruct-q6_K` (2026-04-11)**:
    - Generation rate: **33.86 tok/s** (in expected band for 14B q6 on RTX 5060 Ti — between 4060 Ti and 4070)
    - Prompt eval rate: **897.78 tok/s**
    - Cold load from NVMe: **3.94 s** for the 12 GB model
    - Warm reinvocation: ~3.4 s wall-clock for short responses (real first-token latency ~1 s + CLI overhead)
- **LAN exposure + runtime env vars applied (2026-04-18):** Ollama listens on `0.0.0.0:11434`, reachable from Beelink HA at `http://192.168.1.200:11434`. Full env set in `/etc/systemd/system/ollama.service.d/override.conf`: `OLLAMA_NUM_THREADS=6`, `OLLAMA_HOST=0.0.0.0:11434`, `OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_KV_CACHE_TYPE=q8_0`, `OLLAMA_KEEP_ALIVE=30m`, `OLLAMA_NUM_PARALLEL=2`, `OLLAMA_MAX_LOADED_MODELS=2`. Note: hostname `gpu-llm` does not resolve from Beelink — use the IP.
- **Docker + NVIDIA Container Toolkit (recorded 2026-07-07):** prerequisite for the Wyoming GPU containers.
  - Docker CE from the official apt repo — keyring `/etc/apt/keyrings/docker.asc`, source `https://download.docker.com/linux/ubuntu noble stable`; packages `docker-ce docker-ce-cli containerd.io docker-ce-rootless-extras` (29.5.3 as of 2026-07-07).
  - NVIDIA Container Toolkit from `https://nvidia.github.io/libnvidia-container/stable/deb` — keyring `/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg`; packages `nvidia-container-toolkit nvidia-container-toolkit-base libnvidia-container-tools libnvidia-container1` (1.19.1).
  - Wired into Docker via `sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker` — confirmed by the `"nvidia"` runtime (`nvidia-container-runtime`) entry in `/etc/docker/daemon.json`. This is what makes the compose `deploy.resources.reservations.devices` GPU block resolve.
- **Full voice pipeline operational (verified 2026-05-27):** all three Wyoming services run as Docker containers and auto-start on boot — previously listed as "planned" under Migration/Integration, now live. **Full working compose (GPU passthrough, volumes, all env) is committed at [`config/voice-pipeline/docker-compose.yml`](config/voice-pipeline/docker-compose.yml)** — the live file exists only at `/opt/voice-pipeline/` on the box, so the repo copy is the recovery source:
  - `wyoming-whisper` (STT) — `lscr.io/linuxserver/faster-whisper:gpu`, port **10300**, on CUDA, model **`turbo`** (large-v3-turbo, downloaded from Hugging Face into `whisper-data/` on first start; ~2.2 GB VRAM), `WHISPER_LANG=en`, beam-size 1. Switched from `small-int8` (~480 MiB) on 2026-11-01 after it misheard "dim it" as "damn it".
  - `wyoming-piper` (TTS) — `rhasspy/wyoming-piper`, port **10200**, voice `en_US-lessac-medium`.
  - `wyoming-openwakeword` (wake word) — `rhasspy/wyoming-openwakeword`, port **10400**, preloaded model `ok_nabu`.
  - **Security (updated 2026-07-07 — supersedes the old "not firewalled" warning):** the containers still bind `0.0.0.0`, but inbound access to `10200/10300/10400` is locked down at the kernel by a persisted `DOCKER-USER` rule that DROPs those ports on `enp42s0` for everything except the Beelink `192.168.1.194` (see hsc-project `network.md §8`; persisted via `iptables-persistent` in `/etc/iptables/rules.v4`). Ollama `:11434` is likewise UFW-restricted to `192.168.1.194`. Neither Ollama nor the Wyoming services have app-level auth, so the firewall scoping is what protects them — keep any future `tailscale0` rules per-port, never a blanket allow.

### TODO

#### 1. Ollama
- ~~Install Ollama~~ (done 2026-04-11)
- ~~Set up as systemd service~~ (done — v0.20.4 with `OLLAMA_NUM_THREADS=6` override; now v0.40.13)
- ~~Pull initial models~~ (done — three-model specialist-swap lineup, see "Ollama installed" block above)
- ~~Configure to listen on LAN~~ (done 2026-04-18 — `OLLAMA_HOST=0.0.0.0:11434`, see "LAN exposure" block above)
- ~~Set recommended runtime env vars~~ (done 2026-04-18 — all 7 vars in `override.conf`, see "LAN exposure" block above)
  - `OLLAMA_FLASH_ATTENTION=1` — faster, less VRAM
  - `OLLAMA_KV_CACHE_TYPE=q8_0` — quantize KV cache, fits more context
  - `OLLAMA_KEEP_ALIVE=30m` — keep model loaded between requests
  - `OLLAMA_NUM_PARALLEL=2` — concurrent requests
  - `OLLAMA_MAX_LOADED_MODELS=2` — keep the 3B voice model always-on alongside one 14B specialist

#### BIOS check (next reboot)
- Verify **Resizable BAR / Above 4G Decoding** is enabled in B550M BIOS — meaningful perf gain for the Blackwell 5060 Ti

#### 2. Monitoring
- nvtop / nvidia-smi for GPU
- Optional: Prometheus node_exporter + nvidia-gpu-exporter for HA dashboard integration

#### 3. CUDA toolkit (only if needed)
- Driver 580 + CUDA 13 already exposed via `nvidia-smi` runtime
- Install full toolkit only if a model tool requires `nvcc`

#### 4. PSU cable cleanup
- 24-pin currently has unnecessary extension stacked on PSU cable — too long
- Remove extension, plug PSU cable directly into board
- Same to verify on EPS / PCIe cables

## Migration / Integration

**End state: gpu-llm + Beelink S12 Pro (HAOS) together replace geekom-a6.**

- **gpu-llm** takes: Ollama, voice pipeline (whisper/piper/openwakeword), any GPU/heavy compute
- **Beelink S12 Pro** takes: Home Assistant OS bare metal (currently HA Core in Docker on a6)
- **geekom-a6** then becomes: work workstation (Ubuntu desktop, ERP/dev work)
- Replaces M1 Mac voice backend (sold)
