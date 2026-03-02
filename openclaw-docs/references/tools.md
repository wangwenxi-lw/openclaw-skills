# OpenClaw Tools Reference

## Overview


# Tools (OpenClaw)

OpenClaw exposes **first-class agent tools** for browser, canvas, nodes, and cron.
These replace the old `openclaw-*` skills: the tools are typed, no shelling,
and the agent should rely on them directly.

## Disabling tools

You can globally allow/deny tools via `tools.allow` / `tools.deny` in `openclaw.json`
(deny wins). This prevents disallowed tools from being sent to model providers.

```json5
{
  tools: { deny: ["browser"] },
}
```

Notes:

- Matching is case-insensitive.
- `*` wildcards are supported (`"*"` means all tools).
- If `tools.allow` only references unknown or unloaded plugin tool names, OpenClaw logs a warning and ignores the allowlist so core tools stay available.

## Tool profiles (base allowlist)

`tools.profile` sets a **base tool allowlist** before `tools.allow`/`tools.deny`.
Per-agent override: `agents.list[].tools.profile`.

Profiles:

- `minimal`: `session_status` only
- `coding`: `group:fs`, `group:runtime`, `group:sessions`, `group:memory`, `image`
- `messaging`: `group:messaging`, `sessions_list`, `sessions_history`, `sessions_send`, `session_status`
- `full`: no restriction (same as unset)

Example (messaging-only by default, allow Slack + Discord tools too):

```json5
{
  tools: {
    profile: "messaging",
    allow: ["slack", "discord"],
  },
}
```

Example (coding profile, but deny exec/process everywhere):

```json5
{
  tools: {
    profile: "coding",
    deny: ["group:runtime"],
  },
}
```

Example (global coding profile, messaging-only support agent):

```json5
{
  tools: { profile: "coding" },
  agents: {
    list: [
      {
        id: "support",
        tools: { profile: "messaging", allow: ["slack"] },
      },
    ],
  },
}
```

## Provider-specific tool policy

Use `tools.byProvider` to **further restrict** tools for specific providers
(or a single `provider/model`) without changing your global defaults.
Per-agent override: `agents.list[].tools.byProvider`.

This is applied **after** the base tool profile and **before** allow/deny lists,
so it can only narrow the tool set.
Provider keys accept either `provider` (e.g. `google-antigravity`) or
`provider/model` (e.g. `openai/gpt-5.2`).

Example (keep global coding profile, but minimal tools for Google Antigravity):

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
    },
  },
}
```

Example (provider/model-specific allowlist for a flaky endpoint):

```json5
{
  tools: {
    allow: ["group:fs", "group:runtime", "sessions_list"],
    byProvider: {
      "openai/gpt-5.2": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

Example (agent-specific override for a single provider):

```json5
{
  agents: {
    list: [
      {
        id: "support",
        tools: {
          byProvider: {
            "google-antigravity": { allow: ["message", "sessions_list"] },
          },
        },
      },
    ],
  },
}
```

## Tool groups (shorthands)

Tool policies (global, agent, sandbox) support `group:*` entries that expand to multiple tools.
Use these in `tools.allow` / `tools.deny`.

Available groups:

- `group:runtime`: `exec`, `bash`, `process`
- `group:fs`: `read`, `write`, `edit`, `apply_patch`
- `group:sessions`: `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status`
- `group:memory`: `memory_search`, `memory_get`
- `group:web`: `web_search`, `web_fetch`
- `group:ui`: `browser`, `canvas`
- `group:automation`: `cron`, `gateway`
- `group:messaging`: `message`
- `group:nodes`: `nodes`
- `group:openclaw`: all built-in OpenClaw tools (excludes provider plugins)

Example (allow only file tools + browser):

```json5
{
  tools: {
    allow: ["group:fs", "browser"],
  },
}
```

## Plugins + tools

Plugins can register **additional tools** (and CLI commands) beyond the core set.
See [Plugins](/tools/plugin) for install + config, and [Skills](/tools/skills) for how
tool usage guidance is injected into prompts. Some plugins ship their own skills
alongside tools (for example, the voice-call plugin).

Optional plugin tools:

- [Lobster](/tools/lobster): typed workflow runtime with resumable approvals (requires the Lobster CLI on the gateway host).
- [LLM Task](/tools/llm-task): JSON-only LLM step for structured workflow output (optional schema validation).

## Tool inventory

### `apply_patch`

Apply structured patches across one or more files. Use for multi-hunk edits.
Experimental: enable via `tools.exec.applyPatch.enabled` (OpenAI models only).
`tools.exec.applyPatch.workspaceOnly` defaults to `true` (workspace-contained). Set it to `false` only if you intentionally want `apply_patch` to write/delete outside the workspace directory.

### `exec`

Run shell commands in the workspace.

Core parameters:

- `command` (required)
- `yieldMs` (auto-background after timeout, default 10000)
- `background` (immediate background)
- `timeout` (seconds; kills the process if exceeded, default 1800)
- `elevated` (bool; run on host if elevated mode is enabled/allowed; only changes behavior when the agent is sandboxed)
- `host` (`sandbox | gateway | node`)
- `security` (`deny | allowlist | full`)
- `ask` (`off | on-miss | always`)
- `node` (node id/name for `host=node`)
- Need a real TTY? Set `pty: true`.

Notes:

- Returns `status: "running"` with a `sessionId` when backgrounded.
- Use `process` to poll/log/write/kill/clear background sessions.
- If `process` is disallowed, `exec` runs synchronously and ignores `yieldMs`/`background`.
- `elevated` is gated by `tools.elevated` plus any `agents.list[].tools.elevated` override (both must allow) and is an alias for `host=gateway` + `security=full`.
- `elevated` only changes behavior when the agent is sandboxed (otherwise it’s a no-op).
- `host=node` can target a macOS companion app or a headless node host (`openclaw node run`).
- gateway/node approvals and allowlists: [Exec approvals](/tools/exec-approvals).

### `process`

Manage background exec sessions.

Core actions:

- `list`, `poll`, `log`, `write`, `kill`, `clear`, `remove`

Notes:

- `poll` returns new output and exit status when complete.
- `log` supports line-based `offset`/`limit` (omit `offset` to grab the last N lines).
- `process` is scoped per agent; sessions from other agents are not visible.

### `loop-detection` (tool-call loop guardrails)

OpenClaw tracks recent tool-call history and blocks or warns when it detects repetitive no-progress loops.
Enable with `tools.loopDetection.enabled: true` (default is `false`).

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
      warningThreshold: 10,
      criticalThreshold: 20,
      globalCircuitBreakerThreshold: 30,
      historySize: 30,
      detectors: {
        genericRepeat: true,
        knownPollNoProgress: true,
        pingPong: true,
      },
    },
  },
}
```

- `genericRepeat`: repeated same tool + same params call pattern.
- `knownPollNoProgress`: repeating poll-like tools with identical outputs.
- `pingPong`: alternating `A/B/A/B` no-progress patterns.
- Per-agent override: `agents.list[].tools.loopDetection`.

### `web_search`

Search the web using Brave Search API.

Core parameters:

- `query` (required)
- `count` (1–10; default from `tools.web.search.maxResults`)

Notes:

- Requires a Brave API key (recommended: `openclaw configure --section web`, or set `BRAVE_API_KEY`).
- Enable via `tools.web.search.enabled`.
- Responses are cached (default 15 min).
- See [Web tools](/tools/web) for setup.

### `web_fetch`

Fetch and extract readable content from a URL (HTML → markdown/text).

Core parameters:

- `url` (required)
- `extractMode` (`markdown` | `text`)
- `maxChars` (truncate long pages)

Notes:

- Enable via `tools.web.fetch.enabled`.
- `maxChars` is clamped by `tools.web.fetch.maxCharsCap` (default 50000).
- Responses are cached (default 15 min).
- For JS-heavy sites, prefer the browser tool.
- See [Web tools](/tools/web) for setup.
- See [Firecrawl](/tools/firecrawl) for the optional anti-bot fallback.

### `browser`

Control the dedicated OpenClaw-managed browser.

Core actions:

- `status`, `start`, `stop`, `tabs`, `open`, `focus`, `close`
- `snapshot` (aria/ai)
- `screenshot` (returns image block + `MEDIA:<path>`)
- `act` (UI actions: click/type/press/hover/drag/select/fill/resize/wait/evaluate)
- `navigate`, `console`, `pdf`, `upload`, `dialog`

Profile management:

- `profiles` — list all browser profiles with status
- `create-profile` — create new profile with auto-allocated port (or `cdpUrl`)
- `delete-profile` — stop browser, delete user data, remove from config (local only)
- `reset-profile` — kill orphan process on profile's port (local only)

Common parameters:

- `profile` (optional; defaults to `browser.defaultProfile`)
- `target` (`sandbox` | `host` | `node`)
- `node` (optional; picks a specific node id/name)
  Notes:
- Requires `browser.enabled=true` (default is `true`; set `false` to disable).
- All actions accept optional `profile` parameter for multi-instance support.
- When `profile` is omitted, uses `browser.defaultProfile` (defaults to "chrome").
- Profile names: lowercase alphanumeric + hyphens only (max 64 chars).
- Port range: 18800-18899 (~100 profiles max).
- Remote profiles are attach-only (no start/stop/reset).
- If a browser-capable node is connected, the tool may auto-route to it (unless you pin `target`).
- `snapshot` defaults to `ai` when Playwright is installed; use `aria` for the accessibility tree.
- `snapshot` also supports role-snapshot options (`interactive`, `compact`, `depth`, `selector`) which return refs like `e12`.
- `act` requires `ref` from `snapshot` (numeric `12` from AI snapshots, or `e12` from role snapshots); use `evaluate` for rare CSS selector needs.
- Avoid `act` → `wait` by default; use it only in exceptional cases (no reliable UI state to wait on).
- `upload` can optionally pass a `ref` to auto-click after arming.
- `upload` also supports `inputRef` (aria ref) or `element` (CSS selector) to set `<input type="file">` directly.

### `canvas`

Drive the node Canvas (present, eval, snapshot, A2UI).

Core actions:

- `present`, `hide`, `navigate`, `eval`
- `snapshot` (returns image block + `MEDIA:<path>`)
- `a2ui_push`, `a2ui_reset`

Notes:

- Uses gateway `node.invoke` under the hood.
- If no `node` is provided, the tool picks a default (single connected node or local mac node).
- A2UI is v0.8 only (no `createSurface`); the CLI rejects v0.9 JSONL with line errors.
- Quick smoke: `openclaw nodes canvas a2ui push --node <id> --text "Hello from A2UI"`.

### `nodes`

Discover and target paired nodes; send notifications; capture camera/screen.

Core actions:

- `status`, `describe`
- `pending`, `approve`, `reject` (pairing)
- `notify` (macOS `system.notify`)
- `run` (macOS `system.run`)
- `camera_snap`, `camera_clip`, `screen_record`
- `location_get`

Notes:

- Camera/screen commands require the node app to be foregrounded.
- Images return image blocks + `MEDIA:<path>`.
- Videos return `FILE:<path>` (mp4).
- Location returns a JSON payload (lat/lon/accuracy/timestamp).
- `run` params: `command` argv array; optional `cwd`, `env` (`KEY=VAL`), `commandTimeoutMs`, `invokeTimeoutMs`, `needsScreenRecording`.

Example (`run`):

```json
{
  "action": "run",
  "node": "office-mac",
  "command": ["echo", "Hello"],
  "env": ["FOO=bar"],
  "commandTimeoutMs": 12000,
  "invokeTimeoutMs": 45000,
  "needsScreenRecording": false
}
```

### `image`

Analyze an image with the configured image model.

Core parameters:

- `image` (required path or URL)
- `prompt` (optional; defaults to "Describe the image.")
- `model` (optional override)
- `maxBytesMb` (optional size cap)

Notes:

- Only available when `agents.defaults.imageModel` is configured (primary or fallbacks), or when an implicit image model can be inferred from your default model + configured auth (best-effort pairing).
- Uses the image model directly (independent of the main chat model).

### `message`

Send messages and channel actions across Discord/Google Chat/Slack/Telegram/WhatsApp/Signal/iMessage/MS Teams.

Core actions:

- `send` (text + optional media; MS Teams also supports `card` for Adaptive Cards)
- `poll` (WhatsApp/Discord/MS Teams polls)
- `react` / `reactions` / `read` / `edit` / `delete`
- `pin` / `unpin` / `list-pins`
- `permissions`
- `thread-create` / `thread-list` / `thread-reply`
- `search`
- `sticker`
- `member-info` / `role-info`
- `emoji-list` / `emoji-upload` / `sticker-upload`
- `role-add` / `role-remove`
- `channel-info` / `channel-list`
- `voice-status`
- `event-list` / `event-create`
- `timeout` / `kick` / `ban`

Notes:

- `send` routes WhatsApp via the Gateway; other channels go direct.
- `poll` uses the Gateway for WhatsApp and MS Teams; Discord polls go direct.
- When a message tool call is bound to an active chat session, sends are constrained to that session’s target to avoid cross-context leaks.

### `cron`

Manage Gateway cron jobs and wakeups.

Core actions:

- `status`, `list`
- `add`, `update`, `remove`, `run`, `runs`
- `wake` (enqueue system event + optional immediate heartbeat)

Notes:

- `add` expects a full cron job object (same schema as `cron.add` RPC).
- `update` uses `{ jobId, patch }` (`id` accepted for compatibility).

### `gateway`

Restart or apply updates to the running Gateway process (in-place).

Core actions:

- `restart` (authorizes + sends `SIGUSR1` for in-process restart; `openclaw gateway` restart in-place)
- `config.get` / `config.schema`
- `config.apply` (validate + write config + restart + wake)
- `config.patch` (merge partial update + restart + wake)
- `update.run` (run update + restart + wake)

Notes:

- Use `delayMs` (defaults to 2000) to avoid interrupting an in-flight reply.
- `restart` is enabled by default; set `commands.restart: false` to disable it.

### `sessions_list` / `sessions_history` / `sessions_send` / `sessions_spawn` / `session_status`

List sessions, inspect transcript history, or send to another session.

Core parameters:

- `sessions_list`: `kinds?`, `limit?`, `activeMinutes?`, `messageLimit?` (0 = none)
- `sessions_history`: `sessionKey` (or `sessionId`), `limit?`, `includeTools?`
- `sessions_send`: `sessionKey` (or `sessionId`), `message`, `timeoutSeconds?` (0 = fire-and-forget)
- `sessions_spawn`: `task`, `label?`, `runtime?`, `agentId?`, `model?`, `thinking?`, `cwd?`, `runTimeoutSeconds?`, `thread?`, `mode?`, `cleanup?`
- `session_status`: `sessionKey?` (default current; accepts `sessionId`), `model?` (`default` clears override)

Notes:

- `main` is the canonical direct-chat key; global/unknown are hidden.
- `messageLimit > 0` fetches last N messages per session (tool messages filtered).
- Session targeting is controlled by `tools.sessions.visibility` (default `tree`: current session + spawned subagent sessions). If you run a shared agent for multiple users, consider setting `tools.sessions.visibility: "self"` to prevent cross-session browsing.
- `sessions_send` waits for final completion when `timeoutSeconds > 0`.
- Delivery/announce happens after completion and is best-effort; `status: "ok"` confirms the agent run finished, not that the announce was delivered.
- `sessions_spawn` supports `runtime: "subagent" | "acp"` (`subagent` default). For ACP runtime behavior, see [ACP Agents](/tools/acp-agents).
- `sessions_spawn` starts a sub-agent run and posts an announce reply back to the requester chat.
  - Supports one-shot mode (`mode: "run"`) and persistent thread-bound mode (`mode: "session"` with `thread: true`).
  - If `thread: true` and `mode` is omitted, mode defaults to `session`.
  - `mode: "session"` requires `thread: true`.
  - If `runTimeoutSeconds` is omitted, OpenClaw uses `agents.defaults.subagents.runTimeoutSeconds` when set; otherwise timeout defaults to `0` (no timeout).
  - Discord thread-bound flows depend on `session.threadBindings.*` and `channels.discord.threadBindings.*`.
  - Reply format includes `Status`, `Result`, and compact stats.
  - `Result` is the assistant completion text; if missing, the latest `toolResult` is used as fallback.
- Manual completion-mode spawns send directly first, with queue fallback and retry on transient failures (`status: "ok"` means run finished, not that announce delivered).
- `sessions_spawn` is non-blocking and returns `status: "accepted"` immediately.
- `sessions_send` runs a reply‑back ping‑pong (reply `REPLY_SKIP` to stop; max turns via `session.agentToAgent.maxPingPongTurns`, 0–5).
- After the ping‑pong, the target agent runs an **announce step**; reply `ANNOUNCE_SKIP` to suppress the announcement.
- Sandbox clamp: when the current session is sandboxed and `agents.defaults.sandbox.sessionToolsVisibility: "spawned"`, OpenClaw clamps `tools.sessions.visibility` to `tree`.

### `agents_list`

List agent ids that the current session may target with `sessions_spawn`.

Notes:

- Result is restricted to per-agent allowlists (`agents.list[].subagents.allowAgents`).
- When `["*"]` is configured, the tool includes all configured agents and marks `allowAny: true`.

## Parameters (common)

Gateway-backed tools (`canvas`, `nodes`, `cron`):

- `gatewayUrl` (default `ws://127.0.0.1:18789`)
- `gatewayToken` (if auth enabled)
- `timeoutMs`

