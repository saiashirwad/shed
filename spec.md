# shed

Self-hosted developer workspace with a persistent AI agent. Your own always-on computer that thinks.

## What it is

A compiled single-binary daemon that turns a VPS into a full development environment with an AI assistant baked in. Shed IS the dev environment — you don't sync files from your laptop. You clone repos, edit code, run commands, and build projects entirely through shed's web UI and its AI agent.

## Architecture

### Runtime

- **Bun** — single compiled binary via `bun build --compile`
- **Single package** — backend and frontend in one `package.json`, Bun serves the built React assets
- **Process supervisor model** — the daemon is a thin HTTP server + process manager. AI agent sessions, terminal PTYs, and cron jobs are managed child processes
- **Graceful shutdown** — on `SIGTERM`, finish the current LLM streaming response, persist to SQLite, then exit. Active terminals and agent sessions are accepted losses on restart for v1.

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
- **bash** — shell command execution with streaming (default 2-minute timeout, configurable, two-stage kill: SIGTERM then SIGKILL)
- **grep** — pattern search via ripgrep
- **find** — glob matching via fd
- **ls** — directory listing

External binaries (rg, fd) auto-downloaded on first run via pi's tools-manager.

Additional tools built for shed:
- **manage_schedule** — create/update/delete/list scheduled agent jobs (so users can manage schedules conversationally)
- **spawn_subagent** — create child agents with configurable tool sets

### Extensibility: Skills

Uses pi's skill system directly — same SKILL.md format (Agent Skills standard), same directories (`~/.pi/agent/skills/` and `.pi/skills/`). Skills built for pi work in shed with zero changes. MCP support deferred — skills first.

### Data model: Two-layer sync

Two distinct data transport layers:

1. **Chat streaming** — direct WebSocket stream for live agent messages and tool calls. Token-by-token, low-latency. This is the real-time path.
2. **Sync layer (TanStack DB)** — everything else: conversation list, file states, agent tree, subagent statuses, schedules, settings. The UI reads from a local replica via TanStack DB, feels instant on interaction, syncs with the server's SQLite in the background.

**Seam between layers:** Streaming writes to a client-side buffer the UI renders from. On persist, the sync layer delivers the canonical row. The client reconciles by ID — if it already has the message from streaming, it swaps in the synced version silently. Scrolling back to old conversations loads purely from the sync layer.

## Web UI

### Framework

React + Vite. No SSR — single-user app behind auth. Built assets served directly by Bun. TanStack DB for reactive state synchronization.

### Layout: Tabbed workspace

- **Tab bar** — open multiple tabs, one visible at a time. Split panes deferred to later.
- **Tab types:**
  - **Chat** — conversation with an agent
  - **Terminal** — interactive shell (xterm.js + Bun PTY over WebSocket)
  - **Editor** — CodeMirror, editable, save writes to disk. LSP support deferred to v2.
  - **Settings** — OAuth logins, API key entry, default model, subagent cap, schedules, system prompt

### Chat layout: Two columns

When a chat tab is active:

- **Left: Conversation pane** — user messages, agent text responses, and **inline collapsed tool calls**. Tool calls appear as minimal one-line summaries (e.g., `read auth.ts`, `bash: npm test`) that can be expanded to see full input/output. The chat stream stays clean but the narrative stays coherent — the user doesn't have to cross-reference a separate panel to follow the agent's reasoning.
- **Right: Workspace sidebar** — persistent state summary, slim by design:
  - **Agent tree** — tree view of the current agent and its subagents with status indicators and one-line status per node (e.g., "running — editing auth.ts", "done — 3 files changed"). Unread/activity badges on nodes with new messages or completions. Clicking a subagent swaps the chat pane to show that subagent's conversation. Breadcrumb navigation to go back up: `Main Agent > Fix auth tests`.
    ```
    > Main Agent (active)
      |- > Subagent: "Fix auth tests" (active) •
      |- > Subagent: "Update docs" (active)
      |- > Subagent: "Refactor utils" (done, 3 files)
    ```
  - **Files section** — files touched this conversation, last action (read/edited/created), diff indicators, click to open in editor tab.

### Background streaming

All agent WebSocket connections stay alive regardless of which agent is currently viewed. Switching between agents in the UI is instant — no loading state, the messages are already buffered.

### Model selection

Per-tab dropdown in the chat tab header. Each conversation can use a different model. Default model configurable in settings.

### Token counter

Running token usage and estimated cost displayed per conversation in the chat header. No enforcement — purely for user awareness.

## Subagents

### Spawning

The main agent has a `spawn_subagent` tool. Parameters:
- Task description / prompt
- Tool set — configurable per spawn. Presets like `"read-only"`, `"coding"`, `"full"`, or explicit list like `["read", "grep", "find", "bash"]`. Default: full access.

### Concurrency

Configurable advisory cap, default 5 concurrent subagents. Jobs beyond the cap queue with a semaphore. Warning shown when cap is reached, not a hard block.

### Depth

Arbitrary depth — subagents can spawn their own subagents. Data model is a tree (each agent has a `parentId`). UI handles one level well for v1; deeper nesting renders in the tree but without special UX treatment.

### Guardrails

- **Doom loop detection** — if the same tool call repeats 3 times consecutively, the agent pauses and flags it to the user before continuing.
- **Bash timeout** — 2-minute default, configurable per-command. Two-stage kill (SIGTERM → SIGKILL).
- No hard token/cost budgets — the token counter in the UI provides visibility, the user decides when to stop.

## Persistence

### SQLite via `bun:sqlite`

