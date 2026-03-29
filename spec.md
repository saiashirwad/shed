# shed

Self-hosted developer workspace with a persistent AI agent. Your own always-on computer that thinks.

## What it is

A compiled single-binary daemon that turns a VPS into a full development environment with an AI assistant baked in. Shed IS the dev environment — you don't sync files from your laptop. You clone repos, edit code, run commands, and build projects entirely through shed's web UI and its AI agent.

## Architecture

### Runtime

- **Bun** — single compiled binary via `bun build --compile`
- **Single package** — backend and frontend in one `package.json`, Bun serves the built Svelte assets
- **Process supervisor model** — the daemon is a thin HTTP server + process manager. AI agent sessions, terminal PTYs, and cron jobs are managed child processes

### AI stack

Built on top of the pi monorepo (`@mariozechner/pi-*`):

- **`@mariozechner/pi-ai`** — unified LLM streaming API across 21+ providers (Anthropic, OpenAI, Google, Bedrock, Groq, xAI, etc.). Handles BYOK via env vars, inline keys, and OAuth token refresh
- **`@mariozechner/pi-agent-core`** — agent loop (LLM call -> tool execution -> repeat), state management, streaming events, tool validation, mid-execution steering

Shed provides:
- HTTP/WebSocket server (Hono)
- Session/conversation persistence (SQLite)
- Cron scheduler (Bun.CronJob)
- Tool implementations (reused from pi-coding-agent)
- Auth and UI

### Tools

Reused directly from `@mariozechner/pi-coding-agent` — fully decoupled from the CLI, exported as factory functions conforming to `AgentTool` from pi-agent-core:

- **read** — file reading with image support
- **write** — file creation/writing
- **edit** — text replacement with fuzzy matching
- **bash** — shell command execution with streaming
- **grep** — pattern search via ripgrep
- **find** — glob matching via fd
- **ls** — directory listing

External binaries (rg, fd) auto-downloaded on first run via pi's tools-manager.

Additional tools built for shed:
- **manage_schedule** — create/update/delete/list scheduled agent jobs (so users can manage schedules conversationally)
- **spawn_subagent** — create child agents with configurable tool sets

### Extensibility: Skills

Uses pi's skill system directly — same SKILL.md format (Agent Skills standard), same directories (`~/.pi/agent/skills/` and `.pi/skills/`). Skills built for pi work in shed with zero changes. MCP support deferred — skills first.

## Web UI

### Framework

Svelte. Built assets served directly by Bun.

### Layout: Tabbed workspace

- **Tab bar** — open multiple tabs, one visible at a time. Split panes deferred to later.
- **Tab types:**
  - **Chat** — conversation with an agent
  - **Terminal** — interactive shell (xterm.js + Bun.Terminal over WebSocket)
  - **Editor** — CodeMirror, editable, save writes to disk
  - **Settings** — OAuth logins, API key entry, default model, subagent cap, schedules, system prompt

### Chat layout: Two columns

When a chat tab is active:

- **Left: Conversation pane** — pure conversation. User messages and agent text responses. Tool calls do NOT appear here — the chat stays clean.
- **Right: Workspace panel** — always visible. Shows what the agent is doing and has touched:
  - **Files section** — files touched this conversation, last action (read/edited/created), diff indicators, click to open in editor tab. Currently active files pulse/highlight.
  - **Commands section** — recent shell commands, exit status, expandable output. Running commands show a spinner.
  - **Agent tree** — tree view of the current agent and its subagents:
    ```
    > Main Agent (active)
      |- files, commands...
      |- > Subagent: "Fix auth tests" (active)
      |- > Subagent: "Update docs" (active)
      |- > Subagent: "Refactor utils" (done, collapsed)
    ```
    Clicking a subagent swaps the chat pane to show that subagent's conversation. The workspace panel updates to reflect that subagent's context. Breadcrumb navigation to go back up: `Main Agent > Fix auth tests`.

### Tool call display

Tool calls appear only in the workspace panel, never in the chat stream. The chat contains the agent's reasoning and conclusions. The workspace panel shows the work.