Note: when `gatewayUrl` is set, include `gatewayToken` explicitly. Tools do not inherit config
or environment credentials for overrides, and missing explicit credentials is an error.

Browser tool:

- `profile` (optional; defaults to `browser.defaultProfile`)
- `target` (`sandbox` | `host` | `node`)
- `node` (optional; pin a specific node id/name)

## Recommended agent flows

Browser automation:

1. `browser` → `status` / `start`
2. `snapshot` (ai or aria)
3. `act` (click/type/press)
4. `screenshot` if you need visual confirmation

Canvas render:

1. `canvas` → `present`
2. `a2ui_push` (optional)
3. `snapshot`

Node targeting:

1. `nodes` → `status`
2. `describe` on the chosen node
3. `notify` / `run` / `camera_snap` / `screen_record`

## Safety

- Avoid direct `system.run`; use `nodes` → `run` only with explicit user consent.
- Respect user consent for camera/screen capture.
- Use `status/describe` to ensure permissions before invoking media commands.

## How tools are presented to the agent

Tools are exposed in two parallel channels:

1. **System prompt text**: a human-readable list + guidance.
2. **Tool schema**: the structured function definitions sent to the model API.

That means the agent sees both “what tools exist” and “how to call them.” If a tool
doesn’t appear in the system prompt or the schema, the model cannot call it.

## Skills


# Skills (OpenClaw)

OpenClaw uses **[AgentSkills](https://agentskills.io)-compatible** skill folders to teach the agent how to use tools. Each skill is a directory containing a `SKILL.md` with YAML frontmatter and instructions. OpenClaw loads **bundled skills** plus optional local overrides, and filters them at load time based on environment, config, and binary presence.

## Locations and precedence

Skills are loaded from **three** places:

1. **Bundled skills**: shipped with the install (npm package or OpenClaw.app)
2. **Managed/local skills**: `~/.openclaw/skills`
3. **Workspace skills**: `<workspace>/skills`

If a skill name conflicts, precedence is:

`<workspace>/skills` (highest) → `~/.openclaw/skills` → bundled skills (lowest)

Additionally, you can configure extra skill folders (lowest precedence) via
`skills.load.extraDirs` in `~/.openclaw/openclaw.json`.

## Per-agent vs shared skills

In **multi-agent** setups, each agent has its own workspace. That means:

- **Per-agent skills** live in `<workspace>/skills` for that agent only.
- **Shared skills** live in `~/.openclaw/skills` (managed/local) and are visible
  to **all agents** on the same machine.
- **Shared folders** can also be added via `skills.load.extraDirs` (lowest
  precedence) if you want a common skills pack used by multiple agents.

If the same skill name exists in more than one place, the usual precedence
applies: workspace wins, then managed/local, then bundled.

## Plugins + skills

Plugins can ship their own skills by listing `skills` directories in
`openclaw.plugin.json` (paths relative to the plugin root). Plugin skills load
when the plugin is enabled and participate in the normal skill precedence rules.
You can gate them via `metadata.openclaw.requires.config` on the plugin’s config
entry. See [Plugins](/tools/plugin) for discovery/config and [Tools](/tools) for the
tool surface those skills teach.

## ClawHub (install + sync)

ClawHub is the public skills registry for OpenClaw. Browse at
[https://clawhub.com](https://clawhub.com). Use it to discover, install, update, and back up skills.
Full guide: [ClawHub](/tools/clawhub).

Common flows:

- Install a skill into your workspace:
  - `clawhub install <skill-slug>`
- Update all installed skills:
  - `clawhub update --all`
- Sync (scan + publish updates):
  - `clawhub sync --all`

By default, `clawhub` installs into `./skills` under your current working
directory (or falls back to the configured OpenClaw workspace). OpenClaw picks
that up as `<workspace>/skills` on the next session.

## Security notes

- Treat third-party skills as **untrusted code**. Read them before enabling.
- Prefer sandboxed runs for untrusted inputs and risky tools. See [Sandboxing](/gateway/sandboxing).
- `skills.entries.*.env` and `skills.entries.*.apiKey` inject secrets into the **host** process
  for that agent turn (not the sandbox). Keep secrets out of prompts and logs.
- For a broader threat model and checklists, see [Security](/gateway/security).

## Format (AgentSkills + Pi-compatible)

`SKILL.md` must include at least:

```markdown
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
```

Notes:

- We follow the AgentSkills spec for layout/intent.
- The parser used by the embedded agent supports **single-line** frontmatter keys only.
- `metadata` should be a **single-line JSON object**.
- Use `{baseDir}` in instructions to reference the skill folder path.
- Optional frontmatter keys:
  - `homepage` — URL surfaced as “Website” in the macOS Skills UI (also supported via `metadata.openclaw.homepage`).
  - `user-invocable` — `true|false` (default: `true`). When `true`, the skill is exposed as a user slash command.
  - `disable-model-invocation` — `true|false` (default: `false`). When `true`, the skill is excluded from the model prompt (still available via user invocation).
  - `command-dispatch` — `tool` (optional). When set to `tool`, the slash command bypasses the model and dispatches directly to a tool.
  - `command-tool` — tool name to invoke when `command-dispatch: tool` is set.
  - `command-arg-mode` — `raw` (default). For tool dispatch, forwards the raw args string to the tool (no core parsing).

    The tool is invoked with params:
    `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.

## Gating (load-time filters)

OpenClaw **filters skills at load time** using `metadata` (single-line JSON):

```markdown
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
```

Fields under `metadata.openclaw`:

- `always: true` — always include the skill (skip other gates).
- `emoji` — optional emoji used by the macOS Skills UI.
- `homepage` — optional URL shown as “Website” in the macOS Skills UI.
- `os` — optional list of platforms (`darwin`, `linux`, `win32`). If set, the skill is only eligible on those OSes.
- `requires.bins` — list; each must exist on `PATH`.
- `requires.anyBins` — list; at least one must exist on `PATH`.
- `requires.env` — list; env var must exist **or** be provided in config.
- `requires.config` — list of `openclaw.json` paths that must be truthy.
- `primaryEnv` — env var name associated with `skills.entries.<name>.apiKey`.
- `install` — optional array of installer specs used by the macOS Skills UI (brew/node/go/uv/download).

Note on sandboxing:

- `requires.bins` is checked on the **host** at skill load time.
- If an agent is sandboxed, the binary must also exist **inside the container**.
  Install it via `agents.defaults.sandbox.docker.setupCommand` (or a custom image).
  `setupCommand` runs once after the container is created.
  Package installs also require network egress, a writable root FS, and a root user in the sandbox.
  Example: the `summarize` skill (`skills/summarize/SKILL.md`) needs the `summarize` CLI
  in the sandbox container to run there.

Installer example:

```markdown
name: gemini
description: Use Gemini CLI for coding assistance and Google search lookups.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Install Gemini CLI (brew)",
            },
          ],
      },
  }
```

Notes:

- If multiple installers are listed, the gateway picks a **single** preferred option (brew when available, otherwise node).
- If all installers are `download`, OpenClaw lists each entry so you can see the available artifacts.
- Installer specs can include `os: ["darwin"|"linux"|"win32"]` to filter options by platform.
- Node installs honor `skills.install.nodeManager` in `openclaw.json` (default: npm; options: npm/pnpm/yarn/bun).
  This only affects **skill installs**; the Gateway runtime should still be Node
  (Bun is not recommended for WhatsApp/Telegram).
- Go installs: if `go` is missing and `brew` is available, the gateway installs Go via Homebrew first and sets `GOBIN` to Homebrew’s `bin` when possible.
- Download installs: `url` (required), `archive` (`tar.gz` | `tar.bz2` | `zip`), `extract` (default: auto when archive detected), `stripComponents`, `targetDir` (default: `~/.openclaw/tools/<skillKey>`).

If no `metadata.openclaw` is present, the skill is always eligible (unless
disabled in config or blocked by `skills.allowBundled` for bundled skills).

## Config overrides (`~/.openclaw/openclaw.json`)

Bundled/managed skills can be toggled and supplied with env values:

```json5
{
  skills: {
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

Note: if the skill name contains hyphens, quote the key (JSON5 allows quoted keys).

Config keys match the **skill name** by default. If a skill defines
`metadata.openclaw.skillKey`, use that key under `skills.entries`.

Rules:

- `enabled: false` disables the skill even if it’s bundled/installed.
- `env`: injected **only if** the variable isn’t already set in the process.
- `apiKey`: convenience for skills that declare `metadata.openclaw.primaryEnv`.
  Supports plaintext string or SecretRef object (`{ source, provider, id }`).
- `config`: optional bag for custom per-skill fields; custom keys must live here.
- `allowBundled`: optional allowlist for **bundled** skills only. If set, only
  bundled skills in the list are eligible (managed/workspace skills unaffected).

## Environment injection (per agent run)

When an agent run starts, OpenClaw:

1. Reads skill metadata.
2. Applies any `skills.entries.<key>.env` or `skills.entries.<key>.apiKey` to
   `process.env`.
3. Builds the system prompt with **eligible** skills.
4. Restores the original environment after the run ends.

This is **scoped to the agent run**, not a global shell environment.

## Session snapshot (performance)

OpenClaw snapshots the eligible skills **when a session starts** and reuses that list for subsequent turns in the same session. Changes to skills or config take effect on the next new session.

Skills can also refresh mid-session when the skills watcher is enabled or when a new eligible remote node appears (see below). Think of this as a **hot reload**: the refreshed list is picked up on the next agent turn.

## Remote macOS nodes (Linux gateway)

If the Gateway is running on Linux but a **macOS node** is connected **with `system.run` allowed** (Exec approvals security not set to `deny`), OpenClaw can treat macOS-only skills as eligible when the required binaries are present on that node. The agent should execute those skills via the `nodes` tool (typically `nodes.run`).

This relies on the node reporting its command support and on a bin probe via `system.run`. If the macOS node goes offline later, the skills remain visible; invocations may fail until the node reconnects.

## Skills watcher (auto-refresh)

By default, OpenClaw watches skill folders and bumps the skills snapshot when `SKILL.md` files change. Configure this under `skills.load`:

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250,
    },
  },
}
```

## Token impact (skills list)

When skills are eligible, OpenClaw injects a compact XML list of available skills into the system prompt (via `formatSkillsForPrompt` in `pi-coding-agent`). The cost is deterministic:

- **Base overhead (only when ≥1 skill):** 195 characters.
- **Per skill:** 97 characters + the length of the XML-escaped `<name>`, `<description>`, and `<location>` values.

Formula (characters):

```
total = 195 + Σ (97 + len(name_escaped) + len(description_escaped) + len(location_escaped))
```

Notes:

- XML escaping expands `& < > " '` into entities (`&amp;`, `&lt;`, etc.), increasing length.
- Token counts vary by model tokenizer. A rough OpenAI-style estimate is ~4 chars/token, so **97 chars ≈ 24 tokens** per skill plus your actual field lengths.

## Managed skills lifecycle

OpenClaw ships a baseline set of skills as **bundled skills** as part of the
install (npm package or OpenClaw.app). `~/.openclaw/skills` exists for local
overrides (for example, pinning/patching a skill without changing the bundled
copy). Workspace skills are user-owned and override both on name conflicts.

## Config reference

See [Skills config](/tools/skills-config) for the full configuration schema.

## Looking for more skills?

Browse [https://clawhub.com](https://clawhub.com).


## Skills Configuration


# Skills Config

All skills-related configuration lives under `skills` in `~/.openclaw/openclaw.json`.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills", "~/Projects/oss/some-skill-pack/skills"],
      watch: true,
      watchDebounceMs: 250,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun (Gateway runtime still Node; bun not recommended)
    },
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // or plaintext string
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

