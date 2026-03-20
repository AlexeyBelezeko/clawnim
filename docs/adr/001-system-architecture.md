# ADR-001: System Architecture

**Status:** Accepted
**Date:** 2026-03-20

## Decision

```
┌─────────┐       ┌──────────────┐       ┌────────────────┐
│ Channel  │◄─────►│ Orchestrator │◄─────►│ Agent Runner   │
│ (user)   │       │ (single proc)│       │ (Docker ctnr)  │
└─────────┘       │   + SQLite   │       │  mounted vol   │
                  └──────────────┘       └────────────────┘
                         ▲                       ▲
                         │     shared volume      │
                         └───────────────────────┘
                            (file-based IPC)
```

### 1. Single-process orchestrator

- Connects to channels via channel interface
- Manages agent lifecycle (create, monitor, terminate containers)
- Persists state in embedded SQLite (WAL mode)
- Communicates with agents through files on a shared volume

### 2. Channel interface

Channels are pluggable. Each channel implements:
- `start()` — connect and begin receiving messages
- `stop()` — disconnect
- `send(chat_id, text)` — send a message to a chat
- `on_message(callback)` — register handler for incoming messages

Incoming messages normalized to: `{ channel, chat_id, topic_id, sender_id, sender_name, text }`

Telegram is the only channel for now. Others (Slack, Discord, CLI) can be added by implementing the same interface.

### 3. Docker containers for agents

- One container per agent session
- Mounted shared volume for IPC
- Ephemeral — destroyed after task completion or timeout
- Image includes agent runtime and tools

### 4. SQLite — only storage

- Task queue and history
- Agent session metadata
- User configuration
- Conversation context

### 5. File-based IPC

- Orchestrator writes task definitions, agents read
- Agents write status updates and results
- Protocol defined in ADR-002
