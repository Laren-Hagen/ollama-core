# Stack Implementation Guide: Ollama + Open Web UI

**Stack**: Personal workstation — single-user local deployment  
**Compose file**: `docker-compose.yml` in this directory  
**Access URL**: http://localhost:3000

---

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Environment Setup](#2-environment-setup)
3. [GPU vs CPU Mode](#3-gpu-vs-cpu-mode)
4. [Starting the Stack](#4-starting-the-stack)
5. [First Model Pull](#5-first-model-pull)
6. [Verifying the Stack](#6-verifying-the-stack)
7. [Accessing Open Web UI](#7-accessing-open-web-ui)
8. [Stopping and Restarting](#8-stopping-and-restarting)
9. [Configuration Reference](#9-configuration-reference)
10. [Non-Obvious Design Decisions](#10-non-obvious-design-decisions)

---

## 1. Prerequisites

### Required

| Requirement | Minimum Version | Notes |
|-------------|----------------|-------|
| Docker Engine | 24.0+ | Compose Spec `deploy.resources` requires Docker Compose v2. Check: `docker compose version` |
| Docker Compose | v2.20+ (bundled with Docker Desktop 4.22+) | Standalone Compose v1 (`docker-compose`) is not supported |
| Available RAM | 8 GB free (CPU mode) or 2 GB free (GPU mode) | Ollama uses VRAM in GPU mode; system RAM is minimal |
| Available disk | 10 GB+ | Varies by model. `llama3.1:8b` (Q4_K_M) requires ~5.5 GB |

### Required for NVIDIA GPU Mode (optional)

| Requirement | Notes | Source |
|-------------|-------|--------|
| NVIDIA driver 531+ | Check: `nvidia-smi` | [NVIDIA driver install](https://www.nvidia.com/Download/index.aspx) |
| NVIDIA Container Toolkit | Enables GPU passthrough to Docker containers | [Install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) |
| Docker configured for NVIDIA runtime | Run after toolkit install | See GPU setup steps below |

**GPU prerequisites — setup commands (Linux):**

```bash
# Install NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use the NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify
docker run --rm --gpus all nvidia/cuda:12.0-base-ubuntu22.04 nvidia-smi
```

Sources:
- [NVIDIA Container Toolkit — Install Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [Ollama Docker — NVIDIA GPU](https://docs.ollama.com/docker)

---

## 2. Environment Setup

Before starting the stack, copy the template and fill in the required values:

```bash
# Step 1: Copy the template
cp .env.template .env

# Step 2: Open .env in your editor
#   Fill in the three REQUIRED values:
#     WEBUI_SECRET_KEY     — generate with: openssl rand -hex 32
#     WEBUI_ADMIN_EMAIL    — your admin login email (e.g., admin@localhost)
#     WEBUI_ADMIN_PASSWORD — a strong password for the admin account
```

**Generating `WEBUI_SECRET_KEY`:**

```bash
openssl rand -hex 32
# Example output: a3f2c1d4e5b6789012345678901234567890abcdef1234567890abcdef12345678
# Copy the output into .env as the value for WEBUI_SECRET_KEY
```

**Why `WEBUI_SECRET_KEY` is mandatory:**  
Open Web UI uses this key to sign JWTs (session tokens). If the key is not set, the container auto-generates one at startup. The auto-generated key is created fresh on every container **recreate** (not just restart), which invalidates all active user sessions. Setting it explicitly in `.env` keeps sessions stable across recreates. Source: [Open Web UI env config — WEBUI_SECRET_KEY](https://docs.openwebui.com/reference/env-configuration/#webui_secret_key)

**`.env` security checklist before starting:**

- [ ] `WEBUI_SECRET_KEY` is set to a 64-character hex string (output of `openssl rand -hex 32`)
- [ ] `WEBUI_ADMIN_EMAIL` is set
- [ ] `WEBUI_ADMIN_PASSWORD` is set to a strong, unique password
- [ ] `.env` is listed in `.gitignore` — verify with `git check-ignore -v .env`
- [ ] You have NOT committed `.env` to any branch

---

## 3. GPU vs CPU Mode

### GPU Mode (default — requires NVIDIA GPU + Container Toolkit)

The `docker-compose.yml` includes an NVIDIA GPU passthrough block by default:

```yaml
# In docker-compose.yml — ollama service — deploy.resources.reservations:
devices:
  - driver: nvidia
    count: all
    capabilities: [gpu]
```

This is already in place. If the NVIDIA Container Toolkit is installed and the host has a compatible GPU, Ollama will use the GPU automatically with no further changes.

**Minimum GPU**: 8 GB VRAM (RTX 3060 12 GB, RTX 3070, RTX 4060, or equivalent).  
The default 8192 token context with `llama3.1:8b` Q4_K_M requires ~6.2 GB VRAM.

**Verifying GPU is in use after startup:**

```bash
docker exec -it ollama ollama ps
# Look for: PROCESSOR column showing "100% GPU"
```

### CPU Mode (no NVIDIA GPU)

1. Open `docker-compose.yml`
2. Find the `devices:` block under `ollama → deploy → resources → reservations`
3. **Remove** the entire `devices:` section (approximately 8 lines):

   ```yaml
   # DELETE these lines for CPU mode:
         devices:
           - driver: nvidia
             count: all
             capabilities: [gpu]
   ```

4. Save the file. No environment variable changes are needed.

Ollama detects the absence of a compatible GPU and falls back to CPU inference automatically.

**CPU mode performance expectations** (approximate, varies by hardware):

| Hardware | Throughput | Notes |
|----------|-----------|-------|
| Modern desktop CPU (Ryzen 7 / Core i7) | 6–12 tok/s | With llama3.1:8b Q4_K_M |
| Laptop CPU (Ryzen 5 / Core i5) | 3–7 tok/s | Slower, but functional |
| RAM requirement | 8 GB free | Model weights load into system RAM instead of VRAM |

**Verifying CPU mode after startup:**

```bash
docker exec -it ollama ollama ps
# Look for: PROCESSOR column showing "100% CPU"
```

Source: [Ollama Docker — CPU and GPU](https://docs.ollama.com/docker)

---

## 4. Starting the Stack

```bash
# From the ollama-core/ directory:

# Pull images and start all services in detached mode
docker compose up -d

# Follow logs to confirm both services start cleanly
docker compose logs -f

# Check service health status
docker compose ps
```

Expected output from `docker compose ps` when healthy:

```
NAME          IMAGE                                    STATUS
ollama        ollama/ollama:0.22.1                     Up X minutes (healthy)
open-webui    ghcr.io/open-webui/open-webui:v0.9.2    Up X minutes (healthy)
```

Open Web UI will not reach `(healthy)` until Ollama is `(healthy)` first, because of the `depends_on: condition: service_healthy` dependency. This is expected. Both services should be healthy within 2–3 minutes of first start.

---

## 5. First Model Pull

The Ollama container starts without any models. Pull a model before using Open Web UI:

```bash
# Pull llama3.1:8b (Q4_K_M quantization, ~5.5 GB download)
# This is a capable general-purpose 8B chat model suitable for this stack.
docker exec -it ollama ollama pull llama3.1:8b

# Alternative: smaller model for limited resources
docker exec -it ollama ollama pull llama3.2:3b    # ~2.0 GB, faster on CPU

# List pulled models
docker exec ollama ollama list
```

**Note:** `llama3.1:8b` is the correct Ollama tag for Meta's Llama 3.1 8B parameter model. The `llama3.2` family contains 1B, 3B, and 11B (vision) sizes — there is no 8B variant in that release. Source: [Ollama model library](https://ollama.com/library)

**After pulling a model:**
- Return to http://localhost:3000
- Select the model from the model selector in the top-left of the chat interface
- Models pulled into Ollama appear automatically; no restart is required

---

## 6. Verifying the Stack

### Ollama API

```bash
# Confirm the Ollama API is responding (from the host — if port is published for debug)
curl http://localhost:11434/api/version

# Confirm Open Web UI can reach Ollama via internal Docker DNS
docker exec open-webui curl -sf http://ollama:11434/api/version
# Expected: {"version":"0.22.1"} (or similar JSON)
```

### Open Web UI

```bash
# Check health endpoint
curl http://localhost:3000/health
# Expected: {"status":"ok"} or HTTP 200

# Check service health from Docker
docker compose ps
# Both services should show (healthy)
```

### Volume persistence test

```bash
# Recreate the stack (simulates upgrade or restart scenario)
docker compose down
docker compose up -d

# After both services are healthy:
# 1. Browse to http://localhost:3000
# 2. Your admin account should still exist (DB persisted via open-webui-data volume)
# 3. Models should still be listed (persisted via ollama-models volume)
# 4. Your existing session should remain valid (WEBUI_SECRET_KEY was stable)
```

Source: [Open Web UI integration spec — Verification Checklist](../mission-command/missions/147-stack-design-and-implementation/deliverables/open-web-ui-integration-spec.md)

---

## 7. Accessing Open Web UI

| URL | Notes |
|-----|-------|
| http://localhost:3000 | Primary browser access point |
| http://localhost:3000/health | Health check endpoint |

The UI is bound to `127.0.0.1:3000` — accessible only from the local machine.  
To allow LAN access (other devices on your network), change the port mapping in `docker-compose.yml` from `"127.0.0.1:3000:8080"` to `"3000:8080"` and restart the stack.

**First login:**
- Email: the value you set for `WEBUI_ADMIN_EMAIL` in `.env`
- Password: the value you set for `WEBUI_ADMIN_PASSWORD` in `.env`

---

## 8. Stopping and Restarting

```bash
# Stop services (containers removed, volumes preserved)
docker compose down

# Stop and remove volumes — DESTRUCTIVE: deletes all models and chat history
# docker compose down -v    # DO NOT RUN unless you intend to wipe all data

# Restart a single service
docker compose restart ollama
docker compose restart open-webui

# View live logs
docker compose logs -f
docker compose logs -f ollama
docker compose logs -f open-webui
```

---

## 9. Configuration Reference

All tunable settings are in `.env`. The most commonly adjusted values:

| Variable | Default | When to change |
|----------|---------|----------------|
| `OLLAMA_CONTEXT_LENGTH` | `8192` | Increase for longer conversations; requires more VRAM/RAM |
| `OLLAMA_KEEP_ALIVE` | `5m` | Set `-1` to keep model permanently loaded; `0` to free VRAM immediately |
| `WEBUI_SECRET_KEY` | *(must be set)* | Rotate if accidentally exposed |
| `WEBUI_ADMIN_PASSWORD` | *(must be set)* | Change via UI after first login for better security hygiene |

After changing `.env`, restart the affected service:

```bash
docker compose up -d --force-recreate ollama       # for Ollama env changes
docker compose up -d --force-recreate open-webui   # for Open Web UI env changes
```

---

## 10. Non-Obvious Design Decisions

### Why Ollama's port 11434 is not published to the host

`OLLAMA_HOST=0.0.0.0:11434` binds the Ollama API server to all container network interfaces so that Open Web UI (a sibling container on the same `ollama-net` bridge) can reach it by service name (`http://ollama:11434`). Binding to `0.0.0.0` inside the container is required for this internal DNS-based routing to work.

Critically, this does **not** expose the port to the host machine. Port 11434 is not in the `ports:` block, so it is only reachable from within `ollama-net`. Publishing 11434 to the host would expose the unauthenticated Ollama API (no API key required by default) to any process on the host machine.

Source: [Ollama FAQ — How can I expose Ollama on my network?](https://docs.ollama.com/faq)

### Why `OLLAMA_BASE_URL` must use the service name, not localhost

Inside the Open Web UI container, `localhost` and `127.0.0.1` refer to the Open Web UI container's own loopback interface — not the host machine, and not the Ollama container. Using `http://ollama:11434` relies on Docker's internal DNS resolution, which maps the service name `ollama` to the Ollama container's IP address within `ollama-net`.

Source: [Open Web UI — Connection Tips](https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-ollama/#connection-tips)

### Why `read_only: true` is combined with `tmpfs` and named volumes

Both services run with `read_only: true` (read-only root filesystem). Docker volume mounts — both named volumes and `tmpfs` entries — are applied on top of the read-only root filesystem and are independently writable.

**Ollama** has two writable mount points:
- `tmpfs` at `/root/.ollama` — ephemeral, for Ollama non-model state (identity keys, runtime config)
- Named volume `ollama-models` at `/root/.ollama/models` — persistent, for model weights

The named volume is mounted as a child of the tmpfs. This creates a stacked mount: writes to `/root/.ollama/` (non-models) go to the tmpfs (ephemeral); writes to `/root/.ollama/models/` go to the named volume (persistent). Ollama non-model state (identity keys) does not need to persist for this single-container stack.

**Open Web UI** has:
- `tmpfs` at `/tmp` — for transient Python temp files
- Named volume `open-webui-data` at `/app/backend/data` — persistent, for SQLite DB and uploads
- `PYTHONDONTWRITEBYTECODE=1` — prevents Python from attempting to write `__pycache__` bytecode to the read-only package directories

### Why `ENABLE_SIGNUP=False` even though `WEBUI_ADMIN_EMAIL`/`WEBUI_ADMIN_PASSWORD` disables it automatically

`WEBUI_ADMIN_EMAIL` + `WEBUI_ADMIN_PASSWORD` disable signup automatically after the admin account is created on first run. However, `ENABLE_SIGNUP` is a `PersistentConfig` variable — once written to the database it persists across container restarts. Setting it explicitly to `False` in the environment provides a persistent, documented lock that is visible in the compose configuration.

Source: [Open Web UI env config — ENABLE_SIGNUP](https://docs.openwebui.com/reference/env-configuration/#enable_signup)

### Why resource `reservations` are set alongside `limits`

`deploy.resources.reservations` documents the minimum resources the container needs to start and function. On a workstation with limited free memory, Docker will refuse to schedule the container if reservations cannot be satisfied, giving a clear error rather than an OOM kill after startup. The Ollama reservation of 8 GB RAM ensures the container starts only when adequate system RAM is available for CPU-mode inference.

Source: [Docker Compose — deploy.resources](https://docs.docker.com/compose/compose-file/deploy/)

### If you enable RAG or embedding features in Open Web UI

Open Web UI downloads sentence-transformer embedding models to `~/.cache` at runtime when RAG features are enabled. These models are re-downloaded on every container recreate because `/root/.cache` is not backed by a named volume. To persist embedding models, add a named volume entry:

```yaml
# In docker-compose.yml — open-webui service:
volumes:
  - open-webui-data:/app/backend/data
  - open-webui-cache:/root/.cache    # add this line

# In the top-level volumes: block:
  open-webui-cache:
    driver: local
```

This is not included in the default configuration because RAG is not part of this stack's initial scope and the additional volume adds complexity.