### Model selection

Per-tab dropdown in the chat tab header. Each conversation can use a different model. Default model configurable in settings.

## Subagents

### Spawning

The main agent has a `spawn_subagent` tool. Parameters:
- Task description / prompt
- Tool set — configurable per spawn. Presets like `"read-only"`, `"coding"`, `"full"`, or explicit list like `["read", "grep", "find", "bash"]`. Default: full access.

### Concurrency

Configurable cap, default 5 concurrent subagents. Jobs beyond the cap queue with a semaphore.

### Depth

Arbitrary depth — subagents can spawn their own subagents. Data model is a tree (each agent has a `parentId`). UI handles one level well for v1; deeper nesting renders in the tree but without special UX treatment.

## Persistence

### SQLite via `bun:sqlite`

- **conversations** — id, title, created_at, model, parent_agent_id, is_scheduled, schedule_name
- **messages** — id, conversation_id, role, content (JSON), timestamp

Full `AgentMessage[]` stored as JSON — already serializable from pi-agent-core.

## Provider auth

### Two mechanisms, both supported:

**API keys** — entered in the settings UI, stored in SQLite. Pi-ai's `getEnvApiKey()` also reads from environment variables.

**OAuth** (Claude Max, ChatGPT Plus/Pro, GitHub Copilot, Google Gemini CLI, Antigravity):
- Uses pi-ai's OAuth utilities for token exchange and refresh
- Web UI flow: click "Login with X" -> opens auth URL in new tab -> user authenticates -> pastes code back into shed's UI -> shed exchanges code for tokens
- Tokens stored in SQLite, auto-refreshed on expiry

Reads from pi's `~/.pi/agent/auth.json` as well — if you've already logged in via pi CLI, shed picks up those credentials.

## Scheduled agents

### Three ways to create:

1. **Config file** — JSON file as source of truth, easy to version control
2. **Settings UI** — create/edit/delete from a schedules section
3. **Conversationally** — tell the agent "create a scheduled job to check API health every 6 hours" via the `manage_schedule` tool

All three write to the same backing store.

### Execution

Bun.CronJob runs scheduled agents. Each run creates a conversation tagged as "scheduled" with the job name. Appears in the conversation list with a badge — open it to see what the agent did.

### Job definition

- Cron expression or interval
- Prompt / task description
- Model to use
- Tool set to allow

## Auth

Simple token auth. `SHED_TOKEN` env var. On first visit, prompt for token, set HTTP-only cookie. Single-user tool.

## System prompt

Layered:
- **Base layer** — shipped with shed. Instructs the agent on tool usage, safety, file operations. Not user-editable (unless they modify the source).
- **User layer** — editable via a `system-prompt.md` file or through the settings UI. Appended on top of the base layer. Personality, project context, preferences.

## Data storage

All shed data lives in `~/.config/shed/`:

```
~/.config/shed/
  shed.db          # SQLite — conversations, messages, scheduled jobs, credentials
  settings.json    # user config
  system-prompt.md # user's custom system prompt layer
  bin/             # auto-downloaded rg, fd
```

## Deployment

### Build

`bun build --compile` produces a single executable.

### Run

Download binary, run it. First run auto-downloads rg/fd to `~/.config/shed/bin/`.

### Network

Shed binds to `localhost:3000` by default (configurable via `SHED_HOST` and `SHED_PORT` env vars). HTTPS is the user's responsibility.

**Recommended setup: Cloudflare Tunnel.** Install `cloudflared`, point a subdomain at `localhost:3000`, automatic HTTPS with zero cert management. Documented in README as the blessed path. Users can also use Caddy, nginx, Tailscale, SSH tunnels, etc.

## WebSocket protocol

One WebSocket per tab:
- `/ws/chat/:conversationId` — chat streaming
- `/ws/terminal/:id` — PTY I/O

## Non-goals for v1

- Split panes / multi-tab visible layouts
- MCP server support (skills first)
- Docker deployment
- Multi-user / team features
- File syncing / local-remote split
- Fancy subagent deep-nesting UX (tree renders, but no special treatment beyond one level)