## Fields

- `allowBundled`: optional allowlist for **bundled** skills only. When set, only
  bundled skills in the list are eligible (managed/workspace skills unaffected).
- `load.extraDirs`: additional skill directories to scan (lowest precedence).
- `load.watch`: watch skill folders and refresh the skills snapshot (default: true).
- `load.watchDebounceMs`: debounce for skill watcher events in milliseconds (default: 250).
- `install.preferBrew`: prefer brew installers when available (default: true).
- `install.nodeManager`: node installer preference (`npm` | `pnpm` | `yarn` | `bun`, default: npm).
  This only affects **skill installs**; the Gateway runtime should still be Node
  (Bun not recommended for WhatsApp/Telegram).
- `entries.<skillKey>`: per-skill overrides.

Per-skill fields:

- `enabled`: set `false` to disable a skill even if it’s bundled/installed.
- `env`: environment variables injected for the agent run (only if not already set).
- `apiKey`: optional convenience for skills that declare a primary env var.
  Supports plaintext string or SecretRef object (`{ source, provider, id }`).

## Notes

- Keys under `entries` map to the skill name by default. If a skill defines
  `metadata.openclaw.skillKey`, use that key instead.
- Changes to skills are picked up on the next agent turn when the watcher is enabled.

### Sandboxed skills + env vars

When a session is **sandboxed**, skill processes run inside Docker. The sandbox
does **not** inherit the host `process.env`.

Use one of:

- `agents.defaults.sandbox.docker.env` (or per-agent `agents.list[].sandbox.docker.env`)
- bake the env into your custom sandbox image

Global `env` and `skills.entries.<skill>.env/apiKey` apply to **host** runs only.

## ClawHub


# ClawHub

ClawHub is the **public skill registry for OpenClaw**. It is a free service: all skills are public, open, and visible to everyone for sharing and reuse. A skill is just a folder with a `SKILL.md` file (plus supporting text files). You can browse skills in the web app or use the CLI to search, install, update, and publish skills.

