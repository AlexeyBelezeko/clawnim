# ADR-001: System Architecture

**Status:** Accepted
**Date:** 2026-03-20

## Decision

```
┌─────────┐       ┌──────────────┐       ┌────────────────┐
│ Telegram │◄─────►│ Orchestrator │◄─────►│ Agent Runner   │
│ (user)   │       │ (single proc)│       │ (Docker ctnr)  │
└─────────┘       │   + SQLite   │       │  mounted vol   │
                  └──────────────┘       └────────────────┘
                         ▲                       ▲
                         │     shared volume      │
                         └───────────────────────┘
                            (file-based IPC)
```

### 1. Single-process orchestrator

- Connects to Telegram Bot API
- Manages agent lifecycle (create, monitor, terminate containers)
- Persists state in embedded SQLite (WAL mode)
- Communicates with agents through files on a shared volume

### 2. Telegram — only user channel

- Send tasks as messages
- Receive progress updates and results as replies
- Manage agent sessions through bot commands

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
