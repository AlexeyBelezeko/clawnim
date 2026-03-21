# ADR-005: Telegram Bot Interface

**Status:** Accepted
**Date:** 2026-03-20

## Decision

Telegram is the first channel implementation. Bot API with long polling (no webhooks).

Implements the channel interface defined in ADR-001. Other channels can be added by implementing the same interface.

### Commands

| Command | Scope | Description |
|---|---|---|
| `/chatid` | any | Reply with current chat ID, user ID, and topic ID. Bypasses access control. |

No admin/user role distinction. All `allowed_users` have equal access.

### Message routing

Any text message in an allowed chat from an allowed user is processed:

1. If no agent running for this chat/topic — spawn container, create task
2. If agent already running — pipe message as follow-up via `input/*.json`

No trigger keyword or bot mention required.

### Topic handling

In group chats with topics enabled:
- Each topic gets its own session and agent container automatically
- First message in a topic spawns the agent
- Session ID: `telegram_{chat_id}_{topic_id}`

In chats without topics:
- One session per chat
- Session ID: `telegram_{chat_id}`

### Output format

Agent responses sent as plain text Telegram messages. No markdown, no file attachments.

Messages longer than Telegram's 4096 char limit are split into multiple messages.

### Access control flow

```
message received
  -> check sender in allowed_users? no -> drop
  -> check chat in allowed_chats? no -> drop
  -> route to session
```

### Bot configuration

`.env` file:
```
TELEGRAM_BOT_TOKEN=...
```

Allowed users and chats managed via SQLite (`allowed_users`, `allowed_chats` tables). Initial setup requires direct DB insert or a seed script.