Site: [clawhub.ai](https://clawhub.ai)

## What ClawHub is

- A public registry for OpenClaw skills.
- A versioned store of skill bundles and metadata.
- A discovery surface for search, tags, and usage signals.

## How it works

1. A user publishes a skill bundle (files + metadata).
2. ClawHub stores the bundle, parses metadata, and assigns a version.
3. The registry indexes the skill for search and discovery.
4. Users browse, download, and install skills in OpenClaw.

## What you can do

- Publish new skills and new versions of existing skills.
- Discover skills by name, tags, or search.
- Download skill bundles and inspect their files.
- Report skills that are abusive or unsafe.
- If you are a moderator, hide, unhide, delete, or ban.

## Who this is for (beginner-friendly)

If you want to add new capabilities to your OpenClaw agent, ClawHub is the easiest way to find and install skills. You do not need to know how the backend works. You can:

- Search for skills by plain language.
- Install a skill into your workspace.
- Update skills later with one command.
- Back up your own skills by publishing them.

## Quick start (non-technical)

1. Install the CLI (see next section).
2. Search for something you need:
   - `clawhub search "calendar"`
3. Install a skill:
   - `clawhub install <skill-slug>`
4. Start a new OpenClaw session so it picks up the new skill.

## Install the CLI

Pick one:

```bash
npm i -g clawhub
```

```bash
pnpm add -g clawhub
```

## How it fits into OpenClaw

By default, the CLI installs skills into `./skills` under your current working directory. If a OpenClaw workspace is configured, `clawhub` falls back to that workspace unless you override `--workdir` (or `CLAWHUB_WORKDIR`). OpenClaw loads workspace skills from `<workspace>/skills` and will pick them up in the **next** session. If you already use `~/.openclaw/skills` or bundled skills, workspace skills take precedence.

For more detail on how skills are loaded, shared, and gated, see
[Skills](/tools/skills).

## Skill system overview

A skill is a versioned bundle of files that teaches OpenClaw how to perform a
specific task. Each publish creates a new version, and the registry keeps a
history of versions so users can audit changes.

A typical skill includes:

- A `SKILL.md` file with the primary description and usage.
- Optional configs, scripts, or supporting files used by the skill.
- Metadata such as tags, summary, and install requirements.

ClawHub uses metadata to power discovery and safely expose skill capabilities.
The registry also tracks usage signals (such as stars and downloads) to improve
ranking and visibility.

## What the service provides (features)

- **Public browsing** of skills and their `SKILL.md` content.
- **Search** powered by embeddings (vector search), not just keywords.
- **Versioning** with semver, changelogs, and tags (including `latest`).
- **Downloads** as a zip per version.
- **Stars and comments** for community feedback.
- **Moderation** hooks for approvals and audits.
- **CLI-friendly API** for automation and scripting.

## Security and moderation

ClawHub is open by default. Anyone can upload skills, but a GitHub account must
be at least one week old to publish. This helps slow down abuse without blocking
legitimate contributors.

Reporting and moderation:

- Any signed in user can report a skill.
- Report reasons are required and recorded.
- Each user can have up to 20 active reports at a time.
- Skills with more than 3 unique reports are auto hidden by default.
- Moderators can view hidden skills, unhide them, delete them, or ban users.
- Abusing the report feature can result in account bans.

Interested in becoming a moderator? Ask in the OpenClaw Discord and contact a
moderator or maintainer.

## CLI commands and parameters

Global options (apply to all commands):

- `--workdir <dir>`: Working directory (default: current dir; falls back to OpenClaw workspace).
- `--dir <dir>`: Skills directory, relative to workdir (default: `skills`).
- `--site <url>`: Site base URL (browser login).
- `--registry <url>`: Registry API base URL.
- `--no-input`: Disable prompts (non-interactive).
- `-V, --cli-version`: Print CLI version.

Auth:

- `clawhub login` (browser flow) or `clawhub login --token <token>`
- `clawhub logout`
- `clawhub whoami`

Options:

- `--token <token>`: Paste an API token.
- `--label <label>`: Label stored for browser login tokens (default: `CLI token`).
- `--no-browser`: Do not open a browser (requires `--token`).

Search:

- `clawhub search "query"`
- `--limit <n>`: Max results.

Install:

- `clawhub install <slug>`
- `--version <version>`: Install a specific version.
- `--force`: Overwrite if the folder already exists.

Update:

- `clawhub update <slug>`
- `clawhub update --all`
- `--version <version>`: Update to a specific version (single slug only).
- `--force`: Overwrite when local files do not match any published version.

List:

- `clawhub list` (reads `.clawhub/lock.json`)

Publish:

- `clawhub publish <path>`
- `--slug <slug>`: Skill slug.
- `--name <name>`: Display name.
- `--version <version>`: Semver version.
- `--changelog <text>`: Changelog text (can be empty).
- `--tags <tags>`: Comma-separated tags (default: `latest`).

Delete/undelete (owner/admin only):

- `clawhub delete <slug> --yes`
- `clawhub undelete <slug> --yes`

Sync (scan local skills + publish new/updated):

- `clawhub sync`
- `--root <dir...>`: Extra scan roots.
- `--all`: Upload everything without prompts.
- `--dry-run`: Show what would be uploaded.
- `--bump <type>`: `patch|minor|major` for updates (default: `patch`).
- `--changelog <text>`: Changelog for non-interactive updates.
- `--tags <tags>`: Comma-separated tags (default: `latest`).
- `--concurrency <n>`: Registry checks (default: 4).

## Common workflows for agents

### Search for skills

```bash
clawhub search "postgres backups"
```

### Download new skills

```bash
clawhub install my-skill-pack
```

### Update installed skills

```bash
clawhub update --all
```

### Back up your skills (publish or sync)

For a single skill folder:

```bash
clawhub publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0 --tags latest
```

To scan and back up many skills at once:

```bash
clawhub sync --all
```

## Advanced details (technical)

### Versioning and tags

- Each publish creates a new **semver** `SkillVersion`.
- Tags (like `latest`) point to a version; moving tags lets you roll back.
- Changelogs are attached per version and can be empty when syncing or publishing updates.

### Local changes vs registry versions

Updates compare the local skill contents to registry versions using a content hash. If local files do not match any published version, the CLI asks before overwriting (or requires `--force` in non-interactive runs).

### Sync scanning and fallback roots

`clawhub sync` scans your current workdir first. If no skills are found, it falls back to known legacy locations (for example `~/openclaw/skills` and `~/.openclaw/skills`). This is designed to find older skill installs without extra flags.

### Storage and lockfile

- Installed skills are recorded in `.clawhub/lock.json` under your workdir.
- Auth tokens are stored in the ClawHub CLI config file (override via `CLAWHUB_CONFIG_PATH`).

### Telemetry (install counts)

When you run `clawhub sync` while logged in, the CLI sends a minimal snapshot to compute install counts. You can disable this entirely:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

## Environment variables

- `CLAWHUB_SITE`: Override the site URL.
- `CLAWHUB_REGISTRY`: Override the registry API URL.
- `CLAWHUB_CONFIG_PATH`: Override where the CLI stores the token/config.
- `CLAWHUB_WORKDIR`: Override the default workdir.
- `CLAWHUB_DISABLE_TELEMETRY=1`: Disable telemetry on `sync`.

## Exec Tool


# Exec tool

Run shell commands in the workspace. Supports foreground + background execution via `process`.
If `process` is disallowed, `exec` runs synchronously and ignores `yieldMs`/`background`.
Background sessions are scoped per agent; `process` only sees sessions from the same agent.

## Parameters

- `command` (required)
- `workdir` (defaults to cwd)
- `env` (key/value overrides)
- `yieldMs` (default 10000): auto-background after delay
- `background` (bool): background immediately
- `timeout` (seconds, default 1800): kill on expiry
- `pty` (bool): run in a pseudo-terminal when available (TTY-only CLIs, coding agents, terminal UIs)
- `host` (`sandbox | gateway | node`): where to execute
- `security` (`deny | allowlist | full`): enforcement mode for `gateway`/`node`
- `ask` (`off | on-miss | always`): approval prompts for `gateway`/`node`
- `node` (string): node id/name for `host=node`
- `elevated` (bool): request elevated mode (gateway host); `security=full` is only forced when elevated resolves to `full`

Notes:

- `host` defaults to `sandbox`.
- `elevated` is ignored when sandboxing is off (exec already runs on the host).
- `gateway`/`node` approvals are controlled by `~/.openclaw/exec-approvals.json`.
- `node` requires a paired node (companion app or headless node host).
- If multiple nodes are available, set `exec.node` or `tools.exec.node` to select one.
- On non-Windows hosts, exec uses `SHELL` when set; if `SHELL` is `fish`, it prefers `bash` (or `sh`)
  from `PATH` to avoid fish-incompatible scripts, then falls back to `SHELL` if neither exists.
- On Windows hosts, exec prefers PowerShell 7 (`pwsh`) discovery (Program Files, ProgramW6432, then PATH),
  then falls back to Windows PowerShell 5.1.
- Host execution (`gateway`/`node`) rejects `env.PATH` and loader overrides (`LD_*`/`DYLD_*`) to
  prevent binary hijacking or injected code.
- Important: sandboxing is **off by default**. If sandboxing is off and `host=sandbox` is explicitly
  configured/requested, exec now fails closed instead of silently running on the gateway host.
  Enable sandboxing or use `host=gateway` with approvals.
- Script preflight checks (for common Python/Node shell-syntax mistakes) only inspect files inside the
  effective `workdir` boundary. If a script path resolves outside `workdir`, preflight is skipped for
  that file.

## Config

- `tools.exec.notifyOnExit` (default: true): when true, backgrounded exec sessions enqueue a system event and request a heartbeat on exit.
- `tools.exec.approvalRunningNoticeMs` (default: 10000): emit a single “running” notice when an approval-gated exec runs longer than this (0 disables).
- `tools.exec.host` (default: `sandbox`)
- `tools.exec.security` (default: `deny` for sandbox, `allowlist` for gateway + node when unset)
- `tools.exec.ask` (default: `on-miss`)
- `tools.exec.node` (default: unset)
- `tools.exec.pathPrepend`: list of directories to prepend to `PATH` for exec runs (gateway + sandbox only).
- `tools.exec.safeBins`: stdin-only safe binaries that can run without explicit allowlist entries. For behavior details, see [Safe bins](/tools/exec-approvals#safe-bins-stdin-only).
- `tools.exec.safeBinTrustedDirs`: additional explicit directories trusted for `safeBins` path checks. `PATH` entries are never auto-trusted. Built-in defaults are `/bin` and `/usr/bin`.
- `tools.exec.safeBinProfiles`: optional custom argv policy per safe bin (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`).

Example:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### PATH handling

- `host=gateway`: merges your login-shell `PATH` into the exec environment. `env.PATH` overrides are
  rejected for host execution. The daemon itself still runs with a minimal `PATH`:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
- `host=sandbox`: runs `sh -lc` (login shell) inside the container, so `/etc/profile` may reset `PATH`.
  OpenClaw prepends `env.PATH` after profile sourcing via an internal env var (no shell interpolation);
  `tools.exec.pathPrepend` applies here too.
- `host=node`: only non-blocked env overrides you pass are sent to the node. `env.PATH` overrides are
  rejected for host execution and ignored by node hosts. If you need additional PATH entries on a node,
  configure the node host service environment (systemd/launchd) or install tools in standard locations.

Per-agent node binding (use the agent list index in config):

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Control UI: the Nodes tab includes a small “Exec node binding” panel for the same settings.

## Session overrides (`/exec`)

Use `/exec` to set **per-session** defaults for `host`, `security`, `ask`, and `node`.
Send `/exec` with no arguments to show the current values.

Example:

```
/exec host=gateway security=allowlist ask=on-miss node=mac-1
```

## Authorization model

`/exec` is only honored for **authorized senders** (channel allowlists/pairing plus `commands.useAccessGroups`).
It updates **session state only** and does not write config. To hard-disable exec, deny it via tool
policy (`tools.deny: ["exec"]` or per-agent). Host approvals still apply unless you explicitly set
`security=full` and `ask=off`.

## Exec approvals (companion app / node host)

Sandboxed agents can require per-request approval before `exec` runs on the gateway or node host.
See [Exec approvals](/tools/exec-approvals) for the policy, allowlist, and UI flow.

When approvals are required, the exec tool returns immediately with
`status: "approval-pending"` and an approval id. Once approved (or denied / timed out),
the Gateway emits system events (`Exec finished` / `Exec denied`). If the command is still
running after `tools.exec.approvalRunningNoticeMs`, a single `Exec running` notice is emitted.

## Allowlist + safe bins

Manual allowlist enforcement matches **resolved binary paths only** (no basename matches). When
`security=allowlist`, shell commands are auto-allowed only if every pipeline segment is
allowlisted or a safe bin. Chaining (`;`, `&&`, `||`) and redirections are rejected in
allowlist mode unless every top-level segment satisfies the allowlist (including safe bins).
Redirections remain unsupported.

`autoAllowSkills` is a separate convenience path in exec approvals. It is not the same as
manual path allowlist entries. For strict explicit trust, keep `autoAllowSkills` disabled.

Use the two controls for different jobs:

- `tools.exec.safeBins`: small, stdin-only stream filters.
- `tools.exec.safeBinTrustedDirs`: explicit extra trusted directories for safe-bin executable paths.
- `tools.exec.safeBinProfiles`: explicit argv policy for custom safe bins.
- allowlist: explicit trust for executable paths.

Do not treat `safeBins` as a generic allowlist, and do not add interpreter/runtime binaries (for example `python3`, `node`, `ruby`, `bash`). If you need those, use explicit allowlist entries and keep approval prompts enabled.
`openclaw security audit` warns when interpreter/runtime `safeBins` entries are missing explicit profiles, and `openclaw doctor --fix` can scaffold missing custom `safeBinProfiles` entries.

For full policy details and examples, see [Exec approvals](/tools/exec-approvals#safe-bins-stdin-only) and [Safe bins versus allowlist](/tools/exec-approvals#safe-bins-versus-allowlist).

## Examples

Foreground:

```json
{ "tool": "exec", "command": "ls -la" }
```

Background + poll:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

Send keys (tmux-style):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Submit (send CR only):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Paste (bracketed by default):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch (experimental)

`apply_patch` is a subtool of `exec` for structured multi-file edits.
Enable it explicitly:

```json5
{
  tools: {
    exec: {
      applyPatch: { enabled: true, workspaceOnly: true, allowModels: ["gpt-5.2"] },
    },
  },
}
```

Notes:

- Only available for OpenAI/OpenAI Codex models.
- Tool policy still applies; `allow: ["exec"]` implicitly allows `apply_patch`.
- Config lives under `tools.exec.applyPatch`.
- `tools.exec.applyPatch.workspaceOnly` defaults to `true` (workspace-contained). Set it to `false` only if you intentionally want `apply_patch` to write/delete outside the workspace directory.

## Browser Tool


# Browser (openclaw-managed)

OpenClaw can run a **dedicated Chrome/Brave/Edge/Chromium profile** that the agent controls.
It is isolated from your personal browser and is managed through a small local
control service inside the Gateway (loopback only).

Beginner view:

- Think of it as a **separate, agent-only browser**.
- The `openclaw` profile does **not** touch your personal browser profile.
- The agent can **open tabs, read pages, click, and type** in a safe lane.
- The default `chrome` profile uses the **system default Chromium browser** via the
  extension relay; switch to `openclaw` for the isolated managed browser.

## What you get

- A separate browser profile named **openclaw** (orange accent by default).
- Deterministic tab control (list/open/focus/close).
- Agent actions (click/type/drag/select), snapshots, screenshots, PDFs.
- Optional multi-profile support (`openclaw`, `work`, `remote`, ...).

This browser is **not** your daily driver. It is a safe, isolated surface for
agent automation and verification.

## Quick start

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

If you get “Browser disabled”, enable it in config (see below) and restart the
Gateway.

## Profiles: `openclaw` vs `chrome`

- `openclaw`: managed, isolated browser (no extension required).
- `chrome`: extension relay to your **system browser** (requires the OpenClaw
  extension to be attached to a tab).

Set `browser.defaultProfile: "openclaw"` if you want managed mode by default.

## Configuration

Browser settings live in `~/.openclaw/openclaw.json`.

```json5
{
  browser: {
    enabled: true, // default: true
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // default trusted-network mode
      // allowPrivateNetwork: true, // legacy alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // legacy single-profile override
    remoteCdpTimeoutMs: 1500, // remote CDP HTTP timeout (ms)
    remoteCdpHandshakeTimeoutMs: 3000, // remote CDP WebSocket handshake timeout (ms)
    defaultProfile: "chrome",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

Notes:

- The browser control service binds to loopback on a port derived from `gateway.port`
  (default: `18791`, which is gateway + 2). The relay uses the next port (`18792`).
- If you override the Gateway port (`gateway.port` or `OPENCLAW_GATEWAY_PORT`),
  the derived browser ports shift to stay in the same “family”.
- `cdpUrl` defaults to the relay port when unset.
- `remoteCdpTimeoutMs` applies to remote (non-loopback) CDP reachability checks.
- `remoteCdpHandshakeTimeoutMs` applies to remote CDP WebSocket reachability checks.
- Browser navigation/open-tab is SSRF-guarded before navigation and best-effort re-checked on final `http(s)` URL after navigation.
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` defaults to `true` (trusted-network model). Set it to `false` for strict public-only browsing.
- `browser.ssrfPolicy.allowPrivateNetwork` remains supported as a legacy alias for compatibility.
- `attachOnly: true` means “never launch a local browser; only attach if it is already running.”
- `color` + per-profile `color` tint the browser UI so you can see which profile is active.
- Default profile is `chrome` (extension relay). Use `defaultProfile: "openclaw"` for the managed browser.
- Auto-detect order: system default browser if Chromium-based; otherwise Chrome → Brave → Edge → Chromium → Chrome Canary.
- Local `openclaw` profiles auto-assign `cdpPort`/`cdpUrl` — set those only for remote CDP.

## Use Brave (or another Chromium-based browser)

If your **system default** browser is Chromium-based (Chrome/Brave/Edge/etc),
OpenClaw uses it automatically. Set `browser.executablePath` to override
auto-detection:

CLI example:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## Local vs remote control

- **Local control (default):** the Gateway starts the loopback control service and can launch a local browser.
- **Remote control (node host):** run a node host on the machine that has the browser; the Gateway proxies browser actions to it.
- **Remote CDP:** set `browser.profiles.<name>.cdpUrl` (or `browser.cdpUrl`) to
  attach to a remote Chromium-based browser. In this case, OpenClaw will not launch a local browser.

Remote CDP URLs can include auth:

- Query tokens (e.g., `https://provider.example?token=<token>`)
- HTTP Basic auth (e.g., `https://user:pass@provider.example`)

OpenClaw preserves the auth when calling `/json/*` endpoints and when connecting
to the CDP WebSocket. Prefer environment variables or secrets managers for
tokens instead of committing them to config files.

## Node browser proxy (zero-config default)

If you run a **node host** on the machine that has your browser, OpenClaw can
auto-route browser tool calls to that node without any extra browser config.
This is the default path for remote gateways.

Notes:

- The node host exposes its local browser control server via a **proxy command**.
- Profiles come from the node’s own `browser.profiles` config (same as local).
- Disable if you don’t want it:
  - On the node: `nodeHost.browserProxy.enabled=false`
  - On the gateway: `gateway.nodes.browser.mode="off"`

## Browserless (hosted remote CDP)

[Browserless](https://browserless.io) is a hosted Chromium service that exposes
CDP endpoints over HTTPS. You can point a OpenClaw browser profile at a
Browserless region endpoint and authenticate with your API key.

Example:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "https://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

Notes:

- Replace `<BROWSERLESS_API_KEY>` with your real Browserless token.
- Choose the region endpoint that matches your Browserless account (see their docs).

## Security

Key ideas:

- Browser control is loopback-only; access flows through the Gateway’s auth or node pairing.
- If browser control is enabled and no auth is configured, OpenClaw auto-generates `gateway.auth.token` on startup and persists it to config.
- Keep the Gateway and any node hosts on a private network (Tailscale); avoid public exposure.
- Treat remote CDP URLs/tokens as secrets; prefer env vars or a secrets manager.

Remote CDP tips:

- Prefer HTTPS endpoints and short-lived tokens where possible.
- Avoid embedding long-lived tokens directly in config files.

## Profiles (multi-browser)

OpenClaw supports multiple named profiles (routing configs). Profiles can be:

- **openclaw-managed**: a dedicated Chromium-based browser instance with its own user data directory + CDP port
- **remote**: an explicit CDP URL (Chromium-based browser running elsewhere)
- **extension relay**: your existing Chrome tab(s) via the local relay + Chrome extension

Defaults:

- The `openclaw` profile is auto-created if missing.
- The `chrome` profile is built-in for the Chrome extension relay (points at `http://127.0.0.1:18792` by default).
- Local CDP ports allocate from **18800–18899** by default.
- Deleting a profile moves its local data directory to Trash.

All control endpoints accept `?profile=<name>`; the CLI uses `--browser-profile`.

## Chrome extension relay (use your existing Chrome)

OpenClaw can also drive **your existing Chrome tabs** (no separate “openclaw” Chrome instance) via a local CDP relay + a Chrome extension.

Full guide: [Chrome extension](/tools/chrome-extension)

Flow:

- The Gateway runs locally (same machine) or a node host runs on the browser machine.
- A local **relay server** listens at a loopback `cdpUrl` (default: `http://127.0.0.1:18792`).
- You click the **OpenClaw Browser Relay** extension icon on a tab to attach (it does not auto-attach).
- The agent controls that tab via the normal `browser` tool, by selecting the right profile.

If the Gateway runs elsewhere, run a node host on the browser machine so the Gateway can proxy browser actions.

### Sandboxed sessions

If the agent session is sandboxed, the `browser` tool may default to `target="sandbox"` (sandbox browser).
Chrome extension relay takeover requires host browser control, so either:

- run the session unsandboxed, or
- set `agents.defaults.sandbox.browser.allowHostControl: true` and use `target="host"` when calling the tool.

### Setup

1. Load the extension (dev/unpacked):

```bash
openclaw browser extension install
```

- Chrome → `chrome://extensions` → enable “Developer mode”
- “Load unpacked” → select the directory printed by `openclaw browser extension path`
- Pin the extension, then click it on the tab you want to control (badge shows `ON`).

2. Use it:

- CLI: `openclaw browser --browser-profile chrome tabs`
- Agent tool: `browser` with `profile="chrome"`

Optional: if you want a different name or relay port, create your own profile:

```bash
openclaw browser create-profile \
  --name my-chrome \
  --driver extension \
  --cdp-url http://127.0.0.1:18792 \
  --color "#00AA00"
```

Notes:

- This mode relies on Playwright-on-CDP for most operations (screenshots/snapshots/actions).
- Detach by clicking the extension icon again.

## Isolation guarantees

- **Dedicated user data dir**: never touches your personal browser profile.
- **Dedicated ports**: avoids `9222` to prevent collisions with dev workflows.
- **Deterministic tab control**: target tabs by `targetId`, not “last tab”.

## Browser selection

When launching locally, OpenClaw picks the first available:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

You can override with `browser.executablePath`.

Platforms:

- macOS: checks `/Applications` and `~/Applications`.
- Linux: looks for `google-chrome`, `brave`, `microsoft-edge`, `chromium`, etc.
- Windows: checks common install locations.

## Control API (optional)

For local integrations only, the Gateway exposes a small loopback HTTP API:

- Status/start/stop: `GET /`, `POST /start`, `POST /stop`
- Tabs: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`
- Snapshot/screenshot: `GET /snapshot`, `POST /screenshot`
- Actions: `POST /navigate`, `POST /act`
- Hooks: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Downloads: `POST /download`, `POST /wait/download`
- Debugging: `GET /console`, `POST /pdf`
- Debugging: `GET /errors`, `GET /requests`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Network: `POST /response/body`
- State: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- State: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Settings: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

All endpoints accept `?profile=<name>`.

If gateway auth is configured, browser HTTP routes require auth too:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` or HTTP Basic auth with that password

### Playwright requirement

Some features (navigate/act/AI snapshot/role snapshot, element screenshots, PDF) require
Playwright. If Playwright isn’t installed, those endpoints return a clear 501
error. ARIA snapshots and basic screenshots still work for openclaw-managed Chrome.
For the Chrome extension relay driver, ARIA snapshots and screenshots require Playwright.

If you see `Playwright is not available in this gateway build`, install the full
Playwright package (not `playwright-core`) and restart the gateway, or reinstall
OpenClaw with browser support.

#### Docker Playwright install

If your Gateway runs in Docker, avoid `npx playwright` (npm override conflicts).
Use the bundled CLI instead:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

To persist browser downloads, set `PLAYWRIGHT_BROWSERS_PATH` (for example,
`/home/node/.cache/ms-playwright`) and make sure `/home/node` is persisted via
`OPENCLAW_HOME_VOLUME` or a bind mount. See [Docker](/install/docker).

## How it works (internal)

High-level flow:

- A small **control server** accepts HTTP requests.
- It connects to Chromium-based browsers (Chrome/Brave/Edge/Chromium) via **CDP**.
- For advanced actions (click/type/snapshot/PDF), it uses **Playwright** on top
  of CDP.
- When Playwright is missing, only non-Playwright operations are available.

This design keeps the agent on a stable, deterministic interface while letting
you swap local/remote browsers and profiles.

## CLI quick reference

All commands accept `--browser-profile <name>` to target a specific profile.
All commands also accept `--json` for machine-readable output (stable payloads).

Basics:

- `openclaw browser status`
- `openclaw browser start`
- `openclaw browser stop`
- `openclaw browser tabs`
- `openclaw browser tab`
- `openclaw browser tab new`
- `openclaw browser tab select 2`
- `openclaw browser tab close 2`
- `openclaw browser open https://example.com`
- `openclaw browser focus abcd1234`
- `openclaw browser close abcd1234`

Inspection:

- `openclaw browser screenshot`
- `openclaw browser screenshot --full-page`
- `openclaw browser screenshot --ref 12`
- `openclaw browser screenshot --ref e12`
- `openclaw browser snapshot`
- `openclaw browser snapshot --format aria --limit 200`
- `openclaw browser snapshot --interactive --compact --depth 6`
- `openclaw browser snapshot --efficient`
- `openclaw browser snapshot --labels`
- `openclaw browser snapshot --selector "#main" --interactive`
- `openclaw browser snapshot --frame "iframe#main" --interactive`
- `openclaw browser console --level error`
- `openclaw browser errors --clear`
- `openclaw browser requests --filter api --clear`
- `openclaw browser pdf`
- `openclaw browser responsebody "**/api" --max-chars 5000`

Actions:

- `openclaw browser navigate https://example.com`
- `openclaw browser resize 1280 720`
- `openclaw browser click 12 --double`
- `openclaw browser click e12 --double`
- `openclaw browser type 23 "hello" --submit`
- `openclaw browser press Enter`
- `openclaw browser hover 44`
- `openclaw browser scrollintoview e12`
- `openclaw browser drag 10 11`
- `openclaw browser select 9 OptionA OptionB`
- `openclaw browser download e12 report.pdf`
- `openclaw browser waitfordownload report.pdf`
- `openclaw browser upload /tmp/openclaw/uploads/file.pdf`
- `openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `openclaw browser dialog --accept`
- `openclaw browser wait --text "Done"`
- `openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `openclaw browser evaluate --fn '(el) => el.textContent' --ref 7`
- `openclaw browser highlight e12`
- `openclaw browser trace start`
- `openclaw browser trace stop`

State:

- `openclaw browser cookies`
- `openclaw browser cookies set session abc123 --url "https://example.com"`
- `openclaw browser cookies clear`
- `openclaw browser storage local get`
- `openclaw browser storage local set theme dark`
- `openclaw browser storage session clear`
- `openclaw browser set offline on`
- `openclaw browser set headers --headers-json '{"X-Debug":"1"}'`
- `openclaw browser set credentials user pass`
- `openclaw browser set credentials --clear`
- `openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `openclaw browser set geo --clear`
- `openclaw browser set media dark`
- `openclaw browser set timezone America/New_York`
- `openclaw browser set locale en-US`
- `openclaw browser set device "iPhone 14"`

Notes:

- `upload` and `dialog` are **arming** calls; run them before the click/press
  that triggers the chooser/dialog.
- Download and trace output paths are constrained to OpenClaw temp roots:
  - traces: `/tmp/openclaw` (fallback: `${os.tmpdir()}/openclaw`)
  - downloads: `/tmp/openclaw/downloads` (fallback: `${os.tmpdir()}/openclaw/downloads`)
- Upload paths are constrained to an OpenClaw temp uploads root:
  - uploads: `/tmp/openclaw/uploads` (fallback: `${os.tmpdir()}/openclaw/uploads`)
- `upload` can also set file inputs directly via `--input-ref` or `--element`.
- `snapshot`:
  - `--format ai` (default when Playwright is installed): returns an AI snapshot with numeric refs (`aria-ref="<n>"`).
  - `--format aria`: returns the accessibility tree (no refs; inspection only).
  - `--efficient` (or `--mode efficient`): compact role snapshot preset (interactive + compact + depth + lower maxChars).
  - Config default (tool/CLI only): set `browser.snapshotDefaults.mode: "efficient"` to use efficient snapshots when the caller does not pass a mode (see [Gateway configuration](/gateway/configuration#browser-openclaw-managed-browser)).
  - Role snapshot options (`--interactive`, `--compact`, `--depth`, `--selector`) force a role-based snapshot with refs like `ref=e12`.
  - `--frame "<iframe selector>"` scopes role snapshots to an iframe (pairs with role refs like `e12`).
  - `--interactive` outputs a flat, easy-to-pick list of interactive elements (best for driving actions).
  - `--labels` adds a viewport-only screenshot with overlayed ref labels (prints `MEDIA:<path>`).
- `click`/`type`/etc require a `ref` from `snapshot` (either numeric `12` or role ref `e12`).
  CSS selectors are intentionally not supported for actions.

## Snapshots and refs

OpenClaw supports two “snapshot” styles:

- **AI snapshot (numeric refs)**: `openclaw browser snapshot` (default; `--format ai`)
  - Output: a text snapshot that includes numeric refs.
  - Actions: `openclaw browser click 12`, `openclaw browser type 23 "hello"`.
  - Internally, the ref is resolved via Playwright’s `aria-ref`.

- **Role snapshot (role refs like `e12`)**: `openclaw browser snapshot --interactive` (or `--compact`, `--depth`, `--selector`, `--frame`)
  - Output: a role-based list/tree with `[ref=e12]` (and optional `[nth=1]`).
  - Actions: `openclaw browser click e12`, `openclaw browser highlight e12`.
  - Internally, the ref is resolved via `getByRole(...)` (plus `nth()` for duplicates).
  - Add `--labels` to include a viewport screenshot with overlayed `e12` labels.

Ref behavior:

- Refs are **not stable across navigations**; if something fails, re-run `snapshot` and use a fresh ref.
- If the role snapshot was taken with `--frame`, role refs are scoped to that iframe until the next role snapshot.

## Wait power-ups

You can wait on more than just time/text:

- Wait for URL (globs supported by Playwright):
  - `openclaw browser wait --url "**/dash"`
- Wait for load state:
  - `openclaw browser wait --load networkidle`
- Wait for a JS predicate:
  - `openclaw browser wait --fn "window.ready===true"`
- Wait for a selector to become visible:
  - `openclaw browser wait "#main"`

These can be combined:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## Debug workflows

When an action fails (e.g. “not visible”, “strict mode violation”, “covered”):

1. `openclaw browser snapshot --interactive`
2. Use `click <ref>` / `type <ref>` (prefer role refs in interactive mode)
3. If it still fails: `openclaw browser highlight <ref>` to see what Playwright is targeting
4. If the page behaves oddly:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. For deep debugging: record a trace:
   - `openclaw browser trace start`
   - reproduce the issue
   - `openclaw browser trace stop` (prints `TRACE:<path>`)

## JSON output

`--json` is for scripting and structured tooling.

Examples:

```bash
openclaw browser status --json
openclaw browser snapshot --interactive --json
openclaw browser requests --filter api --json
openclaw browser cookies --json
```

Role snapshots in JSON include `refs` plus a small `stats` block (lines/chars/refs/interactive) so tools can reason about payload size and density.

## State and environment knobs

These are useful for “make the site behave like X” workflows:

- Cookies: `cookies`, `cookies set`, `cookies clear`
- Storage: `storage local|session get|set|clear`
- Offline: `set offline on|off`
- Headers: `set headers --headers-json '{"X-Debug":"1"}'` (legacy `set headers --json '{"X-Debug":"1"}'` remains supported)
- HTTP basic auth: `set credentials user pass` (or `--clear`)
- Geolocation: `set geo <lat> <lon> --origin "https://example.com"` (or `--clear`)
- Media: `set media dark|light|no-preference|none`
- Timezone / locale: `set timezone ...`, `set locale ...`
- Device / viewport:
  - `set device "iPhone 14"` (Playwright device presets)
  - `set viewport 1280 720`

## Security & privacy

- The openclaw browser profile may contain logged-in sessions; treat it as sensitive.
- `browser act kind=evaluate` / `openclaw browser evaluate` and `wait --fn`
  execute arbitrary JavaScript in the page context. Prompt injection can steer
  this. Disable it with `browser.evaluateEnabled=false` if you do not need it.
- For logins and anti-bot notes (X/Twitter, etc.), see [Browser login + X/Twitter posting](/tools/browser-login).
- Keep the Gateway/node host private (loopback or tailnet-only).
- Remote CDP endpoints are powerful; tunnel and protect them.

Strict-mode example (block private/internal destinations by default):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // optional exact allow
    },
  },
}
```

## Troubleshooting

For Linux-specific issues (especially snap Chromium), see
[Browser troubleshooting](/tools/browser-linux-troubleshooting).

## Agent tools + how control works

The agent gets **one tool** for browser automation:

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

How it maps:

- `browser snapshot` returns a stable UI tree (AI or ARIA).
- `browser act` uses the snapshot `ref` IDs to click/type/drag/select.
- `browser screenshot` captures pixels (full page or element).
- `browser` accepts:
  - `profile` to choose a named browser profile (openclaw, chrome, or remote CDP).
  - `target` (`sandbox` | `host` | `node`) to select where the browser lives.
  - In sandboxed sessions, `target: "host"` requires `agents.defaults.sandbox.browser.allowHostControl=true`.
  - If `target` is omitted: sandboxed sessions default to `sandbox`, non-sandbox sessions default to `host`.
  - If a browser-capable node is connected, the tool may auto-route to it unless you pin `target="host"` or `target="node"`.

This keeps the agent deterministic and avoids brittle selectors.

## Subagents


# Sub-agents

Sub-agents are background agent runs spawned from an existing agent run. They run in their own session (`agent:<agentId>:subagent:<uuid>`) and, when finished, **announce** their result back to the requester chat channel.

## Slash command

Use `/subagents` to inspect or control sub-agent runs for the **current session**:

- `/subagents list`
- `/subagents kill <id|#|all>`
- `/subagents log <id|#> [limit] [tools]`
- `/subagents info <id|#>`
- `/subagents send <id|#> <message>`
- `/subagents steer <id|#> <message>`
- `/subagents spawn <agentId> <task> [--model <model>] [--thinking <level>]`

Thread binding controls:

These commands work on channels that support persistent thread bindings. See **Thread supporting channels** below.

- `/focus <subagent-label|session-key|session-id|session-label>`
- `/unfocus`
- `/agents`
- `/session ttl <duration|off>`

`/subagents info` shows run metadata (status, timestamps, session id, transcript path, cleanup).

### Spawn behavior

`/subagents spawn` starts a background sub-agent as a user command, not an internal relay, and it sends one final completion update back to the requester chat when the run finishes.

- The spawn command is non-blocking; it returns a run id immediately.
- On completion, the sub-agent announces a summary/result message back to the requester chat channel.
- For manual spawns, delivery is resilient:
  - OpenClaw tries direct `agent` delivery first with a stable idempotency key.
  - If direct delivery fails, it falls back to queue routing.
  - If queue routing is still not available, the announce is retried with a short exponential backoff before final give-up.
- The completion message is a system message and includes:
  - `Result` (`assistant` reply text, or latest `toolResult` if the assistant reply is empty)
  - `Status` (`completed successfully` / `failed` / `timed out`)
  - compact runtime/token stats
- `--model` and `--thinking` override defaults for that specific run.
- Use `info`/`log` to inspect details and output after completion.
- `/subagents spawn` is one-shot mode (`mode: "run"`). For persistent thread-bound sessions, use `sessions_spawn` with `thread: true` and `mode: "session"`.
- For ACP harness sessions (Codex, Claude Code, Gemini CLI), use `sessions_spawn` with `runtime: "acp"` and see [ACP Agents](/tools/acp-agents).

Primary goals:

- Parallelize "research / long task / slow tool" work without blocking the main run.
- Keep sub-agents isolated by default (session separation + optional sandboxing).
- Keep the tool surface hard to misuse: sub-agents do **not** get session tools by default.
- Support configurable nesting depth for orchestrator patterns.

Cost note: each sub-agent has its **own** context and token usage. For heavy or repetitive
tasks, set a cheaper model for sub-agents and keep your main agent on a higher-quality model.
You can configure this via `agents.defaults.subagents.model` or per-agent overrides.

## Tool

Use `sessions_spawn`:

- Starts a sub-agent run (`deliver: false`, global lane: `subagent`)
- Then runs an announce step and posts the announce reply to the requester chat channel
- Default model: inherits the caller unless you set `agents.defaults.subagents.model` (or per-agent `agents.list[].subagents.model`); an explicit `sessions_spawn.model` still wins.
- Default thinking: inherits the caller unless you set `agents.defaults.subagents.thinking` (or per-agent `agents.list[].subagents.thinking`); an explicit `sessions_spawn.thinking` still wins.
- Default run timeout: if `sessions_spawn.runTimeoutSeconds` is omitted, OpenClaw uses `agents.defaults.subagents.runTimeoutSeconds` when set; otherwise it falls back to `0` (no timeout).

Tool params:

- `task` (required)
- `label?` (optional)
- `agentId?` (optional; spawn under another agent id if allowed)
- `model?` (optional; overrides the sub-agent model; invalid values are skipped and the sub-agent runs on the default model with a warning in the tool result)
- `thinking?` (optional; overrides thinking level for the sub-agent run)
- `runTimeoutSeconds?` (defaults to `agents.defaults.subagents.runTimeoutSeconds` when set, otherwise `0`; when set, the sub-agent run is aborted after N seconds)
- `thread?` (default `false`; when `true`, requests channel thread binding for this sub-agent session)
- `mode?` (`run|session`)
  - default is `run`
  - if `thread: true` and `mode` omitted, default becomes `session`
  - `mode: "session"` requires `thread: true`
- `cleanup?` (`delete|keep`, default `keep`)

## Thread-bound sessions

When thread bindings are enabled for a channel, a sub-agent can stay bound to a thread so follow-up user messages in that thread keep routing to the same sub-agent session.

### Thread supporting channels

- Discord (currently the only supported channel): supports persistent thread-bound subagent sessions (`sessions_spawn` with `thread: true`), manual thread controls (`/focus`, `/unfocus`, `/agents`, `/session ttl`), and adapter keys `channels.discord.threadBindings.enabled`, `channels.discord.threadBindings.ttlHours`, and `channels.discord.threadBindings.spawnSubagentSessions`.

Quick flow:

1. Spawn with `sessions_spawn` using `thread: true` (and optionally `mode: "session"`).
2. OpenClaw creates or binds a thread to that session target in the active channel.
3. Replies and follow-up messages in that thread route to the bound session.
4. Use `/session ttl` to inspect/update auto-unfocus TTL.
5. Use `/unfocus` to detach manually.

Manual controls:

- `/focus <target>` binds the current thread (or creates one) to a sub-agent/session target.
- `/unfocus` removes the binding for the current bound thread.
- `/agents` lists active runs and binding state (`thread:<id>` or `unbound`).
- `/session ttl` only works for focused bound threads.

Config switches:

- Global default: `session.threadBindings.enabled`, `session.threadBindings.ttlHours`
- Channel override and spawn auto-bind keys are adapter-specific. See **Thread supporting channels** above.

See [Configuration Reference](/gateway/configuration-reference) and [Slash commands](/tools/slash-commands) for current adapter details.

Allowlist:

- `agents.list[].subagents.allowAgents`: list of agent ids that can be targeted via `agentId` (`["*"]` to allow any). Default: only the requester agent.

Discovery:

- Use `agents_list` to see which agent ids are currently allowed for `sessions_spawn`.

Auto-archive:

- Sub-agent sessions are automatically archived after `agents.defaults.subagents.archiveAfterMinutes` (default: 60).
- Archive uses `sessions.delete` and renames the transcript to `*.deleted.<timestamp>` (same folder).
- `cleanup: "delete"` archives immediately after announce (still keeps the transcript via rename).
- Auto-archive is best-effort; pending timers are lost if the gateway restarts.
- `runTimeoutSeconds` does **not** auto-archive; it only stops the run. The session remains until auto-archive.
- Auto-archive applies equally to depth-1 and depth-2 sessions.

## Nested Sub-Agents

By default, sub-agents cannot spawn their own sub-agents (`maxSpawnDepth: 1`). You can enable one level of nesting by setting `maxSpawnDepth: 2`, which allows the **orchestrator pattern**: main → orchestrator sub-agent → worker sub-sub-agents.

### How to enable

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2, // allow sub-agents to spawn children (default: 1)
        maxChildrenPerAgent: 5, // max active children per agent session (default: 5)
        maxConcurrent: 8, // global concurrency lane cap (default: 8)
        runTimeoutSeconds: 900, // default timeout for sessions_spawn when omitted (0 = no timeout)
      },
    },
  },
}
```

### Depth levels

| Depth | Session key shape                            | Role                                          | Can spawn?                   |
| ----- | -------------------------------------------- | --------------------------------------------- | ---------------------------- |
| 0     | `agent:<id>:main`                            | Main agent                                    | Always                       |
| 1     | `agent:<id>:subagent:<uuid>`                 | Sub-agent (orchestrator when depth 2 allowed) | Only if `maxSpawnDepth >= 2` |
| 2     | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | Sub-sub-agent (leaf worker)                   | Never                        |

### Announce chain

Results flow back up the chain:

1. Depth-2 worker finishes → announces to its parent (depth-1 orchestrator)
2. Depth-1 orchestrator receives the announce, synthesizes results, finishes → announces to main
3. Main agent receives the announce and delivers to the user

Each level only sees announces from its direct children.

### Tool policy by depth

- **Depth 1 (orchestrator, when `maxSpawnDepth >= 2`)**: Gets `sessions_spawn`, `subagents`, `sessions_list`, `sessions_history` so it can manage its children. Other session/system tools remain denied.
- **Depth 1 (leaf, when `maxSpawnDepth == 1`)**: No session tools (current default behavior).
- **Depth 2 (leaf worker)**: No session tools — `sessions_spawn` is always denied at depth 2. Cannot spawn further children.

### Per-agent spawn limit

Each agent session (at any depth) can have at most `maxChildrenPerAgent` (default: 5) active children at a time. This prevents runaway fan-out from a single orchestrator.

### Cascade stop

Stopping a depth-1 orchestrator automatically stops all its depth-2 children:

- `/stop` in the main chat stops all depth-1 agents and cascades to their depth-2 children.
- `/subagents kill <id>` stops a specific sub-agent and cascades to its children.
- `/subagents kill all` stops all sub-agents for the requester and cascades.

## Authentication

Sub-agent auth is resolved by **agent id**, not by session type:

- The sub-agent session key is `agent:<agentId>:subagent:<uuid>`.
- The auth store is loaded from that agent's `agentDir`.
- The main agent's auth profiles are merged in as a **fallback**; agent profiles override main profiles on conflicts.

Note: the merge is additive, so main profiles are always available as fallbacks. Fully isolated auth per agent is not supported yet.

## Announce

Sub-agents report back via an announce step:

- The announce step runs inside the sub-agent session (not the requester session).
- If the sub-agent replies exactly `ANNOUNCE_SKIP`, nothing is posted.
- Otherwise the announce reply is posted to the requester chat channel via a follow-up `agent` call (`deliver=true`).
- Announce replies preserve thread/topic routing when available on channel adapters.
- Announce messages are normalized to a stable template:
  - `Status:` derived from the run outcome (`success`, `error`, `timeout`, or `unknown`).
  - `Result:` the summary content from the announce step (or `(not available)` if missing).
  - `Notes:` error details and other useful context.
- `Status` is not inferred from model output; it comes from runtime outcome signals.

Announce payloads include a stats line at the end (even when wrapped):

- Runtime (e.g., `runtime 5m12s`)
- Token usage (input/output/total)
- Estimated cost when model pricing is configured (`models.providers.*.models[].cost`)
- `sessionKey`, `sessionId`, and transcript path (so the main agent can fetch history via `sessions_history` or inspect the file on disk)

## Tool Policy (sub-agent tools)

By default, sub-agents get **all tools except session tools** and system tools:

- `sessions_list`
- `sessions_history`
- `sessions_send`
- `sessions_spawn`

When `maxSpawnDepth >= 2`, depth-1 orchestrator sub-agents additionally receive `sessions_spawn`, `subagents`, `sessions_list`, and `sessions_history` so they can manage their children.

Override via config:

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1,
      },
    },
  },
  tools: {
    subagents: {
      tools: {
        // deny wins
        deny: ["gateway", "cron"],
        // if allow is set, it becomes allow-only (deny still wins)
        // allow: ["read", "exec", "process"]
      },
    },
  },
}
```

## Concurrency

Sub-agents use a dedicated in-process queue lane:

- Lane name: `subagent`
- Concurrency: `agents.defaults.subagents.maxConcurrent` (default `8`)

## Stopping

- Sending `/stop` in the requester chat aborts the requester session and stops any active sub-agent runs spawned from it, cascading to nested children.
- `/subagents kill <id>` stops a specific sub-agent and cascades to its children.

## Limitations

- Sub-agent announce is **best-effort**. If the gateway restarts, pending "announce back" work is lost.
- Sub-agents still share the same gateway process resources; treat `maxConcurrent` as a safety valve.
- `sessions_spawn` is always non-blocking: it returns `{ status: "accepted", runId, childSessionKey }` immediately.
- Sub-agent context only injects `AGENTS.md` + `TOOLS.md` (no `SOUL.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, or `BOOTSTRAP.md`).
- Maximum nesting depth is 5 (`maxSpawnDepth` range: 1–5). Depth 2 is recommended for most use cases.
- `maxChildrenPerAgent` caps active children per session (default: 5, range: 1–20).

## Slash Commands


# Slash commands

Commands are handled by the Gateway. Most commands must be sent as a **standalone** message that starts with `/`.
The host-only bash chat command uses `! <cmd>` (with `/bash <cmd>` as an alias).

There are two related systems:

- **Commands**: standalone `/...` messages.
- **Directives**: `/think`, `/verbose`, `/reasoning`, `/elevated`, `/exec`, `/model`, `/queue`.
  - Directives are stripped from the message before the model sees it.
  - In normal chat messages (not directive-only), they are treated as “inline hints” and do **not** persist session settings.
  - In directive-only messages (the message contains only directives), they persist to the session and reply with an acknowledgement.
  - Directives are only applied for **authorized senders**. If `commands.allowFrom` is set, it is the only
    allowlist used; otherwise authorization comes from channel allowlists/pairing plus `commands.useAccessGroups`.
    Unauthorized senders see directives treated as plain text.

There are also a few **inline shortcuts** (allowlisted/authorized senders only): `/help`, `/commands`, `/status`, `/whoami` (`/id`).
They run immediately, are stripped before the model sees the message, and the remaining text continues through the normal flow.

## Config

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    debug: false,
    restart: false,
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

- `commands.text` (default `true`) enables parsing `/...` in chat messages.
  - On surfaces without native commands (WhatsApp/WebChat/Signal/iMessage/Google Chat/MS Teams), text commands still work even if you set this to `false`.
- `commands.native` (default `"auto"`) registers native commands.
  - Auto: on for Discord/Telegram; off for Slack (until you add slash commands); ignored for providers without native support.
  - Set `channels.discord.commands.native`, `channels.telegram.commands.native`, or `channels.slack.commands.native` to override per provider (bool or `"auto"`).
  - `false` clears previously registered commands on Discord/Telegram at startup. Slack commands are managed in the Slack app and are not removed automatically.
- `commands.nativeSkills` (default `"auto"`) registers **skill** commands natively when supported.
  - Auto: on for Discord/Telegram; off for Slack (Slack requires creating a slash command per skill).
  - Set `channels.discord.commands.nativeSkills`, `channels.telegram.commands.nativeSkills`, or `channels.slack.commands.nativeSkills` to override per provider (bool or `"auto"`).
- `commands.bash` (default `false`) enables `! <cmd>` to run host shell commands (`/bash <cmd>` is an alias; requires `tools.elevated` allowlists).
- `commands.bashForegroundMs` (default `2000`) controls how long bash waits before switching to background mode (`0` backgrounds immediately).
- `commands.config` (default `false`) enables `/config` (reads/writes `openclaw.json`).
- `commands.debug` (default `false`) enables `/debug` (runtime-only overrides).
- `commands.allowFrom` (optional) sets a per-provider allowlist for command authorization. When configured, it is the
  only authorization source for commands and directives (channel allowlists/pairing and `commands.useAccessGroups`
  are ignored). Use `"*"` for a global default; provider-specific keys override it.
- `commands.useAccessGroups` (default `true`) enforces allowlists/policies for commands when `commands.allowFrom` is not set.

## Command list

Text + native (when enabled):

- `/help`
- `/commands`
- `/skill <name> [input]` (run a skill by name)
- `/status` (show current status; includes provider usage/quota for the current model provider when available)
- `/allowlist` (list/add/remove allowlist entries)
- `/approve <id> allow-once|allow-always|deny` (resolve exec approval prompts)
- `/context [list|detail|json]` (explain “context”; `detail` shows per-file + per-tool + per-skill + system prompt size)
- `/export-session [path]` (alias: `/export`) (export current session to HTML with full system prompt)
- `/whoami` (show your sender id; alias: `/id`)
- `/session ttl <duration|off>` (manage session-level settings, such as TTL)
- `/subagents list|kill|log|info|send|steer|spawn` (inspect, control, or spawn sub-agent runs for the current session)
- `/acp spawn|cancel|steer|close|status|set-mode|set|cwd|permissions|timeout|model|reset-options|doctor|install|sessions` (inspect and control ACP runtime sessions)
- `/agents` (list thread-bound agents for this session)
- `/focus <target>` (Discord: bind this thread, or a new thread, to a session/subagent target)
- `/unfocus` (Discord: remove the current thread binding)
- `/kill <id|#|all>` (immediately abort one or all running sub-agents for this session; no confirmation message)
- `/steer <id|#> <message>` (steer a running sub-agent immediately: in-run when possible, otherwise abort current work and restart on the steer message)
- `/tell <id|#> <message>` (alias for `/steer`)
- `/config show|get|set|unset` (persist config to disk, owner-only; requires `commands.config: true`)
- `/debug show|set|unset|reset` (runtime overrides, owner-only; requires `commands.debug: true`)
- `/usage off|tokens|full|cost` (per-response usage footer or local cost summary)
- `/tts off|always|inbound|tagged|status|provider|limit|summary|audio` (control TTS; see [/tts](/tts))
  - Discord: native command is `/voice` (Discord reserves `/tts`); text `/tts` still works.
- `/stop`
- `/restart`
- `/dock-telegram` (alias: `/dock_telegram`) (switch replies to Telegram)
- `/dock-discord` (alias: `/dock_discord`) (switch replies to Discord)
- `/dock-slack` (alias: `/dock_slack`) (switch replies to Slack)
- `/activation mention|always` (groups only)
- `/send on|off|inherit` (owner-only)
- `/reset` or `/new [model]` (optional model hint; remainder is passed through)
- `/think <off|minimal|low|medium|high|xhigh>` (dynamic choices by model/provider; aliases: `/thinking`, `/t`)
- `/verbose on|full|off` (alias: `/v`)
- `/reasoning on|off|stream` (alias: `/reason`; when on, sends a separate message prefixed `Reasoning:`; `stream` = Telegram draft only)
- `/elevated on|off|ask|full` (alias: `/elev`; `full` skips exec approvals)
- `/exec host=<sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>` (send `/exec` to show current)
- `/model <name>` (alias: `/models`; or `/<alias>` from `agents.defaults.models.*.alias`)
- `/queue <mode>` (plus options like `debounce:2s cap:25 drop:summarize`; send `/queue` to see current settings)
- `/bash <command>` (host-only; alias for `! <command>`; requires `commands.bash: true` + `tools.elevated` allowlists)

Text-only:

- `/compact [instructions]` (see [/concepts/compaction](/concepts/compaction))
- `! <command>` (host-only; one at a time; use `!poll` + `!stop` for long-running jobs)
- `!poll` (check output / status; accepts optional `sessionId`; `/bash poll` also works)
- `!stop` (stop the running bash job; accepts optional `sessionId`; `/bash stop` also works)

Notes:

- Commands accept an optional `:` between the command and args (e.g. `/think: high`, `/send: on`, `/help:`).
- `/new <model>` accepts a model alias, `provider/model`, or a provider name (fuzzy match); if no match, the text is treated as the message body.
- For full provider usage breakdown, use `openclaw status --usage`.
- `/allowlist add|remove` requires `commands.config=true` and honors channel `configWrites`.
- `/usage` controls the per-response usage footer; `/usage cost` prints a local cost summary from OpenClaw session logs.
- `/restart` is enabled by default; set `commands.restart: false` to disable it.
- Discord-only native command: `/vc join|leave|status` controls voice channels (requires `channels.discord.voice` and native commands; not available as text).
- Discord thread-binding commands (`/focus`, `/unfocus`, `/agents`, `/session ttl`) require effective thread bindings to be enabled (`session.threadBindings.enabled` and/or `channels.discord.threadBindings.enabled`).
- ACP command reference and runtime behavior: [ACP Agents](/tools/acp-agents).
- `/verbose` is meant for debugging and extra visibility; keep it **off** in normal use.
- Tool failure summaries are still shown when relevant, but detailed failure text is only included when `/verbose` is `on` or `full`.
- `/reasoning` (and `/verbose`) are risky in group settings: they may reveal internal reasoning or tool output you did not intend to expose. Prefer leaving them off, especially in group chats.
- **Fast path:** command-only messages from allowlisted senders are handled immediately (bypass queue + model).
- **Group mention gating:** command-only messages from allowlisted senders bypass mention requirements.
- **Inline shortcuts (allowlisted senders only):** certain commands also work when embedded in a normal message and are stripped before the model sees the remaining text.
  - Example: `hey /status` triggers a status reply, and the remaining text continues through the normal flow.
- Currently: `/help`, `/commands`, `/status`, `/whoami` (`/id`).
- Unauthorized command-only messages are silently ignored, and inline `/...` tokens are treated as plain text.
- **Skill commands:** `user-invocable` skills are exposed as slash commands. Names are sanitized to `a-z0-9_` (max 32 chars); collisions get numeric suffixes (e.g. `_2`).
  - `/skill <name> [input]` runs a skill by name (useful when native command limits prevent per-skill commands).
  - By default, skill commands are forwarded to the model as a normal request.
  - Skills may optionally declare `command-dispatch: tool` to route the command directly to a tool (deterministic, no model).
  - Example: `/prose` (OpenProse plugin) — see [OpenProse](/prose).
- **Native command arguments:** Discord uses autocomplete for dynamic options (and button menus when you omit required args). Telegram and Slack show a button menu when a command supports choices and you omit the arg.

## Usage surfaces (what shows where)

- **Provider usage/quota** (example: “Claude 80% left”) shows up in `/status` for the current model provider when usage tracking is enabled.
- **Per-response tokens/cost** is controlled by `/usage off|tokens|full` (appended to normal replies).
- `/model status` is about **models/auth/endpoints**, not usage.

## Model selection (`/model`)

`/model` is implemented as a directive.

Examples:

```
/model
/model list
/model 3
/model openai/gpt-5.2
/model opus@anthropic:default
/model status
```

Notes:

- `/model` and `/model list` show a compact, numbered picker (model family + available providers).
- On Discord, `/model` and `/models` open an interactive picker with provider and model dropdowns plus a Submit step.
- `/model <#>` selects from that picker (and prefers the current provider when possible).
- `/model status` shows the detailed view, including configured provider endpoint (`baseUrl`) and API mode (`api`) when available.

## Debug overrides

`/debug` lets you set **runtime-only** config overrides (memory, not disk). Owner-only. Disabled by default; enable with `commands.debug: true`.

Examples:

```
/debug show
/debug set messages.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset messages.responsePrefix
/debug reset
```

Notes:

- Overrides apply immediately to new config reads, but do **not** write to `openclaw.json`.
- Use `/debug reset` to clear all overrides and return to the on-disk config.

## Config updates

`/config` writes to your on-disk config (`openclaw.json`). Owner-only. Disabled by default; enable with `commands.config: true`.

Examples:

```
/config show
/config show messages.responsePrefix
/config get messages.responsePrefix
/config set messages.responsePrefix="[openclaw]"
/config unset messages.responsePrefix
```

Notes:

- Config is validated before write; invalid changes are rejected.
- `/config` updates persist across restarts.

## Surface notes

- **Text commands** run in the normal chat session (DMs share `main`, groups have their own session).
- **Native commands** use isolated sessions:
  - Discord: `agent:<agentId>:discord:slash:<userId>`
  - Slack: `agent:<agentId>:slack:slash:<userId>` (prefix configurable via `channels.slack.slashCommand.sessionPrefix`)
  - Telegram: `telegram:slash:<userId>` (targets the chat session via `CommandTargetSessionKey`)
- **`/stop`** targets the active chat session so it can abort the current run.
- **Slack:** `channels.slack.slashCommand` is still supported for a single `/openclaw`-style command. If you enable `commands.native`, you must create one Slack slash command per built-in command (same names as `/help`). Command argument menus for Slack are delivered as ephemeral Block Kit buttons.

## Web Tools


# Web tools

OpenClaw ships two lightweight web tools:

- `web_search` — Search the web via Brave Search API (default), Perplexity Sonar, or Gemini with Google Search grounding.
- `web_fetch` — HTTP fetch + readable extraction (HTML → markdown/text).

These are **not** browser automation. For JS-heavy sites or logins, use the
[Browser tool](/tools/browser).

## How it works

- `web_search` calls your configured provider and returns results.
  - **Brave** (default): returns structured results (title, URL, snippet).
  - **Perplexity**: returns AI-synthesized answers with citations from real-time web search.
  - **Gemini**: returns AI-synthesized answers grounded in Google Search with citations.
- Results are cached by query for 15 minutes (configurable).
- `web_fetch` does a plain HTTP GET and extracts readable content
  (HTML → markdown/text). It does **not** execute JavaScript.
- `web_fetch` is enabled by default (unless explicitly disabled).

## Choosing a search provider

| Provider            | Pros                                         | Cons                                     | API Key                                      |
| ------------------- | -------------------------------------------- | ---------------------------------------- | -------------------------------------------- |
| **Brave** (default) | Fast, structured results, free tier          | Traditional search results               | `BRAVE_API_KEY`                              |
| **Perplexity**      | AI-synthesized answers, citations, real-time | Requires Perplexity or OpenRouter access | `OPENROUTER_API_KEY` or `PERPLEXITY_API_KEY` |
| **Gemini**          | Google Search grounding, AI-synthesized      | Requires Gemini API key                  | `GEMINI_API_KEY`                             |

See [Brave Search setup](/brave-search) and [Perplexity Sonar](/perplexity) for provider-specific details.

### Auto-detection

If no `provider` is explicitly set, OpenClaw auto-detects which provider to use based on available API keys, checking in this order:

1. **Brave** — `BRAVE_API_KEY` env var or `search.apiKey` config
2. **Gemini** — `GEMINI_API_KEY` env var or `search.gemini.apiKey` config
3. **Perplexity** — `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY` env var or `search.perplexity.apiKey` config
4. **Grok** — `XAI_API_KEY` env var or `search.grok.apiKey` config

If no keys are found, it falls back to Brave (you'll get a missing-key error prompting you to configure one).

### Explicit provider

Set the provider in config:

```json5
{
  tools: {
    web: {
      search: {
        provider: "brave", // or "perplexity" or "gemini"
      },
    },
  },
}
```

Example: switch to Perplexity Sonar (direct API):

```json5
{
  tools: {
    web: {
      search: {
        provider: "perplexity",
        perplexity: {
          apiKey: "pplx-...",
          baseUrl: "https://api.perplexity.ai",
          model: "perplexity/sonar-pro",
        },
      },
    },
  },
}
```

## Getting a Brave API key

1. Create a Brave Search API account at [https://brave.com/search/api/](https://brave.com/search/api/)
2. In the dashboard, choose the **Data for Search** plan (not “Data for AI”) and generate an API key.
3. Run `openclaw configure --section web` to store the key in config (recommended), or set `BRAVE_API_KEY` in your environment.

Brave provides a free tier plus paid plans; check the Brave API portal for the
current limits and pricing.

### Where to set the key (recommended)

**Recommended:** run `openclaw configure --section web`. It stores the key in
`~/.openclaw/openclaw.json` under `tools.web.search.apiKey`.

**Environment alternative:** set `BRAVE_API_KEY` in the Gateway process
environment. For a gateway install, put it in `~/.openclaw/.env` (or your
service environment). See [Env vars](/help/faq#how-does-openclaw-load-environment-variables).

## Using Perplexity (direct or via OpenRouter)

Perplexity Sonar models have built-in web search capabilities and return AI-synthesized
answers with citations. You can use them via OpenRouter (no credit card required - supports
crypto/prepaid).

### Getting an OpenRouter API key

1. Create an account at [https://openrouter.ai/](https://openrouter.ai/)
2. Add credits (supports crypto, prepaid, or credit card)
3. Generate an API key in your account settings

### Setting up Perplexity search

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        provider: "perplexity",
        perplexity: {
          // API key (optional if OPENROUTER_API_KEY or PERPLEXITY_API_KEY is set)
          apiKey: "sk-or-v1-...",
          // Base URL (key-aware default if omitted)
          baseUrl: "https://openrouter.ai/api/v1",
          // Model (defaults to perplexity/sonar-pro)
          model: "perplexity/sonar-pro",
        },
      },
    },
  },
}
```

**Environment alternative:** set `OPENROUTER_API_KEY` or `PERPLEXITY_API_KEY` in the Gateway
environment. For a gateway install, put it in `~/.openclaw/.env`.

If no base URL is set, OpenClaw chooses a default based on the API key source:

- `PERPLEXITY_API_KEY` or `pplx-...` → `https://api.perplexity.ai`
- `OPENROUTER_API_KEY` or `sk-or-...` → `https://openrouter.ai/api/v1`
- Unknown key formats → OpenRouter (safe fallback)

### Available Perplexity models

| Model                            | Description                          | Best for          |
| -------------------------------- | ------------------------------------ | ----------------- |
| `perplexity/sonar`               | Fast Q&A with web search             | Quick lookups     |
| `perplexity/sonar-pro` (default) | Multi-step reasoning with web search | Complex questions |
| `perplexity/sonar-reasoning-pro` | Chain-of-thought analysis            | Deep research     |

## Using Gemini (Google Search grounding)

Gemini models support built-in [Google Search grounding](https://ai.google.dev/gemini-api/docs/grounding),
which returns AI-synthesized answers backed by live Google Search results with citations.

### Getting a Gemini API key

1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Create an API key
3. Set `GEMINI_API_KEY` in the Gateway environment, or configure `tools.web.search.gemini.apiKey`

### Setting up Gemini search

```json5
{
  tools: {
    web: {
      search: {
        provider: "gemini",
        gemini: {
          // API key (optional if GEMINI_API_KEY is set)
          apiKey: "AIza...",
          // Model (defaults to "gemini-2.5-flash")
          model: "gemini-2.5-flash",
        },
      },
    },
  },
}
```

**Environment alternative:** set `GEMINI_API_KEY` in the Gateway environment.
For a gateway install, put it in `~/.openclaw/.env`.

### Notes

- Citation URLs from Gemini grounding are automatically resolved from Google's
  redirect URLs to direct URLs.
- Redirect resolution uses the SSRF guard path (HEAD + redirect checks + http/https validation) before returning the final citation URL.
- This redirect resolver follows the trusted-network model (private/internal networks allowed by default) to match Gateway operator trust assumptions.
- The default model (`gemini-2.5-flash`) is fast and cost-effective.
  Any Gemini model that supports grounding can be used.

## web_search

Search the web using your configured provider.

### Requirements

- `tools.web.search.enabled` must not be `false` (default: enabled)
- API key for your chosen provider:
  - **Brave**: `BRAVE_API_KEY` or `tools.web.search.apiKey`
  - **Perplexity**: `OPENROUTER_API_KEY`, `PERPLEXITY_API_KEY`, or `tools.web.search.perplexity.apiKey`

### Config

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "BRAVE_API_KEY_HERE", // optional if BRAVE_API_KEY is set
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
    },
  },
}
```

### Tool parameters

- `query` (required)
- `count` (1–10; default from config)
- `country` (optional): 2-letter country code for region-specific results (e.g., "DE", "US", "ALL"). If omitted, Brave chooses its default region.
- `search_lang` (optional): ISO language code for search results (e.g., "de", "en", "fr")
- `ui_lang` (optional): ISO language code for UI elements
- `freshness` (optional): filter by discovery time
  - Brave: `pd`, `pw`, `pm`, `py`, or `YYYY-MM-DDtoYYYY-MM-DD`
  - Perplexity: `pd`, `pw`, `pm`, `py`

**Examples:**

```javascript
// German-specific search
await web_search({
  query: "TV online schauen",
  count: 10,
  country: "DE",
  search_lang: "de",
});

// French search with French UI
await web_search({
  query: "actualités",
  country: "FR",
  search_lang: "fr",
  ui_lang: "fr",
});

// Recent results (past week)
await web_search({
  query: "TMBG interview",
  freshness: "pw",
});
```

## web_fetch

Fetch a URL and extract readable content.

### web_fetch requirements

- `tools.web.fetch.enabled` must not be `false` (default: enabled)
- Optional Firecrawl fallback: set `tools.web.fetch.firecrawl.apiKey` or `FIRECRAWL_API_KEY`.

### web_fetch config

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true,
        maxChars: 50000,
        maxCharsCap: 50000,
        maxResponseBytes: 2000000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        userAgent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 14_7_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
        readability: true,
        firecrawl: {
          enabled: true,
          apiKey: "FIRECRAWL_API_KEY_HERE", // optional if FIRECRAWL_API_KEY is set
          baseUrl: "https://api.firecrawl.dev",
          onlyMainContent: true,
          maxAgeMs: 86400000, // ms (1 day)
          timeoutSeconds: 60,
        },
      },
    },
  },
}
```

### web_fetch tool parameters

- `url` (required, http/https only)
- `extractMode` (`markdown` | `text`)
- `maxChars` (truncate long pages)

Notes:

- `web_fetch` uses Readability (main-content extraction) first, then Firecrawl (if configured). If both fail, the tool returns an error.
- Firecrawl requests use bot-circumvention mode and cache results by default.
- `web_fetch` sends a Chrome-like User-Agent and `Accept-Language` by default; override `userAgent` if needed.
- `web_fetch` blocks private/internal hostnames and re-checks redirects (limit with `maxRedirects`).
- `maxChars` is clamped to `tools.web.fetch.maxCharsCap`.
- `web_fetch` caps the downloaded response body size to `tools.web.fetch.maxResponseBytes` before parsing; oversized responses are truncated and include a warning.
- `web_fetch` is best-effort extraction; some sites will need the browser tool.
- See [Firecrawl](/tools/firecrawl) for key setup and service details.
- Responses are cached (default 15 minutes) to reduce repeated fetches.
- If you use tool profiles/allowlists, add `web_search`/`web_fetch` or `group:web`.
- If the Brave key is missing, `web_search` returns a short setup hint with a docs link.

## Elevated Mode


# Elevated Mode (/elevated directives)

## What it does

- `/elevated on` runs on the gateway host and keeps exec approvals (same as `/elevated ask`).
- `/elevated full` runs on the gateway host **and** auto-approves exec (skips exec approvals).
- `/elevated ask` runs on the gateway host but keeps exec approvals (same as `/elevated on`).
- `on`/`ask` do **not** force `exec.security=full`; configured security/ask policy still applies.
- Only changes behavior when the agent is **sandboxed** (otherwise exec already runs on the host).
- Directive forms: `/elevated on|off|ask|full`, `/elev on|off|ask|full`.
- Only `on|off|ask|full` are accepted; anything else returns a hint and does not change state.

## What it controls (and what it doesn’t)

- **Availability gates**: `tools.elevated` is the global baseline. `agents.list[].tools.elevated` can further restrict elevated per agent (both must allow).
- **Per-session state**: `/elevated on|off|ask|full` sets the elevated level for the current session key.
- **Inline directive**: `/elevated on|ask|full` inside a message applies to that message only.
- **Groups**: In group chats, elevated directives are only honored when the agent is mentioned. Command-only messages that bypass mention requirements are treated as mentioned.
- **Host execution**: elevated forces `exec` onto the gateway host; `full` also sets `security=full`.
- **Approvals**: `full` skips exec approvals; `on`/`ask` honor them when allowlist/ask rules require.
- **Unsandboxed agents**: no-op for location; only affects gating, logging, and status.
- **Tool policy still applies**: if `exec` is denied by tool policy, elevated cannot be used.
- **Separate from `/exec`**: `/exec` adjusts per-session defaults for authorized senders and does not require elevated.

## Resolution order

1. Inline directive on the message (applies only to that message).
2. Session override (set by sending a directive-only message).
3. Global default (`agents.defaults.elevatedDefault` in config).

## Setting a session default

- Send a message that is **only** the directive (whitespace allowed), e.g. `/elevated full`.
- Confirmation reply is sent (`Elevated mode set to full...` / `Elevated mode disabled.`).
- If elevated access is disabled or the sender is not on the approved allowlist, the directive replies with an actionable error and does not change session state.
- Send `/elevated` (or `/elevated:`) with no argument to see the current elevated level.

## Availability + allowlists

- Feature gate: `tools.elevated.enabled` (default can be off via config even if the code supports it).
- Sender allowlist: `tools.elevated.allowFrom` with per-provider allowlists (e.g. `discord`, `whatsapp`).
- Unprefixed allowlist entries match sender-scoped identity values only (`SenderId`, `SenderE164`, `From`); recipient routing fields are never used for elevated authorization.
- Mutable sender metadata requires explicit prefixes:
  - `name:<value>` matches `SenderName`
  - `username:<value>` matches `SenderUsername`
  - `tag:<value>` matches `SenderTag`
  - `id:<value>`, `from:<value>`, `e164:<value>` are available for explicit identity targeting
- Per-agent gate: `agents.list[].tools.elevated.enabled` (optional; can only further restrict).
- Per-agent allowlist: `agents.list[].tools.elevated.allowFrom` (optional; when set, the sender must match **both** global + per-agent allowlists).
- Discord fallback: if `tools.elevated.allowFrom.discord` is omitted, the `channels.discord.allowFrom` list is used as a fallback (legacy: `channels.discord.dm.allowFrom`). Set `tools.elevated.allowFrom.discord` (even `[]`) to override. Per-agent allowlists do **not** use the fallback.
- All gates must pass; otherwise elevated is treated as unavailable.

## Logging + status

- Elevated exec calls are logged at info level.
- Session status includes elevated mode (e.g. `elevated=ask`, `elevated=full`).

## ACP Agents


# ACP agents

[Agent Client Protocol (ACP)](https://agentclientprotocol.com/) sessions let OpenClaw run external coding harnesses (for example Pi, Claude Code, Codex, OpenCode, and Gemini CLI) through an ACP backend plugin.

If you ask OpenClaw in plain language to "run this in Codex" or "start Claude Code in a thread", OpenClaw should route that request to the ACP runtime (not the native sub-agent runtime).

## Fast operator flow

Use this when you want a practical `/acp` runbook:

1. Spawn a session:
   - `/acp spawn codex --mode persistent --thread auto`
2. Work in the bound thread (or target that session key explicitly).
3. Check runtime state:
   - `/acp status`
4. Tune runtime options as needed:
   - `/acp model <provider/model>`
   - `/acp permissions <profile>`
   - `/acp timeout <seconds>`
5. Nudge an active session without replacing context:
   - `/acp steer tighten logging and continue`
6. Stop work:
   - `/acp cancel` (stop current turn), or
   - `/acp close` (close session + remove bindings)

## Quick start for humans

Examples of natural requests:

- "Start a persistent Codex session in a thread here and keep it focused."
- "Run this as a one-shot Claude Code ACP session and summarize the result."
- "Use Gemini CLI for this task in a thread, then keep follow-ups in that same thread."

What OpenClaw should do:

1. Pick `runtime: "acp"`.
2. Resolve the requested harness target (`agentId`, for example `codex`).
3. If thread binding is requested and the current channel supports it, bind the ACP session to the thread.
4. Route follow-up thread messages to that same ACP session until unfocused/closed/expired.

## ACP versus sub-agents

Use ACP when you want an external harness runtime. Use sub-agents when you want OpenClaw-native delegated runs.

| Area          | ACP session                           | Sub-agent run                      |
| ------------- | ------------------------------------- | ---------------------------------- |
| Runtime       | ACP backend plugin (for example acpx) | OpenClaw native sub-agent runtime  |
| Session key   | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Main commands | `/acp ...`                            | `/subagents ...`                   |
| Spawn tool    | `sessions_spawn` with `runtime:"acp"` | `sessions_spawn` (default runtime) |

See also [Sub-agents](/tools/subagents).

## Thread-bound sessions (channel-agnostic)

When thread bindings are enabled for a channel adapter, ACP sessions can be bound to threads:

- OpenClaw binds a thread to a target ACP session.
- Follow-up messages in that thread route to the bound ACP session.
- ACP output is delivered back to the same thread.
- Unfocus/close/archive/TTL expiry removes the binding.

Thread binding support is adapter-specific. If the active channel adapter does not support thread bindings, OpenClaw returns a clear unsupported/unavailable message.

Required feature flags for thread-bound ACP:

- `acp.enabled=true`
- `acp.dispatch.enabled=true`
- Channel-adapter ACP thread-spawn flag enabled (adapter-specific)
  - Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

### Thread supporting channels

- Any channel adapter that exposes session/thread binding capability.
- Current built-in support: Discord.
- Plugin channels can add support through the same binding interface.

## Start ACP sessions (interfaces)

### From `sessions_spawn`

Use `runtime: "acp"` to start an ACP session from an agent turn or tool call.

```json
{
  "task": "Open the repo and summarize failing tests",
  "runtime": "acp",
  "agentId": "codex",
  "thread": true,
  "mode": "session"
}
```

Notes:

- `runtime` defaults to `subagent`, so set `runtime: "acp"` explicitly for ACP sessions.
- If `agentId` is omitted, OpenClaw uses `acp.defaultAgent` when configured.
- `mode: "session"` requires `thread: true` to keep a persistent bound conversation.

Interface details:

- `task` (required): initial prompt sent to the ACP session.
- `runtime` (required for ACP): must be `"acp"`.
- `agentId` (optional): ACP target harness id. Falls back to `acp.defaultAgent` if set.
- `thread` (optional, default `false`): request thread binding flow where supported.
- `mode` (optional): `run` (one-shot) or `session` (persistent).
  - default is `run`
  - if `thread: true` and mode omitted, OpenClaw may default to persistent behavior per runtime path
  - `mode: "session"` requires `thread: true`
- `cwd` (optional): requested runtime working directory (validated by backend/runtime policy).
- `label` (optional): operator-facing label used in session/banner text.

### From `/acp` command

Use `/acp spawn` for explicit operator control from chat when needed.

```text
/acp spawn codex --mode persistent --thread auto
/acp spawn codex --mode oneshot --thread off
/acp spawn codex --thread here
```

Key flags:

- `--mode persistent|oneshot`
- `--thread auto|here|off`
- `--cwd <absolute-path>`
- `--label <name>`

See [Slash Commands](/tools/slash-commands).

## Session target resolution

Most `/acp` actions accept an optional session target (`session-key`, `session-id`, or `session-label`).

Resolution order:

1. Explicit target argument (or `--session` for `/acp steer`)
   - tries key
   - then UUID-shaped session id
   - then label
2. Current thread binding (if this conversation/thread is bound to an ACP session)
3. Current requester session fallback

If no target resolves, OpenClaw returns a clear error (`Unable to resolve session target: ...`).

## Spawn thread modes

`/acp spawn` supports `--thread auto|here|off`.

| Mode   | Behavior                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------- |
| `auto` | In an active thread: bind that thread. Outside a thread: create/bind a child thread when supported. |
| `here` | Require current active thread; fail if not in one.                                                  |
| `off`  | No binding. Session starts unbound.                                                                 |

Notes:

- On non-thread binding surfaces, default behavior is effectively `off`.
- Thread-bound spawn requires channel policy support (for Discord: `channels.discord.threadBindings.spawnAcpSessions=true`).

## ACP controls

Available command family:

- `/acp spawn`
- `/acp cancel`
- `/acp steer`
- `/acp close`
- `/acp status`
- `/acp set-mode`
- `/acp set`
- `/acp cwd`
- `/acp permissions`
- `/acp timeout`
- `/acp model`
- `/acp reset-options`
- `/acp sessions`
- `/acp doctor`
- `/acp install`

`/acp status` shows the effective runtime options and, when available, both runtime-level and backend-level session identifiers.

Some controls depend on backend capabilities. If a backend does not support a control, OpenClaw returns a clear unsupported-control error.

## ACP command cookbook

| Command              | What it does                                              | Example                                                        |
| -------------------- | --------------------------------------------------------- | -------------------------------------------------------------- |
| `/acp spawn`         | Create ACP session; optional thread bind.                 | `/acp spawn codex --mode persistent --thread auto --cwd /repo` |
| `/acp cancel`        | Cancel in-flight turn for target session.                 | `/acp cancel agent:codex:acp:<uuid>`                           |
| `/acp steer`         | Send steer instruction to running session.                | `/acp steer --session support inbox prioritize failing tests`  |
| `/acp close`         | Close session and unbind thread targets.                  | `/acp close`                                                   |
| `/acp status`        | Show backend, mode, state, runtime options, capabilities. | `/acp status`                                                  |
| `/acp set-mode`      | Set runtime mode for target session.                      | `/acp set-mode plan`                                           |
| `/acp set`           | Generic runtime config option write.                      | `/acp set model openai/gpt-5.2`                                |
| `/acp cwd`           | Set runtime working directory override.                   | `/acp cwd /Users/user/Projects/repo`                           |
| `/acp permissions`   | Set approval policy profile.                              | `/acp permissions strict`                                      |
| `/acp timeout`       | Set runtime timeout (seconds).                            | `/acp timeout 120`                                             |
| `/acp model`         | Set runtime model override.                               | `/acp model anthropic/claude-opus-4-5`                         |
| `/acp reset-options` | Remove session runtime option overrides.                  | `/acp reset-options`                                           |
| `/acp sessions`      | List recent ACP sessions from store.                      | `/acp sessions`                                                |
| `/acp doctor`        | Backend health, capabilities, actionable fixes.           | `/acp doctor`                                                  |
| `/acp install`       | Print deterministic install and enable steps.             | `/acp install`                                                 |

## Runtime options mapping

`/acp` has convenience commands and a generic setter.

Equivalent operations:

- `/acp model <id>` maps to runtime config key `model`.
- `/acp permissions <profile>` maps to runtime config key `approval_policy`.
- `/acp timeout <seconds>` maps to runtime config key `timeout`.
- `/acp cwd <path>` updates runtime cwd override directly.
- `/acp set <key> <value>` is the generic path.
  - Special case: `key=cwd` uses the cwd override path.
- `/acp reset-options` clears all runtime overrides for target session.

## acpx harness support (current)

Current acpx built-in harness aliases:

- `pi`
- `claude`
- `codex`
- `opencode`
- `gemini`

When OpenClaw uses the acpx backend, prefer these values for `agentId` unless your acpx config defines custom agent aliases.

Direct acpx CLI usage can also target arbitrary adapters via `--agent <command>`, but that raw escape hatch is an acpx CLI feature (not the normal OpenClaw `agentId` path).

## Required config

Core ACP baseline:

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: ["pi", "claude", "codex", "opencode", "gemini"],
    maxConcurrentSessions: 8,
    stream: {
      coalesceIdleMs: 300,
      maxChunkChars: 1200,
    },
    runtime: {
      ttlMinutes: 120,
    },
  },
}
```

Thread binding config is channel-adapter specific. Example for Discord:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      ttlHours: 24,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        spawnAcpSessions: true,
      },
    },
  },
}
```

