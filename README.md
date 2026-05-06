# Agent-zero-stack
Agent-Zero

Here’s a practical, copy‑pasteable guide for deploying Agent Zero on:

- Hostinger KVM 2 VPS (recommended layout, HTTPS‑ready)
- Local Ubuntu machine via Docker / Docker Compose

I’ll assume you’re comfortable with SSH, Docker, and basic Linux.

***

## 1. What you need first

- Hostinger KVM 2 VPS with Ubuntu 22.04 (or Hostinger’s Docker template).[1][2][3]
- Local Ubuntu 22.04+ box (for the local section).  
- Docker Engine and Docker Compose plugin on both.[4][5][6]
- An LLM provider key (OpenRouter works great and is recommended by Agent Zero docs).[1][4]

Official image and docs:

- Image: `agent0ai/agent-zero:latest`[5][4]
- Docs: [Agent Zero installation page](https://www.agent-zero.ai/p/docs/installation/) and GitHub install doc.[4][5]

***

## 2. Install Docker & Compose (VPS + local Ubuntu)

Run these steps on both your Hostinger VPS and local Ubuntu if they don’t already have Docker.

```bash
# Update
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker’s official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker apt repo
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start
sudo systemctl enable docker
sudo systemctl start docker

# Optional: run docker as non-root (logout/login after this)
sudo usermod -aG docker "$USER"

# Verify
docker --version
docker compose version
```

This pattern is the same as typical VPS Docker guides and Agent Zero‑style hosting docs.[6][5][4]

***

## 3. Minimal Docker Compose for Agent Zero (reusable)

Here is a clean `docker-compose.yml` that works both on Hostinger and local Ubuntu. It exposes HTTP on `50080` and persists Agent Zero data under `./data`.

```yaml
version: "3.9"

services:
  agent-zero:
    image: agent0ai/agent-zero:latest
    container_name: agent-zero
    restart: unless-stopped
    ports:
      - "50080:80"          # host:container
    environment:
      # Auth for Agent Zero web UI
      A0_USERNAME: "admin"
      A0_PASSWORD: "change_me_strong_password"

      # Example: OpenRouter key; you can also configure in UI later
      OPENROUTER_API_KEY: "your_openrouter_api_key_here"

      # Optional: tweak base URL if behind reverse proxy
      # A0_BASE_URL: "https://agent.your-domain.com"

    volumes:
      - .//a0/usr      # persistent user data, logs, configs

    # Optional extra constraints
    # deploy:
    #   resources:
    #     limits:
    #       cpus: "2.0"
    #       memory: "6G"
```

- Port `80` inside the container is the default for Agent Zero’s web UI.[5][4]
- The volume `/a0/usr` is the documented data directory for persistent user data.[4][5]

You can keep this in a repo and reference it as your “compose link” (e.g., GitHub Gist). The VPS guide you saw mentions “Paste the contents from the Agent Zero GitHub Gist” — this file is structurally equivalent.[1]

***

## 4. Deploy Agent Zero on Hostinger KVM 2

### 4.1. Create VPS and prepare OS

1. In Hostinger, choose the **KVM 2** plan; it’s explicitly recommended for Agent Zero‑class workloads (2 vCPU, 8 GB RAM).[3][1]
2. Use Ubuntu 22.04 or Hostinger’s **Docker** template (preinstalls Docker + Compose).[2]
3. Add your SSH key in the panel, then create the server.  

SSH in:

```bash
ssh root@your_vps_ip
```

If you didn’t choose the Docker template, run the Docker install from section 2.

### 4.2. Create Agent Zero project folder

```bash
mkdir -p /opt/agent-zero
cd /opt/agent-zero
nano docker-compose.yml   # or vim
```

Paste the `docker-compose.yml` from section 3 and adjust:

- `A0_USERNAME` and `A0_PASSWORD` to your desired creds.[4]
- `OPENROUTER_API_KEY` or any other provider keys you want baked in.[1][4]

### 4.3. Start Agent Zero

```bash
cd /opt/agent-zero

# Pull latest image
docker pull agent0ai/agent-zero:latest

# Start in detached mode
docker compose up -d
```

The pattern `docker compose up -d` with a dedicated directory is consistent with typical VPS deployment guides for Agent Zero‑style apps.[6][1]

Check status:

```bash
docker compose ps
docker compose logs -f agent-zero   # Ctrl+C to exit logs
```

### 4.4. Open firewall and access

On Hostinger’s Ubuntu, UFW may be disabled by default. If you use it:

```bash
sudo ufw allow 50080/tcp
sudo ufw reload
```

Now open in your browser:

```text
http://YOUR_VPS_IP:50080
```

You should see Agent Zero’s onboarding/login screen similar to the official docs’ first run experience (username/password prompt, port selection, onboarding flow).[4]

### 4.5. Optional: Add domain + HTTPS (Caddy example)

If you want a clean `https://agent.your-domain.com` instead of `http://ip:50080`, run a Caddy reverse proxy in the same compose file:

```yaml
version: "3.9"

services:
  agent-zero:
    image: agent0ai/agent-zero:latest
    container_name: agent-zero
    restart: unless-stopped
    environment:
      A0_USERNAME: "admin"
      A0_PASSWORD: "change_me_strong_password"
      OPENROUTER_API_KEY: "your_openrouter_api_key_here"
      A0_BASE_URL: "https://agent.your-domain.com"
    volumes:
      - .//a0/usr

  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
```

Create `Caddyfile` in `/opt/agent-zero`:

```caddy
agent.your-domain.com {
    reverse_proxy agent-zero:80
}
```

- Point `agent.your-domain.com` A record to your VPS IP in DNS (Cloudflare or wherever).  
- Caddy will auto‑issue a Let’s Encrypt cert. This setup mirrors standard reverse‑proxy guides for similar agents.[7]

***

## 5. Run Agent Zero locally in Docker (Ubuntu workstation)

This is almost the same as the VPS, just with localhost ports and optional local LLM integration (e.g., Ollama).

### 5.1. Pull and run with plain Docker (no Compose)

If you just want fastest path:

```bash
# Pull image
docker pull agent0ai/agent-zero:latest

# Quick run (no persistence)
docker run -d \
  --name agent-zero \
  -p 50080:80 \
  -e A0_USERNAME=admin \
  -e A0_PASSWORD=change_me_strong_password \
  agent0ai/agent-zero:latest
```

This is the basic run pattern documented by Agent Zero’s official installation page.[4]

For persistence:

```bash
mkdir -p ~/agent-zero-data

docker run -d \
  --name agent-zero \
  -p 50080:80 \
  -v ~/agent-zero-/a0/usr \
  -e A0_USERNAME=admin \
  -e A0_PASSWORD=change_me_strong_password \
  agent0ai/agent-zero:latest
```

- Open `http://localhost:50080` to access the UI.[4]

### 5.2. Run with Docker Compose locally

Create `~/agent-zero/docker-compose.yml`:

```bash
mkdir -p ~/agent-zero
cd ~/agent-zero
nano docker-compose.yml
```

Paste:

```yaml
version: "3.9"

services:
  agent-zero:
    image: agent0ai/agent-zero:latest
    container_name: agent-zero
    restart: unless-stopped
    ports:
      - "50080:80"
    environment:
      A0_USERNAME: "admin"
      A0_PASSWORD: "change_me_strong_password"
    volumes:
      - .//a0/usr
```

Run:

```bash
docker compose up -d
```

The Agent Zero Docker install docs explicitly support this style: map host port to container `80` and persist `/a0/usr`.[5][4]

***

## 6. Local models with Ollama (optional but powerful)

If you want a fully local stack (Agent Zero + Ollama) via Docker Compose, use a combined file. This is the pattern shown in “Agent Zero + Ollama” Docker setups.[8][4]

```yaml
version: "3.9"

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ./ollama:/root/.ollama

  agent-zero:
    image: agent0ai/agent-zero:latest
    container_name: agent-zero
    restart: unless-stopped
    depends_on:
      - ollama
    ports:
      - "50080:80"
    environment:
      A0_USERNAME: "admin"
      A0_PASSWORD: "change_me_strong_password"
    volumes:
      - .//a0/usr
```

Then:

```bash
docker compose up -d
```

Configure Ollama in Agent Zero UI:

1. Open Agent Zero → model config screen.[4]
2. Select **Ollama** as provider, set base URL to `http://host.docker.internal:11434` on macOS/Windows, or `http://ollama:11434` inside the same Docker network on Linux.[4]
3. Enter the model name you have pulled in Ollama (e.g., `llama3`, `qwen2`, etc.), save.  

The official docs describe using `host.docker.internal` or host IP when connecting to local LLM services.[4]

***

## 7. Useful Docker commands for maintaining Agent Zero

On VPS or local, inside your project directory (`/opt/agent-zero` or `~/agent-zero`):

```bash
# Check containers
docker compose ps

# Live logs
docker compose logs -f agent-zero

# Restart
docker compose restart agent-zero

# Stop and remove (without deleting volumes)
docker compose down

# Full clean including volumes (be careful!)
docker compose down -v
```

These are the same commands used in typical Agent Zero on‑VPS guides.[6]

***

If you want, I can:

- Tailor a production‑ready `docker-compose.yml` for your exact domain + Cloudflare + Caddy/Traefik style.  
- Generate a GitHub‑ready `README.md` (with the compose file and the exact “compose link” you can share with your team).

Sources
[1] How to Deploy Agent Zero on a VPS — The Ultimate Open-Source ... https://www.alphire.com/how-to-deploy-agent-zero-on-a-vps-the-ultimate-open-source-ai-agent/
[2] OpenClaw on Hostinger VPS: Step-by-Step Setup https://www.remoteopenclaw.com/blog/hostinger-openclaw-vps-setup-guide
[3] Agent Zero VPS Hosting | One-Click Docker Setup - Hostinger https://www.hostinger.com/in/vps/docker/agent-zero
[4] Installation - Agent Zero https://www.agent-zero.ai/p/docs/installation/
[5] agent-zero/docs/setup/installation.md at main - GitHub https://github.com/agent0ai/agent-zero/blob/main/docs/setup/installation.md
[6] Agent Zero AI Setup Guide on RamNode VPS | Autonomous AI Agent https://www.ramnode.com/guides/agentzero
[7] Hermes Agent on Hostinger VPS — Deploy Guide 2026 https://openclawlaunch.com/guides/hermes-agent-hostinger
[8] Run AI Locally FREE! Agent Zero + Ollama (Docker Compose Setup) https://www.youtube.com/watch?v=LiZHqmuSLSc
[9] Deploying to a VPS just got Way Easier - YouTube https://www.youtube.com/watch?v=ZmL46xVdYzM
[10] Hostinger VPS for n8n: The Ultimate Setup Guide | Mike Murphy LLC https://www.facebook.com/mikemurphyco/videos/hostinger-vps-for-n8n-the-ultimate-setup-guide/2008800693242446/
