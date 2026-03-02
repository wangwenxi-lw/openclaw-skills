# Gateway Configuration

## Overview

The Gateway is the always-on Node.js daemon that handles:
- Message routing from channels to the agent
- Session management and persistence
- Tool execution and sandboxing
- Cron jobs and heartbeats
- Authentication and security

Config file: `~/.openclaw/openclaw.json` (JSON5 format — comments and trailing commas allowed)

## Essential Configuration

### Minimal Config

```json5
{
  gateway: {
    mode: "local",
    port: 3578,
  },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

### Gateway Modes

- `local` — Run gateway locally (default for onboarding)
- `remote` — Connect to a remote gateway

### Bind Options

```json5
{
  gateway: {
    bind: "loopback",    // localhost only (default, safest)
    // bind: "lan",      // LAN access (requires auth)
    // bind: "tailnet",  // Tailscale network (requires auth)
    // bind: "auto",     // Auto-detect
    // bind: "0.0.0.0",  // All interfaces (requires auth)
  },
}
```

**Important**: Non-loopback bind requires authentication (`gateway.auth`).

### Authentication

```json5
{
  gateway: {
    auth: {
      mode: "token",           // token | password
      token: "your-secure-token",
      // password: "your-password",  // alternative to token
    },
  },
}
```

- Token auth: Bearer token for API/WebSocket access
- Password auth: Hashed password for human-facing access
- Loopback (localhost) can skip auth but it's recommended

### Security

```json5
{
  gateway: {
    security: {
      rateLimiting: { enabled: true },
    },
  },
}
```

Run `openclaw security audit` for security checks.

## Configuration Reference (Key Fields)

### agents.defaults

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      workspace: "~/.openclaw/workspace",
      userTimezone: "America/New_York",
      timeFormat: "auto",  // auto | 12 | 24
      // Heartbeat
      heartbeat: {
        every: "30m",
        target: "last",
        activeHours: { start: "08:00", end: "24:00" },
      },
      // Sandbox
      sandbox: {
        mode: "off",  // off | non-main | all
      },
      // Bootstrap file limits
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 150000,
      // Block streaming
      blockStreamingDefault: "off",
    },
  },
}
```

### Session Configuration

```json5
{
  session: {
    dmScope: "main",              // main | per-peer | per-channel-peer | per-account-channel-peer
    mainKey: "main",
    reset: {
      mode: "daily",
      atHour: 4,                  // 4 AM local time
      idleMinutes: 120,           // optional idle timeout
    },
    resetTriggers: ["/new", "/reset"],
    maintenance: {
      mode: "warn",               // warn | enforce
      pruneAfter: "30d",
      maxEntries: 500,
      rotateBytes: "10mb",
    },
  },
}
```

### Channel Defaults

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist",   // allowlist | open | disabled
    },
  },
}
```

DM Policies: `pairing` (default), `allowlist`, `open`, `disabled`

### Tools Configuration

```json5
{
  tools: {
    exec: {
      enabled: true,
      applyPatch: false,
    },
    browser: {
      enabled: true,
    },
    web: {
      search: { enabled: true },
      fetch: { enabled: true },
    },
  },
}
```

### Skills Configuration

```json5
{
  skills: {
    entries: {
      "skill-name": {
        enabled: true,
        apiKey: "...",
        env: { API_KEY: "value" },
      },
    },
    load: {
      watch: true,
      watchDebounceMs: 250,
      extraDirs: ["/path/to/shared/skills"],
    },
  },
}
```

## Remote Gateway

For accessing gateway from another machine:

```json5
{
  gateway: {
    mode: "local",
    bind: "tailnet",  // or "lan" or "0.0.0.0"
    auth: { mode: "token", token: "secure-random-token" },
  },
}
```

Client side (remote mode):
```bash
openclaw onboard --mode remote --remote-url https://your-gateway:3578 --remote-token your-token
```

## Tailscale Integration

```json5
{
  gateway: {
    bind: "tailnet",
    tailscale: {
      serve: true,     // Expose via Tailscale Serve (HTTPS)
      // funnel: true,  // Expose via Tailscale Funnel (public)
    },
  },
}
```

Setup: `openclaw onboard --tailscale serve`

## Sandboxing

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",  // off | non-main | all
        scope: "session",  // session | agent
        docker: {
          image: "node:22-slim",
          setupCommand: "apt-get update && apt-get install -y git",
        },
      },
    },
  },
}
```

- `off`: No sandboxing
- `non-main`: Sandbox non-main sessions (sub-agents, cron)
- `all`: Sandbox all sessions including main

## Heartbeat

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",           // Interval (0m to disable)
        target: "last",         // none | last | channel-name
        to: "+15551234567",     // Optional specific recipient
        prompt: "...",          // Custom prompt
        activeHours: {
          start: "08:00",
          end: "22:00",
          timezone: "America/New_York",
        },
      },
    },
  },
}
```

## Configuration Examples

### Single user, WhatsApp + Telegram
```json5
{
  gateway: { mode: "local", port: 3578 },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      workspace: "~/.openclaw/workspace",
      heartbeat: { every: "30m", target: "last" },
    },
  },
  channels: {
    whatsapp: { allowFrom: ["+15551234567"] },
    telegram: { botToken: "123:ABC", allowFrom: ["tg:123456"] },
  },
}
```

### VPS with Tailscale
```json5
{
  gateway: {
    mode: "local",
    bind: "tailnet",
    auth: { mode: "token", token: "secure-token-here" },
    tailscale: { serve: true },
  },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

## Troubleshooting Gateway

```bash
openclaw gateway status     # Check if running
openclaw gateway probe      # Test reachability
openclaw doctor             # Auto-fix common issues
openclaw logs --follow      # Live logs
```

Common issues:
- **EADDRINUSE**: Port already in use → change port or stop other process
- **Auth required**: Non-loopback bind needs `gateway.auth`
- **Mode not set**: Set `gateway.mode: "local"` explicitly
