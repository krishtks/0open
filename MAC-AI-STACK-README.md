# Mac AI Stack — Local AI Assistant with Zscaler MCP Integration

A complete guide to setting up a local AI assistant on Apple Silicon Mac with Zscaler API integration via Telegram.

**Tested on:** MacBook Pro M5 Max 128GB (May 2026)  
**Author:** Krishnan (krishtks)

---

## Stack Overview

| Component | Tool | Role |
|---|---|---|
| 🦙 Model | gemma4:26b via Ollama | Local LLM, 256K context |
| 🤖 Agent | Hermes Agent v0.14+ | Orchestrates tools, manages sessions |
| 🛡️ Zscaler | zscaler-mcp | Live ZIA, ZPA, ZCC, ZDX API access |
| 🧠 Memory | GBrain | Persistent knowledge graph |
| ✈️ Interface | Telegram | Chat from phone or desktop |

---

## Prerequisites

- Apple Silicon Mac with **at least 64GB RAM** (128GB recommended)
- macOS Sequoia or later
- Zscaler tenant with API credentials (ZIA, ZPA, ZCC, ZDX)
- Telegram account and bot token from @BotFather
- GitHub account (optional)

> ⚠️ **Important:** If Zscaler Client Connector (ZCC) is installed on your Mac, you will need SSL bypass rules in ZIA for `api.telegram.org`, `pypi.org`, `github.com`, and `npmjs.org` before starting.

---

## Step 1 — System Setup

### Install Homebrew
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add to PATH (follow the "Next steps" output from installer)
echo >> ~/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv zsh)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv zsh)"
```

### Install Core Tools
```bash
# Ollama — local LLM runtime
brew install ollama
brew services start ollama

# uv — Python package manager (for Zscaler MCP)
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Bun — JavaScript runtime (for GBrain)
curl -fsSL https://bun.sh/install | bash
exec /bin/zsh
```

### Verify installations
```bash
ollama --version    # should show 0.24.0+
uv --version        # should show 0.11+
bun --version       # should show 1.3+
```

---

## Step 2 — Git Setup (Optional)

```bash
git config --global user.name "your-github-username"
git config --global user.email "your@email.com"

# Generate SSH key
ssh-keygen -t ed25519 -C "your@email.com"

# Copy public key and add to github.com/settings/keys
cat ~/.ssh/id_ed25519.pub

# Test
ssh -T git@github.com

# Clone your repos
mkdir -p ~/projects && cd ~/projects
git clone git@github.com:yourusername/your-repo.git
```

---

## Step 3 — Pull AI Models

```bash
# Primary model — 256K context, excellent tool calling on 128GB
ollama pull gemma4:26b

# Embedding model for GBrain semantic search
ollama pull nomic-embed-text

# Optional: backup model
ollama pull qwen3:32b  # Note: only 40K context, use gemma4:26b instead
```

**Model selection guide:**

| RAM | Recommended Model | Context | Tool Calling |
|---|---|---|---|
| 24GB | gemma4 (4B) | 128K | ⚠️ unreliable |
| 48GB | gemma4:26b | 256K | ✅ good |
| 128GB | gemma4:26b | 256K | ✅ excellent |
| 256GB | llama4:maverick | 1M | ✅ outstanding |

---

## Step 4 — Install Hermes Agent

```bash
mkdir -p ~/.hermes
cd ~/.hermes
git clone https://github.com/NousResearch/hermes-agent.git hermes-agent
cd hermes-agent
python3 -m venv venv
source venv/bin/activate
pip install -e '.[all]'

# Add to PATH permanently
echo 'export PATH="$HOME/.hermes/hermes-agent/venv/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Verify
hermes --version
```

---

## Step 5 — Install Zscaler MCP

```bash
uv tool install zscaler-mcp

