# shed

Self-hosted developer workspace with a persistent AI agent. Your own always-on computer that thinks.

## What it is

A lightweight daemon running on a VPS (or locally) that gives you:

- A persistent AI assistant with full access to your filesystem and terminal
- Sandboxed command execution (Docker)
- Conversation context that survives across sessions
- Scheduled agents (cron-style) that run unattended
- A simple chat interface (web UI + Telegram/messaging bot)

## Core features

- **Persistent context** — remembers your projects, files, preferences across sessions
- **Terminal access** — AI can run commands, install packages, manage services in a sandbox
- **File workspace** — shared filesystem the AI can read/write, you can sync to
- **Task queue** — fire-and-forget tasks, scheduled jobs, overnight agents that commit/test/summarize
- **Model agnostic** — bring your own API keys (Claude, OpenAI, local via Ollama)
- **Integrations** — start minimal: git, Linear, notifications. Add more over time.

## Non-goals

- Not a code editor or IDE
- Not a SaaS platform
- No plugin marketplace, no ecosystem play
- No "community templates" or prompt libraries

## Stack (tentative)

- Go single binary
- SQLite for persistence (conversations, task queue, config)
- Docker for sandboxed execution
- Simple web UI (htmx or plain HTML, nothing heavy)
- WebSocket for streaming responses

## Design principles

- Fast to start, fast to respond
- Single binary, minimal dependencies
- Opinionated defaults, minimal config
- Built for people who use a terminal daily
- No bloat, no framework churn
