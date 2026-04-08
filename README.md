# GPU LLM Server

Local LLM box running Ollama, serving models to other machines on the LAN.

## Hardware (assembled 2026-04-08)

| Component | Model |
|---|---|
| GPU | PNY GeForce RTX 5060 Ti Dual OC 16GB GDDR7 |
| CPU | AMD Ryzen 5 5600 |
| Motherboard | MSI B550M PRO-VDH WIFI (mATX, AM4) |
| RAM | Corsair Vengeance LPX 16GB (2x8GB) DDR4-3200 CL16 |
| SSD | Crucial E100 480GB PCIe Gen4 NVMe M.2 |
| Case | FSP CUT593P ATX Mid Tower |
| PSU | Cooler Master MWE 750W Gold V2 Modular White (ATX 3.0) |
| Total | ~AUD $1,713 |

**First boot 2026-04-08:** CPU 41°C idle, board 33°C, all 16GB RAM dual-channel, SSD detected, GPU + all 4 case fans spinning.

## OS Install

- **OS:** Ubuntu Server 24.04 LTS
- **Disk:** entire 480GB SSD, default LVM layout
- **Network:** DHCP on enp42s0 (got 192.168.1.200 on first boot)
- **No GUI at install time** — added manually post-install

## Post-Install Status (2026-04-08)

### Done
- **Ubuntu Server 24.04.4 LTS** installed, full disk, no LUKS
- **Hostname:** `gpu-llm`
- **User:** `adminwill` (sudo, SSH key auth via GitHub import)
- **SSH:** OpenSSH server enabled, key-based auth working from M3 Mac
- **System updated:** `apt update && apt upgrade -y` clean
- **NVIDIA driver:** `nvidia-driver-580-open` (recommended), driver 580.126.09, CUDA 13.0
  - `nvidia-smi` confirms RTX 5060 Ti, 16311 MiB VRAM, 29°C idle, 5W
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

### TODO

#### 1. Ollama
- Install Ollama
- Configure to listen on LAN (`OLLAMA_HOST=0.0.0.0:11434`)
- Pull initial models (TBD — depends on 16GB VRAM budget)
- Set up as systemd service

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
