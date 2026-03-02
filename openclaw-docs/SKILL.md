---
name: openclaw-docs
description: OpenClaw complete documentation reference. Activate when questions about OpenClaw configuration, setup, channels, tools, automation, CLI commands, troubleshooting, or architecture arise. Read the appropriate references/ file for detailed info.
metadata: {"openclaw":{"always":true}}
---

# OpenClaw Documentation Reference

This skill provides comprehensive OpenClaw documentation. For detailed information on any topic, read the corresponding file from `{baseDir}/references/`.

## Quick Reference

### What is OpenClaw?

OpenClaw is a personal AI agent gateway that connects LLMs (Claude, GPT, etc.) to messaging platforms (WhatsApp, Telegram, Discord, Slack, Feishu, Signal, iMessage) and gives them tools (shell exec, browser, web search, file I/O, sub-agents, cron jobs). It runs as a background service on your machine (Mac/Linux/VPS/Raspberry Pi).

### Architecture Overview

```
[Messaging Channels] → [Gateway (Node.js)] → [Agent Runtime] → [LLM Provider API]
                              ↕                      ↕
                        [Session Store]         [Tools/Skills]
                        [Cron/Heartbeat]        [Workspace Files]
```

- **Gateway**: The always-on Node.js daemon that handles routing, sessions, channels, and tool execution
- **Agent Runtime**: Embedded runtime (derived from pi-mono) that manages LLM interactions
- **Workspace**: `~/.openclaw/workspace/` — agent's working directory with AGENTS.md, SOUL.md, etc.
- **Config**: `~/.openclaw/openclaw.json` (JSON5 format)

### Installation (Quick)

```bash
# npm (recommended)
npm i -g openclaw

# Or via pnpm
pnpm add -g openclaw

# First-time setup
openclaw onboard

# Start the gateway service
openclaw gateway start
```

Docker: `docker pull openclaw/openclaw` — see `{baseDir}/references/getting-started.md`

### Essential Config (`~/.openclaw/openclaw.json`)

```json5
{
  // Model provider
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      workspace: "~/.openclaw/workspace",
    },
  },
  // Gateway
  gateway: {
    mode: "local",
    port: 3578,
    bind: "loopback",
    auth: { mode: "token", token: "your-token" },
  },
  // Channels (add the ones you need)
  channels: {
    telegram: { enabled: true, botToken: "...", allowFrom: ["tg:123456"] },
    discord: { enabled: true, token: "..." },
    whatsapp: { allowFrom: ["+15551234567"] },
  },
}
```

### Supported Channels

| Channel | Auth Method | Key Config |
|---------|------------|------------|
| WhatsApp | QR code web link | `channels.whatsapp.allowFrom` |
| Telegram | Bot token (BotFather) | `channels.telegram.botToken` |
| Discord | Bot token | `channels.discord.token` |
| Slack | Bot + App tokens | `channels.slack.botToken` + `appToken` |
| Feishu/Lark | App credentials | `channels.feishu.appId` + `appSecret` |
| Signal | signal-cli | `channels.signal` |
| iMessage | macOS only, BlueBubbles | `channels.imessage` |
| Google Chat | Service account | `channels.googlechat` |
| MS Teams | Bot framework | `channels.msteams` |

### Supported Model Providers

- **Anthropic** (Claude models) — API key or setup-token/OAuth
- **OpenAI** (GPT models) — API key or Codex OAuth
- **OpenRouter** — aggregator for many models
- **Venice AI** — privacy-focused inference
- **Ollama/vLLM** — local models
- **Moonshot/Kimi**, **Mistral**, **Gemini**, **Bedrock**, **Together**, **MiniMax**, and more

Set via: `agents.defaults.model.primary: "provider/model"`

### Key CLI Commands

```bash
openclaw status              # Quick health check
openclaw gateway start|stop|restart  # Service management
openclaw onboard             # Setup wizard
openclaw configure           # Interactive config editor
openclaw doctor              # Health checks + fixes
openclaw channels list       # Show configured channels
openclaw channels status --probe  # Deep channel health
openclaw cron list           # List scheduled jobs
openclaw sessions            # View sessions
openclaw models list         # Available models
openclaw skills list         # Available skills
openclaw logs --follow       # Live gateway logs
openclaw security audit      # Security check
```

### Tools Available to Agent

- **exec/process**: Shell command execution with background support
- **read/write/edit**: File I/O operations
- **browser**: Playwright-based browser automation
- **web_search/web_fetch**: Web search (Brave API) and content fetching
- **subagents**: Spawn child agents for parallel work
- **message**: Send messages via channels
- **nodes**: Control paired devices (camera, screen, exec)
- **canvas**: Present UI canvases
- **image**: Analyze images
- **tts**: Text-to-speech

### Automation

- **Cron Jobs**: `openclaw cron add` — schedule recurring agent tasks
- **Heartbeat**: Periodic agent check-ins (default 30m) with HEARTBEAT.md
- **Hooks**: Event-driven automation (file changes, webhooks)
- **Webhooks**: HTTP endpoints for external triggers

### Skills System

Skills are loaded from three locations (workspace wins on conflict):
1. `<workspace>/skills/` (highest priority)
2. `~/.openclaw/skills/` (managed/local)
3. Bundled skills (shipped with install)

Browse/install: `npx clawhub install <skill-slug>` or visit https://clawhub.com

### Session Management

- Sessions auto-reset daily at 4 AM local time (configurable)
- `/new` or `/reset` starts fresh session
- `session.dmScope: "per-channel-peer"` for multi-user isolation
- Group chats get isolated sessions automatically

### Troubleshooting (Quick)

```bash
# Run this ladder in order:
openclaw status
openclaw gateway status
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

Common issues:
- **No replies**: Check allowFrom/pairing, run `openclaw pairing list`
- **Gateway won't start**: Check port conflict, auth config, mode=local
- **Channel won't connect**: Verify tokens, run `openclaw doctor`

## Detailed Reference Files

For in-depth documentation, read the appropriate reference file:

| Topic | File |
|-------|------|
| Getting Started & Setup | `{baseDir}/references/getting-started.md` |
| Gateway Configuration | `{baseDir}/references/gateway-config.md` |
| Core Concepts | `{baseDir}/references/concepts.md` |
| Messaging Channels | `{baseDir}/references/channels.md` |
| Tools & Skills | `{baseDir}/references/tools.md` |
| Automation (Cron/Hooks) | `{baseDir}/references/automation.md` |
| CLI Commands | `{baseDir}/references/cli.md` |
| Model Providers | `{baseDir}/references/providers.md` |
| Troubleshooting & FAQ | `{baseDir}/references/troubleshooting.md` |

## Official Resources

- Docs: https://docs.openclaw.ai
- Full docs index: https://docs.openclaw.ai/llms.txt
- Skills marketplace: https://clawhub.com
- GitHub: https://github.com/nicobailon/openclaw
- Discord community: https://discord.gg/openclaw