- **conversations** — id, title, created_at, model, parent_agent_id, is_scheduled, schedule_name
- **messages** — id, conversation_id, role, content (JSON), timestamp

Full `AgentMessage[]` stored as JSON — already serializable from pi-agent-core.

**Conversation list** shows only top-level conversations (`parent_agent_id IS NULL`). Subagent conversations are accessed through the agent tree within their parent. Deleting a parent conversation cascade-deletes all subagent conversations.

## Provider auth

### Two mechanisms, both supported:

**API keys** — entered in the settings UI, stored in SQLite. Pi-ai's `getEnvApiKey()` also reads from environment variables.

**OAuth** (Claude Max, ChatGPT Plus/Pro, GitHub Copilot, Google Gemini CLI, Antigravity):
- Uses pi-ai's OAuth utilities for token exchange and refresh
- Web UI flow: click "Login with X" -> opens auth URL in new tab -> user authenticates -> pastes code back into shed's UI -> shed exchanges code for tokens
- Tokens stored in SQLite, auto-refreshed on expiry

Reads from pi's `~/.pi/agent/auth.json` as well — if you've already logged in via pi CLI, shed picks up those credentials.

## Scheduled agents

### Two ways to create:

1. **Settings UI** — create/edit/delete from a schedules section
2. **Conversationally** — tell the agent "create a scheduled job to check API health every 6 hours" via the `manage_schedule` tool. Chat is the primary interface for managing schedules.

**Config file seed:** A JSON config file can be used to bootstrap initial schedules on first boot. Shed reads it, imports jobs into SQLite, then the config file is not consulted again unless the user runs `shed schedules import`. SQLite is the single source of truth.

### Execution

Bun.CronJob runs scheduled agents. Each run creates a conversation tagged as "scheduled" with the job name. Appears in the conversation list with a badge — open it to see what the agent did. Failed runs get a visual failure indicator (red badge) in the conversation list.

### Job definition

- Cron expression or interval
- Prompt / task description
- Model to use
- Tool set to allow

## Auth

Simple token auth. `SHED_TOKEN` env var. On first visit, prompt for token, set cookie:
- `HttpOnly; Secure; SameSite=Strict`
- Configurable expiry, default 30 days
- WebSocket upgrade requests must validate the cookie — `/ws/*` endpoints are not unprotected

Single-user tool. `shed token rotate` CLI command generates a new token and invalidates the old cookie.

## System prompt

Layered, applied in order:
1. **Base layer** — shipped with shed. Instructs the agent on tool usage, safety, file operations. Not user-editable (unless they modify the source).
2. **User layer** — editable via a `system-prompt.md` file or through the settings UI. Global, applied to all conversations.
3. **Per-conversation context** — optional override when starting a new conversation, appended for that conversation only. Covers "different projects need different context" without a full template system.
4. **Active skills** — skill instructions appended last.

## Terminal

Uses `Bun.spawn` with the `terminal` option for full interactive PTY support:

```typescript
const proc = Bun.spawn(["bash", "-i"], {
  terminal: {
    cols: 80,
    rows: 24,
    data(term, data) { /* send to xterm.js via WebSocket */ },
    exit(term, code, signal) { /* cleanup */ },
  },
});

// Resize from xterm.js client
proc.terminal.resize(newCols, newRows);  // sends SIGWINCH
```

**Known issue:** Ctrl+C (`\x03`) bypasses the kernel's line discipline (Bun issue #25779). Workaround: intercept `\x03` in writes and manually send `proc.kill('SIGINT')`. A "Send SIGINT" button in the terminal UI provides an additional escape hatch.

**Note:** node-pty does not work under Bun (onData never fires, issue #25822). No fallback — Bun's native PTY is the only path.

## Data storage

All shed data lives in `~/.config/shed/`:

```
~/.config/shed/
  shed.db          # SQLite — conversations, messages, scheduled jobs, credentials
  settings.json    # user config
  system-prompt.md # user's custom system prompt layer
  bin/             # auto-downloaded rg, fd
```

## Observability

- **Logs** — structured JSON to stdout. Captured by systemd/journald or whatever process manager runs shed.
- **Scheduled run failures** — visible in conversation list with failure badge. No external alerting for v1.

## Deployment

### Build

`bun build --compile` produces a single executable.

### Install

GitHub releases with a one-liner curl install script. `shed update` CLI command pulls the latest release binary and restarts the daemon.

### Run

Download binary, run it. First run auto-downloads rg/fd to `~/.config/shed/bin/`. If rg/fd download fails (air-gapped VPS), shed reports a clear error telling the user to install them manually.

### Network

Shed binds to `localhost:3000` by default (configurable via `SHED_HOST` and `SHED_PORT` env vars). HTTPS is the user's responsibility.

**Recommended setup: Cloudflare Tunnel.** Install `cloudflared`, point a subdomain at `localhost:3000`, automatic HTTPS with zero cert management. Documented in README as the blessed path. Users can also use Caddy, nginx, Tailscale, SSH tunnels, etc.

## WebSocket protocol

One WebSocket per tab:
- `/ws/chat/:conversationId` — chat streaming (direct, low-latency)
- `/ws/terminal/:id` — PTY I/O

All other UI state synced via TanStack DB (not per-tab WebSockets).

## Non-goals for v1

- Split panes / multi-tab visible layouts
- MCP server support (skills first)
- Docker deployment
- Multi-user / team features
- File syncing / local-remote split
- Fancy subagent deep-nesting UX (tree renders, but no special treatment beyond one level)
- LSP / language server integration (v2)
- Session recovery on daemon restart
- External alerting (webhook/email on failures)
- Hard token/cost budgets (visibility only)