If thread-bound ACP spawn does not work, verify the adapter feature flag first:

- Discord: `channels.discord.threadBindings.spawnAcpSessions=true`

See [Configuration Reference](/gateway/configuration-reference).

## Plugin setup for acpx backend

Install and enable plugin:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Local workspace install during development:

```bash
openclaw plugins install ./extensions/acpx
```

Then verify backend health:

```text
/acp doctor
```

### Pinned acpx install strategy (current behavior)

`@openclaw/acpx` now enforces a strict plugin-local pinning model:

1. The extension pins an exact acpx dependency in `extensions/acpx/package.json`.
2. Runtime command is fixed to the plugin-local binary (`extensions/acpx/node_modules/.bin/acpx`), not global `PATH`.
3. Plugin config does not expose `command` or `commandArgs`, so runtime command drift is blocked.
4. Startup registers the ACP backend immediately as not-ready.
5. A background ensure job verifies `acpx --version` against the pinned version.
6. If missing/mismatched, it runs plugin-local install (`npm install --omit=dev --no-save acpx@<pinned>`) and re-verifies before healthy.

Notes:

- OpenClaw startup stays non-blocking while acpx ensure runs.
- If network/install fails, backend remains unavailable and `/acp doctor` reports an actionable fix.

See [Plugins](/tools/plugin).

