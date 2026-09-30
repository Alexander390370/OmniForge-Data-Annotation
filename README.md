# OmniForge - Full-Modal Private Data Annotation Platform

A fully local, privacy-first data annotation platform with AI-assisted pre-annotation for text, image, audio, and video. Built for 20-30 person teams working in shifts (see Known Limitations for concurrency guidance).

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Python 3.10](https://img.shields.io/badge/Python-3.10-blue)
![Docker](https://img.shields.io/badge/Docker-Enabled-blue)

## Overview

OmniForge is a full-modal annotation system built for teams that cannot send data off-premises. Instead of relying on cloud APIs, it uses a "Label Studio frontend + local AI backend" microservice architecture, combining local model inference (Ollama, YOLO, Whisper) with a single consumer GPU (RTX 4060). Data stays local; AI handles pre-annotation; humans focus on refinement.

**Who this is for**: teams that already have Label Studio experience and want to add local AI pre-annotation without adopting a SaaS platform.

## Quick Start

Prerequisites: Windows 11 with WSL2, Docker Desktop, an NVIDIA GPU with recent drivers.

```text
1. Configure WSL2 mirrored networking in %USERPROFILE%\.wslconfig
2. Create the Conda environment: conda create -n ls_ml python=3.10 -y
3. Clone this repo and cd into it
4. Run start_all.bat  (starts Label Studio, detects LAN IP, opens browser)
5. Run switch_text.bat (or switch_image / switch_audio) to bring up one AI backend
```

> ⚠️ First launch will download the Label Studio Docker image and AI model weights — expect several minutes of waiting time.

The `*.bat` scripts are Windows convenience wrappers. Under the hood they invoke WSL shell scripts (`start_ollama.sh` etc.) — Linux users can call those directly.

## Project Structure

```text
OmniForge/
├── admin-ui/                # Admin dashboard frontend
├── user-ui/                 # Custom user frontend (in development — see Limitations)
├── my_ollama_backend/       # Text AI backend (port 9090)
├── my_yolo_backend/         # Image AI backend (port 9091)
│   └── inject_ai.py         # Headless API injection fallback
├── my_whisper_backend/      # Audio AI backend (port 9092)
├── video2frames.bat         # Video frame extraction (custom FPS)
├── frames2video.bat         # Frame-to-video recomposition
├── export_delivery.py       # One-click full-modal export tool
├── backup_data.bat          # Daily backup of ~/label_studio_data
├── start_all.bat            # One-click startup (cleanup + LS + IP detection)
├── switch_text.bat          # Exclusive VRAM switch - text mode
├── switch_image.bat         # Exclusive VRAM switch - image mode
├── switch_audio.bat         # Exclusive VRAM switch - audio mode
└── data/                    # Repo-local artifacts (see note below)
```

> **Note on storage locations**: `data/` inside this repo is for repo-local artifacts only. The critical annotation database and media files live at `~/label_studio_data` — a separate location that must be backed up.

## 1. Infrastructure & Network Architecture

- **Host OS**: Windows 11 + WSL2 (Ubuntu 22.04/24.04)
- **Runtime**: Docker Desktop (WSL2 backend), containerized Label Studio
- **Data mount**: `~/label_studio_data` (SQLite DB + media files — **back this up regularly**)
- **Network architecture**:
  - [DEPRECATED] Ngrok public mapping. Label Studio's strict CSP blocks blob assets over HTTPS tunnels, causing frequent white screens.
  - [CURRENT] WSL2 mirrored networking (`networkingMode=mirrored`). Configured in `%USERPROFILE%\.wslconfig`. WSL and Windows share the network interface — no more `netsh` port forwarding, no more IP drift across WSL restarts.
  - Team access: `http://<host-LAN-IP>:18080`
- **Firewall**: Inbound TCP 18080, 9090, 9091, 9092 allowed. Network profile set to "Private".

## 2. Full-Modal AI Pipelines

All three backends run in a Conda `ls_ml` (Python 3.10) environment, served via Gunicorn.

### 2.1 Text AI Pipeline (9090)

- **Model**: Ollama (Qwen 2.5 7B)
- **Template**: `<TextArea>`, field `$text`
- **Flow**: Receive task → build prompt → call local API → return text.

### 2.2 Image AI Pipeline (9091)

- **Model**: YOLOv8n
- **Key workaround**: Bypasses Label Studio's HTTP 401 image fetch by reading original images directly from disk. Uses `os.path.expanduser("~/label_studio_data/media...")` — this avoids timeouts that occur when the Docker container tries to download images over its own network.

### 2.3 Audio AI Pipeline (9092)

- **Model**: faster-whisper (loaded locally)
- **Flow**: VAD silence filtering → transcription → segment-level or time-range event annotation.
- **Note**: Model files are downloaded via `hf-mirror` or manually placed in `~/.cache/whisper` to avoid HuggingFace access issues.

### 2.4 Video Processing Pipeline

- `video2frames.bat` calls FFmpeg to extract frames → frames are annotated via the image pipeline → `frames2video.bat` recomposes for demo purposes.

## 3. Automation & DevOps Layer

- **`start_all.bat`**: Cleans zombie port processes → starts Label Studio (18080) → detects the real LAN IP (skips VPN/ZeroTier virtual adapters) → opens browser.
- **`switch_*.bat`**: Exclusive VRAM mode switching. Kills the other backends, brings up the target backend, runs a health check with 30s timeout.
- **`inject_ai.py`**: Fallback path when the frontend breaks. Obtains session cookie + CSRF token to bypass frontend limitations and push AI predictions through Label Studio's internal REST API. **Note**: this depends on internal Label Studio APIs and may need updating after Label Studio version upgrades.
- **`backup_data.bat`**: Timestamped `robocopy` of `~/label_studio_data` to a secondary disk, with rotation keeping the last N copies. Run it before ending each work session, or wire it into Task Scheduler.
- **Backend launchers**: `start_ollama.sh` / `start_yolo.sh` / `start_whisper.sh` inside WSL, loading the Conda environment and running Gunicorn.

## 4. Data Delivery & Closed Loop

- **`export_delivery.py`**: Drag a Label Studio JSON export onto the BAT, and it auto-detects the modality:
  - **Image**: YOLO TXT (normalized coords) + auto-copied original images
  - **Text**: `finetune_data.jsonl` (Alpaca format) + `text_classification.csv`
  - **Audio**: `audio_segments.csv` (with timestamps, labels) + `.jsonl`
- **Delivery format**: An `OmniForge_Export_<project-name>` folder, ready for client training pipelines.
- **Backup**: `backup_data.bat` for scheduled copies; also export JSON after each work session as a lightweight safety net.

## 5. Performance & Business Value

| Dimension | Traditional Team | OmniForge | Core Difference |
|-----------|-----------------|-----------|-----------------|
| Driver | Manual labeling | AI pre-annotation + human refinement | Fewer manual steps |
| Data Security | Cloud or third-party tools | Fully local deployment | Suitable for data-sensitive domains |
| Tech Stack | LabelImg/CVAT | LS + Docker + WSL2 + YOLO + Whisper | Full-modal coverage |
| Deployment | Requires a dedicated engineer | One-click scripts | Minimal setup |
| Delivery | Raw JSON | YOLO/JSONL/CSV export | Ready-to-train outputs |

**Throughput reference (single RTX 4060 8GB)**:

- Text: 3,000-5,000 entries/day (input < 2,000 chars)
- Image: 10,000-15,000 images/day (1080P)
- Audio: 15-25 hours/day (transcription)
- Video: 5-7 hours/day (at 1fps extraction)

*Numbers are from short-sample runs under clean GPU conditions. Real-world throughput will drop with complex samples and long prompts.*

In practice, AI pre-annotation typically delivers a 3-5x labeling throughput improvement over fully-manual workflows on suitable datasets.

## 6. Known Limitations

- **SQLite is the current database.** Label Studio recommends PostgreSQL for multi-user deployments. SQLite's write lock makes it a poor fit above ~10 concurrent annotators. Team size does not equal concurrent user count — 20-30 annotators working in shifts can stay under this limit, but a single team all submitting at once cannot. Migrating to PostgreSQL should be the first step before scaling past it.
- **Backend ports 9090-9092 have no authentication or access control.** Any host on the LAN can invoke the endpoints, including the image backend that reads files from disk. **Never expose these ports to the public internet.** Only deploy within a fully trusted internal network. Adding an auth layer or reverse proxy is on the roadmap.
- **Single-client AI workloads.** The RTX 4060 runs one modality backend at a time. Multi-user concurrent AI pre-annotation will queue or OOM.
- **`user-ui/` is in development.** The current interface is Label Studio's built-in frontend; the custom user console is not finished.
- **`inject_ai.py` is version-sensitive.** It relies on Label Studio's internal API shape. Pin your Label Studio version if you depend on it.

## 7. Known Pitfalls & Fixes

### 7.1 Network & Ports

- **`address already in use`**. Stale container or process holding 8080/18080, unkillable from inside WSL. Fix: `taskkill /F /IM docker-proxy.exe` and `sudo fuser -k 8080/tcp` in the BAT script. (Note: 8080 is the historical default; 18080 is what the current scripts bind to.)
- **LAN team can't access, but `localhost` works.** Windows network profile marked as "Public", or a VPN/ZeroTier virtual adapter was picked by `ipconfig`. Fix: force profile to "Private"; script matches on WLAN/Ethernet keywords to get the real IP.
- **Ngrok frontend white screen.** Ngrok's HTTPS termination conflicts with Label Studio's strict CSP, which blocks blob assets. Fix: use internal HTTP only; `inject_ai.py` covers the case where the frontend is unusable.

### 7.2 WSL & Docker Environment

- **WSL IP drift on every restart.** Fix: WSL2 mirrored networking, so WSL and Windows share the IP.
- **Python 3.14 dependency failures.** Fix: use Conda `ls_ml` on Python 3.10.
- **`bcrypt` error: `AttributeError: module 'bcrypt' has no attribute '__about__'`.** `passlib` incompatible with bcrypt 4.x. Fix: `pip install bcrypt==4.0.1`.
- **SQLite `NOT NULL constraint failed: users.id`.** Primary key defined as `BigInteger` (for PostgreSQL compat), so SQLite doesn't auto-increment. Fix: use `Integer`.

### 7.3 AI Models & VRAM

- **VRAM conflicts crash the system.** Ollama uses 5-6GB, YOLO/Whisper use 1-3GB. Fix: exclusive switching via `switch_*.bat`, which kills non-target backends before starting a new one.
- **Audio model download fails.** Fix: `export HF_ENDPOINT=https://hf-mirror.com`, or place the model file manually in `~/.cache/whisper`.

## Related Work

- **[Ternary Bonsai 2 27B on 8GB VRAM](https://github.com/Alexander390370/Bonsai-27B-8GB-VRAM-Setup)** — the same 8GB card, running a 27B quantized LLM at 64K context. The VRAM management patterns in that project inform the exclusive switching design here.

---

Distributed under the MIT License.