# Verify
zscaler-mcp --version
```

### Configure Zscaler credentials
```bash
cat > ~/.zscaler-mcp.env << 'EOF'
ZSCALER_CLIENT_ID=your_client_id
ZSCALER_CLIENT_SECRET=your_client_secret
ZSCALER_CUSTOMER_ID=your_customer_id
ZSCALER_VANITY_DOMAIN=your_vanity_domain
ZSCALER_MCP_SERVICES=zia,zpa,zcc,zdx
ZSCALER_MCP_AUTH_ENABLED=false
EOF
chmod 600 ~/.zscaler-mcp.env
```

### Test Zscaler MCP
```bash
export $(grep -v '^#' ~/.zscaler-mcp.env | xargs)
zscaler-mcp --services zia,zpa,zcc,zdx --list-tools 2>/dev/null | wc -l
# Should show 150+ tools
```

---

## Step 6 — Install GBrain

```bash
cd ~
git clone https://github.com/garrytan/gbrain.git gbrain-src
cd gbrain-src
bun install
bun link

# Initialize brain with local embeddings
mkdir -p ~/brain/concepts ~/brain/people ~/brain/projects
gbrain init
# Select: ollama > nomic-embed-text
# Select: conservative mode
```

### Seed your brain
```bash
cat > ~/brain/concepts/zscaler.md << 'EOF'
# Zscaler

## Compiled Truth
Your name and role here.
Products, partners, projects you work on.
EOF

cat > ~/brain/concepts/lab.md << 'EOF'
# Home Lab

## Compiled Truth
Your hardware, domain, network setup.
EOF

gbrain import ~/brain/ --no-embed
gbrain embed --stale

# Verify
gbrain query "test query"
```

---

## Step 7 — Configure Hermes

### Create Hermes config
```bash
python3 << 'PYEOF'
config = """model:
  provider: ollama-launch
  default: gemma4:26b
  base_url: http://127.0.0.1:11434/v1
  api_key: ollama
  context_length: 131072

providers:
  ollama-launch:
    api: http://127.0.0.1:11434/v1
    default_model: gemma4:26b
    models:
      - gemma4:26b
    name: Ollama

auxiliary:
  compression:
    model: gemma4:26b
    context_length: 131072

agent:
  disabled_toolsets:
    - web
    - terminal
    - code_execution

platform_toolsets:
  telegram:
    - hermes-telegram
    - zscaler
    - memory
    - file
    - skills

mcp_servers:
  zscaler:
    command: /Users/YOUR_USERNAME/.local/bin/uvx
    args:
      - --env-file
      - /Users/YOUR_USERNAME/.zscaler-mcp.env
      - zscaler-mcp
      - --services
      - zia,zpa,zcc,zdx
"""
import os
username = os.environ.get('USER', 'username')
config = config.replace('YOUR_USERNAME', username)
with open(os.path.expanduser('~/.hermes/config.yaml'), 'w') as f:
    f.write(config)
print(f"Config saved for user: {username}")
PYEOF
```

---

## Step 8 — Configure Telegram

### Create a Telegram bot
1. Open Telegram and search for **@BotFather**
2. Send `/newbot` and follow prompts
3. Copy the bot token

### Get your Telegram user ID
1. Search for **@userinfobot** on Telegram
2. Send `/start` — it will reply with your numeric ID

### Configure Hermes
```bash
cat > ~/.hermes/.env << 'EOF'
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_ALLOWED_USERS=your_telegram_user_id
EOF
```

---

## Step 9 — Start the Stack

```bash
source ~/.hermes/hermes-agent/venv/bin/activate
hermes gateway run 2>&1 | tee /tmp/hermes-full.log
```

### Verify it's working
```bash
# Check logs
tail -f ~/.hermes/logs/agent.log

# Look for these lines:
# ✓ telegram connected
# MCP: registered 150 tool(s) from 1 server(s)
# Gateway running with 1 platform(s)
```

### Test from Telegram
Send these messages to your bot:
```
hello
list my ZIA URL filtering rules
show my ZPA application segments
list ZCC enrolled devices
```

### Expected log output (success)
```
tool_turns=2  ← Zscaler MCP called
response ready: time=94s
```

---

## Step 10 — Auto-start at Login

```bash
hermes gateway install

