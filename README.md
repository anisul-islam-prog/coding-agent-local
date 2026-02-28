# credentialless-ai-agent

**Agentic AI without API key exposure. A PCI-DSS/SOC2-compliant architecture for running OpenClaw with local and cloud LLMs.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)](https://www.docker.com/)
[![Ollama](https://img.shields.io/badge/Ollama-Local%20LLM-purple?logo=ollama)](https://ollama.com/)

> **The Problem:** Running AI agents in Docker containers requires cloud API keys. That's a compliance audit failure waiting to happen.  
> **The Solution:** Tunnel cloud LLM requests through a local Ollama gateway—zero credentials in containers, full agentic capability preserved.

---

## Why This Exists

I spent 48 hours trying to force agentic LLMs onto an M2 Mac with 16GB RAM. Hardware failed.  
Then I realized: **resource constraints are a feature, not a bug.**

By architecting around the limitation, I eliminated the #1 credential leak vector in AI infrastructure—while maintaining SOC2/CC6.1 and PCI-DSS Requirement 3 compliance.

**Built for:** Financial services, healthcare, and any environment where "paste your API key here" isn't an option.

## Requirements:
- Mac M2
- 16GB RAM
- 256GB VRAM
- Package manager homebrew installed
- Docker Desktop
  
## 🎯 Recommended Ollama Model
| Model                       | Size   | VRAM Usage | Best For                                                           |
| --------------------------- | ------ | ---------- | ------------------------------------------------------------------ |
| **`qwen2.5-coder:7b`**      | ~4.5GB | ~6-8GB     | **Primary recommendation** - Best quality-to-size ratio for coding |
| **`qwen2.5-coder:14b-instruct-q4_K_M`**| ~9GB | ~10-12GB     |   The optimal model |
| **`deepseek-coder-v2:16b`** | ~9GB   | ~10-12GB   | Alternative for complex reasoning (slower but smarter)             |
| **`phi3:3.8b`**             | ~2.3GB | ~4-5GB     | Ultra-fast fallback for simple completions                         |
## Steps
### 1. Install Ollama
    
```sh    
# Install Ollama natively (required for Metal GPU acceleration)
brew install ollama

# Or use the official installer
curl -fsSL https://ollama.com/install.sh | sh

# Start Ollama service
ollama serve

# Pull the recommended 7B coding model
ollama pull qwen2.5-coder:7b

# Optional: Pull smaller fallback model
ollama pull phi3:3.8b

# Test it works
ollama run qwen2.5-coder:7b "Write a Python function to validate email addresses"

# Check if Ollama is actually running
lsof -i :11434

# Then in another terminal, check if it's using GPU:
ollama ps

# If it shows 100% CPU instead of 100% GPU Metal acceleration is NOT working.
```

#### Optimization of ollama for coding agents

Environment Variables (Add to `~/.zshrc`)
```bash
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_KV_CACHE_TYPE=q4_0
export OLLAMA_NUM_GPU=999
export OLLAMA_MAX_RAM=12GB
```
Apply
```bash
source ~/.zshrc
launchctl setenv OLLAMA_FLASH_ATTENTION 1
launchctl setenv OLLAMA_KV_CACHE_TYPE q4_0
```

Optimal Parameters:
```bash
--num-ctx 4096 
--num-threads 4
```

- `qwen2.5-coder:14b-instruct-q4_K_M` = Best coding performance that fits in 16GB
- `4096` context = Sweet spot for code review/editing
- `4` threads = Matches M2's performance cores
- Flash Attention + Q4 KV cache = ~30% speed boost

#### Optional
Use [OpenWebUI](https://docs.openwebui.com/getting-started/quick-start/) to use the LLM
```bash
# Pull the image
docker pull ghcr.io/open-webui/open-webui:main
# Run the container
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
# Check application is running or not
docker ps

# Go to localhost:3000

# or

# Pull the lightweight image (slim-variant)
docker pull ghcr.io/open-webui/open-webui:main-slim

# Run the container
docker run -d -p 3000:8080 -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main-slim

# Check application is running or not
docker ps

# Go to localhost:3000


# Kill and remove image after use
# Stop and Remove the Container:
docker rm -f open-webui

# Remove the Image (Optional):
docker rmi ghcr.io/open-webui/open-webui:main

# Remove the Volume (Optional, WARNING: Deletes all data): If you want to completely remove your data (chats, settings, etc.):
docker volume rm open-webui

```

### 2. Install dockerized OpenClaw 🦞

#### 🏗️ Architecture Overview

```plain
                        ┌─────────────────────────────────────────────────────────────┐
                        │                    Docker Host (Your Mac)                   │
                        │  ┌───────────────────────────────────────────────────────┐  │
                        │  │              OpenClaw Container                       │  │
                        │  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │  │
                        │  │  │   OpenClaw   │  │   Config     │  │   Model     │  │  │
                        │  │  │   Binary     │  │   Switcher   │  │   Scripts   │  │  │
                        │  │  └──────────────┘  └──────────────┘  └─────────────┘  │  │
                        │  │           │                  │                │       │  │
                        │  │           └──────────────────┴────────────────┘       │  │
                        │  │                          │                            │  │
                        │  │              Persistent Volume (openclaw_data)        │  │
                        │  └──────────────────────────┼─────────────────────────── ┘  │
                        │                             │                               │
                        │                    Bridge Network (Internet Enabled)        │
                        │                             │                               │
                        │              ┌──────────────┼──────────────┐                │
                        │              ▼              ▼              ▼                │
                        │        [Ollama API]  [Web Search]   [GitHub API]            │
                        │        (host.docker   (Serper.dev,   (GitHub CLI)           │
                        │         .internal)     Brave API)                           │
                        └─────────────────────────────────────────────────────────────┘
```

#### 📁 Project Structure

Before Docker can reach `ollama`, configure it to accept external connections:
```bash
# Add to your ~/.zshrc or ~/.bash_profile
export OLLAMA_HOST="0.0.0.0"

# After adding restart the terminal
source ~/.zshrc
```
Then restart Ollama:
```bash
# Kill existing Ollama processes
pkill ollama

# Restart with host binding
OLLAMA_HOST=0.0.0.0 ollama serve

# Should return JSON list of models
curl http://localhost:11434/api/tags
```

```bash
# Create persistent volume to use
mkdir -p ~/openclaw-project/{config,docker}
cd ~/openclaw-project
```


#### 🐳  Docker Compose Configuration

Create `~/openclaw-project/docker/docker-compose.yml`:

[docker-compose.yml](/openclaw-project/docker/docker-compose.yml)

#### 🔧 Scripts

Create a script file `~/openclaw-project/claw` to manage:

[claw](/openclaw-project/claw)

Make `claw` executable by giving it `+x` permission:

```bash
chmod +x ~/openclaw-project/claw
```


Add `claw` to your `~/.bash_profile` or `~/.zshrc` as alias to be able to use it in terminal:

```bash
# Alias added for managing docker openclaw
alias claw="$HOME/openclaw-project/claw"

# After adding restart the terminal
source ~/.zshrc
```

or Use

```bash
# Add to ~/.zshrc for global access
echo 'alias claw="$HOME/openclaw-project/claw"' >> ~/.zshrc
source ~/.zshrc
```

#### ⚙️ Environment Configuration

Create `.env`:
```bash
cd ~/openclaw-project/docker

# Copy .env.example
cp .env.exaample .env

# Generate token to use as OPENCLAW_GATEWAY_TOKEN in .env
openssl rand -hex 32

# Copy the generated token in .env and set to OPENCLAW_GATEWAY_TOKEN
# After copying load the env
source ~/openclaw-project/docker/.env
```
or


```bash
cat > ~/openclaw-project/docker/.env << 'EOF'
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
GITHUB_TOKEN=ghp_your_github_token_here
BRAVE_API_KEY=optional_for_search
EOF
```

Create `~/openclaw-project/config/openclaw.json`. This will contain the configurations of openclaw. (this gets mounted to `~/.openclaw/openclaw.json`):

1. [openclaw-local.json](/openclaw-project/config/openclaw-local.json)

2. [openclaw-cloud.json](/openclaw-project/config/openclaw-cloud.json)

Config for models running locally:

Copy to the config location:
```bash
mkdir -p ~/.openclaw
cp ~/openclaw-project/config/openclaw-local.json ~/.openclaw/openclaw.json
```

Config for models running on Ollama Cloud:

Copy to the config location:
```bash
mkdir -p ~/.openclaw
cp ~/openclaw-project/config/openclaw-cloud.json ~/.openclaw/openclaw.json
```


#### 🚀 Deployment & Usage

First-Time Setup & Onboarding:

Pull required Ollama models for running locally:
 
```bash
ollama pull qwen2.5-coder:14b-instruct-q4_K_M
ollama pull qwen2.5-coder:7b-instruct-q4_K_M
ollama pull qwen2.5-coder:7b
ollama pull phi3:3.8b
ollama pull deepseek-coder-v2:16b
```

## Quick Start

```bash
# Start services (auto-updates images)
claw start

# Open Control UI in browser
claw ui

# Check status
claw health

# Run your first agent command
claw agent -m "Write a Python script to fetch weather data"
```

## Available Commands

### System Management
| Command | Description |
|---------|-------------|
| `claw start` | Pull latest images and start gateway |
| `claw stop` | Stop all services |
| `claw restart` | Restart gateway |
| `claw status` | Show Docker container status |
| `claw logs` | Follow gateway logs |
| `claw update` | Update to latest OpenClaw image |
| `claw health` | Check if gateway is responding |

### Interactive Access
| Command | Description |
|---------|-------------|
| `claw shell` | Open interactive CLI container |
| `claw ui` | Open Control UI in browser |
| `claw url` | Print Control UI URL with token |
| `claw token` | Display gateway authentication token |

### Agent Operations
| Command | Description |
|---------|-------------|
| `claw agent -m "message"` | Run agent with message |
| `claw agent-switch` | Select and switch active agent |
| `claw model` | Select and change default model |

### Device Management
| Command | Description |
|---------|-------------|
| `claw devices` | List pending and paired devices |
| `claw approve` | Approve ALL pending devices |
| `claw approve-device ID` | Approve specific device |

### Configuration
| Command | Description |
|---------|-------------|
| `claw fix-origins` | Set Control UI allowed origins |
| `claw fix-ollama-auth` | Create Ollama authentication profile |
| `claw regenerate-token` | Generate new gateway token |
| `claw doctor` | Run health checks |

## Available Agents

| Agent | Model | Best For |
|-------|-------|----------|
| `kimi` | Kimi K2.5 Cloud | Long context, reasoning |
| `minimax` | MiniMax M2.5 Cloud | Balanced performance |
| `gpt-oss` | GPT-OSS Cloud | General coding tasks |
| `qwen-14b` | Qwen 2.5 Coder 14B | Local, fast inference |
| `qwen-7b` | Qwen 2.5 Coder 7B | Lightweight, efficient |
| `phi3` | Phi-3 3.8B | Ultra-fast, minimal RAM |

## Usage Examples

### Switch Models
```bash
# Interactive model selection with auto-restart
claw model
# Select from: kimi, minimax, gpt-oss, qwen-14b, qwen-7b, phi3
```

### Run Agent with Specific Model
```bash
# Use default agent
claw agent -m "Explain recursion in Python"

# Use specific agent
claw agent -m "Debug this code" --agent kimi

# Use specific model directly
claw agent -m "Write tests" --model ollama/phi3:3.8b
```

### GitHub Operations
```bash
# Agent can clone, modify, commit, and push
claw agent -m "Clone my-repo, add a README, and push to GitHub"
```

### Web Search
```bash
# Agent automatically uses DuckDuckGo for research
claw agent -m "Find the latest Python 3.13 features and summarize"
```

## Troubleshooting

### "Pairing Required" Error
```bash
# Approve browser or CLI device
claw devices    # List pending
claw approve    # Approve all
```

### "No API Key Found for Ollama"
```bash
# Create Ollama auth profile
claw fix-ollama-auth
```

### "Timeout Discovering Ollama Models"
- Ensure Ollama is running on your Mac: `ollama serve`
- Check Ollama is accessible: `curl http://localhost:11434/api/tags`
- Verify `host.docker.internal` resolves in Docker

### Slow Model Loading
- Use smaller models: `phi3:3.8b` or `qwen2.5-coder:7b`
- Keep Ollama models loaded: `export OLLAMA_KEEP_ALIVE=-1`
- Increase Ollama timeout if needed

### Control UI Not Loading
```bash
# Fix allowed origins
claw fix-origins
claw restart
```

## Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Your Mac      │──── │  Docker Desktop  │──── │  OpenClaw       │
│  (Ollama)       │     │  (Port 18789)    │     │  Gateway        │
│  :11434         │     │                  │     │  WebSocket+HTTP │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                              │
                              ▼
                        ┌──────────────────┐
                        │  Control UI      │
                        │http://localhost  │
                        │:18789            │
                        └──────────────────┘
```

## Ports

| Port | Service |
|------|---------|
| 18789 | Gateway WebSocket + Control UI |
| 18791 | Browser control (internal only) |

## Configuration Files

- `~/.openclaw/openclaw.json` - Main configuration
- `~/.openclaw/agents/main/agent/` - Agent profiles
- `~/openclaw-workspace/` - Agent workspace (GitHub sync)

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENCLAW_GATEWAY_TOKEN` | Yes | Authentication token |
| `GITHUB_TOKEN` | No | For GitHub operations |
| `BRAVE_API_KEY` | No | For web search |
| `OLLAMA_BASE_URL` | Auto | Ollama connection URL |

## Updating

```bash
# Update images and restart
claw update
```

## Uninstalling

```bash
claw stop
rm -rf ~/openclaw-project ~/.openclaw ~/openclaw-workspace
```

## Resources

- OpenClaw Docs: https://docs.openclaw.ai
- Ollama Models: https://ollama.com/library
- GitHub Token: https://github.com/settings/tokens

## License

MIT - Use at your own risk.