## Troubleshooting

| Symptom                                                                 | Likely cause                                   | Fix                                                        |
| ----------------------------------------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------- |
| `ACP runtime backend is not configured`                                 | Backend plugin missing or disabled.            | Install and enable backend plugin, then run `/acp doctor`. |
| `ACP is disabled by policy (acp.enabled=false)`                         | ACP globally disabled.                         | Set `acp.enabled=true`.                                    |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`       | Dispatch from normal thread messages disabled. | Set `acp.dispatch.enabled=true`.                           |
| `ACP agent "<id>" is not allowed by policy`                             | Agent not in allowlist.                        | Use allowed `agentId` or update `acp.allowedAgents`.       |
| `Unable to resolve session target: ...`                                 | Bad key/id/label token.                        | Run `/acp sessions`, copy exact key/label, retry.          |
| `--thread here requires running /acp spawn inside an active ... thread` | `--thread here` used outside a thread context. | Move to target thread or use `--thread auto`/`off`.        |
| `Only <user-id> can rebind this thread.`                                | Another user owns thread binding.              | Rebind as owner or use a different thread.                 |
| `Thread bindings are unavailable for <channel>.`                        | Adapter lacks thread binding capability.       | Use `--thread off` or move to supported adapter/channel.   |
| Missing ACP metadata for bound session                                  | Stale/deleted ACP session metadata.            | Recreate with `/acp spawn`, then rebind/focus thread.      |