# Verify
hermes gateway status
launchctl list | grep hermes
```

---

## Key Commands

```bash
# Gateway management
hermes gateway status      # check if running
hermes gateway restart     # restart
hermes gateway stop        # stop
hermes gateway install     # enable auto-start

# Logs
tail -f ~/.hermes/logs/agent.log    # live agent log

# Models
ollama list                # list installed models
ollama ps                  # show loaded models + context
ollama pull gemma4:26b     # pull a model

# GBrain
gbrain query "search term"          # search your brain
gbrain import ~/brain/ --no-embed   # import new files
gbrain embed --stale                # build embeddings
gbrain doctor --json                # health check
```

---

## Key Files

| File | Purpose |
|---|---|
| `~/.hermes/config.yaml` | Model, MCP, toolset configuration |
| `~/.hermes/.env` | Telegram token, user IDs |
| `~/.zscaler-mcp.env` | Zscaler API credentials |
| `~/.hermes/logs/agent.log` | Agent activity log |
| `~/brain/` | GBrain knowledge files |
| `~/gbrain-src/` | GBrain source code |
| `~/.hermes/hermes-agent/` | Hermes source code |

---

## Troubleshooting

### Hermes command not found
```bash
source ~/.hermes/hermes-agent/venv/bin/activate
# Or permanently:
echo 'export PATH="$HOME/.hermes/hermes-agent/venv/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### tool_turns=0 (model not calling tools)
- Check `platform_toolsets` in config includes `zscaler`
- Clear old sessions: `rm -rf ~/.hermes/sessions/*`
- Use gemma4:26b — it has 256K context which fits all 150 tool definitions

### Context window too small error
- Use gemma4:26b (256K context) not qwen3:32b (40K context)
- Add `context_length: 131072` to model config

### Telegram SSL errors (if ZCC installed)
Add ZIA SSL bypass for these domains:
```
api.telegram.org
*.telegram.org
pypi.org
github.com
npmjs.org
```

Also patch Hermes Telegram SSL:
```bash
sed -i '' 's/def __init__(self, fallback_ips: Iterable\[str\], \*\*transport_kwargs):/def __init__(self, fallback_ips: Iterable[str], **transport_kwargs):\n        transport_kwargs.setdefault("verify", False)/' \
  ~/.hermes/hermes-agent/gateway/platforms/telegram_network.py
```

### Gateway keeps getting killed (launchd conflict)
```bash
launchctl unload ~/Library/LaunchAgents/ai.hermes.gateway.plist 2>/dev/null
pkill -9 -f "hermes_cli"
rm -f ~/.hermes/gateway.lock ~/.hermes/gateway.pid
hermes gateway run
```

### Zscaler 503 errors
Transient API errors — just retry the query. If persistent, check credentials in `~/.zscaler-mcp.env`.

---

## Hardware Recommendations

| Use Case | Minimum RAM | Recommended |
|---|---|---|
| Basic assistant | 16GB | 24GB |
| Zscaler MCP tool calling | 48GB | 128GB |
| Fine-tuning (LoRA) | 64GB | 128GB |
| Full fine-tuning 70B | 128GB | 256GB |

**Recommended machines (2026):**
- MacBook Pro M5 Max 128GB — best all-rounder, portable
- Mac Studio M5 Ultra 256GB — dedicated AI server

---

## What You Can Ask the Bot

```
# Zscaler ZIA
list my ZIA URL filtering rules
show ZIA SSL inspection policies
list ZIA DLP rules
check sandbox quota
show ATP malware settings

# Zscaler ZPA
show my ZPA application segments
list ZPA app connectors
show ZPA access policies
list ZPA provisioning keys

# Zscaler ZCC
list enrolled ZCC devices
show ZCC trusted networks
list ZCC forwarding profiles

# Zscaler ZDX
list ZDX alerts
show ZDX device health
list ZDX applications

# General AI
write a Python script to...
explain how [Zscaler feature] works
help me debug this error: [paste error]
summarize this document: [paste text]
```

---

*Built with ❤️ on MacBook Pro M5 Max 128GB*
*Stack: Ollama + gemma4:26b + Hermes Agent + Zscaler MCP + GBrain + Telegram*
