# Getting Started & Setup

## Installation

### Requirements
- Node.js v22+ (LTS recommended)
- npm, pnpm, or yarn
- macOS, Linux, or Windows (WSL recommended)
- Works on Raspberry Pi (ARM64)

### Install Methods

**npm (recommended for most users):**
```bash
npm i -g openclaw
```

**pnpm:**
```bash
pnpm add -g openclaw
```

**Docker:**
```bash
docker pull openclaw/openclaw
docker run -d --name openclaw \
  -v ~/.openclaw:/root/.openclaw \
  -p 3578:3578 \
  openclaw/openclaw
```

**From source (hackable):**
```bash
git clone https://github.com/nicobailon/openclaw.git
cd openclaw
npm install
npm run build
```

### First-Time Setup

```bash
# Interactive setup wizard (recommended)
openclaw onboard

# Or minimal setup
openclaw setup
```

The wizard walks through:
1. Model provider authentication (Anthropic/OpenAI/OpenRouter/etc.)
2. Gateway configuration (port, bind, auth)
3. Channel setup (WhatsApp/Telegram/Discord)
4. Skills installation
5. Gateway service installation

### Onboard Options

```bash
openclaw onboard --mode local          # Local gateway mode
openclaw onboard --mode remote         # Connect to remote gateway
openclaw onboard --flow quickstart     # Quick setup
openclaw onboard --flow advanced       # Manual configuration
openclaw onboard --non-interactive     # No prompts
openclaw onboard --reset               # Reset config before wizard
```

## Setup Wizard (`openclaw setup`)

```bash
openclaw setup --workspace ~/my-workspace
openclaw setup --wizard                # Run onboarding wizard
```

Creates `~/.openclaw/openclaw.json` if missing and initializes workspace files.

## Bootstrapping

On first run, OpenClaw creates these workspace files:
- `AGENTS.md` — Operating instructions and memory conventions
- `SOUL.md` — Agent persona, boundaries, tone
- `TOOLS.md` — User-maintained tool notes
- `BOOTSTRAP.md` — One-time first-run ritual (deleted after completion)
- `IDENTITY.md` — Agent name/vibe/emoji
- `USER.md` — User profile

To disable bootstrap file creation for pre-seeded workspaces:
```json5
{ agent: { skipBootstrap: true } }
```

## Updating

```bash
# npm
npm update -g openclaw

# pnpm
pnpm update -g openclaw

# From source
cd ~/openclaw && git pull && npm install && npm run build

# Self-update (ask the agent)
# The agent can run openclaw's update mechanism
```

After updating: `openclaw gateway restart`

Check version: `openclaw --version`

## Docker Setup

### docker-compose.yml
```yaml
version: '3.8'
services:
  openclaw:
    image: openclaw/openclaw:latest
    container_name: openclaw
    restart: unless-stopped
    ports:
      - "3578:3578"
    volumes:
      - ~/.openclaw:/root/.openclaw
    environment:
      - ANTHROPIC_API_KEY=sk-ant-...
```

### Docker Environment Variables
- `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.
- `OPENCLAW_CONFIG_PATH` — custom config file path
- `OPENCLAW_STATE_DIR` — custom state directory

## Where Things Live

| Path | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main config (JSON5) |
| `~/.openclaw/workspace/` | Agent workspace (default) |
| `~/.openclaw/agents/<id>/` | Per-agent state + sessions |
| `~/.openclaw/credentials/` | Channel credentials (WhatsApp, etc.) |
| `~/.openclaw/skills/` | Managed/local skills |
| `~/.openclaw/logs/` | Gateway logs |

## VPS / Remote Setup

For running on a VPS:
1. Install Node.js and openclaw
2. Set `gateway.bind: "tailnet"` or `"lan"` (not loopback)
3. Configure auth: `gateway.auth.mode: "token"` with a strong token
4. Use Tailscale for secure remote access (recommended)
5. Install as system service: `openclaw gateway install`

Minimum VPS: 1 vCPU, 1GB RAM, Ubuntu 22.04+
