# Telegram Channel

Implements the channel interface for Telegram Bot API.

## Configuration

`.env` file:
```
TELEGRAM_BOT_TOKEN=...
```

## Connection

Uses Telegram Bot API with long polling (`getUpdates`). No webhooks.

## Access control

On each incoming message:
1. Check `sender_id` in `allowed_users` — drop if not found
2. Check `chat_id` in `allowed_chats` — drop if not found

Unauthorized messages silently ignored.

## Message handling

All text messages in allowed chats from allowed users are forwarded to `session_manager` as:
```
{
  channel: "telegram",
  chat_id: <chat_id>,
  topic_id: <topic_id or null>,
  sender_id: <user_id>,
  sender_name: <first_name + last_name>,
  text: <message text>
}
```

## Commands

| Command | Response |
|---|---|
| `/chatid` | Reply with current chat ID (and topic ID if applicable) |
| `/kill` | Forward to `session_manager` to stop agent in current chat/topic |
| `/status` | Forward to `session_manager`, reply with agent state |

## Topic support

In group chats with topics (forums):
- `message_thread_id` from Telegram update used as `topic_id`
- Each topic is an independent session
- Session ID: `telegram_{chat_id}_{topic_id}`

In chats without topics:
- `topic_id` is null
- Session ID: `telegram_{chat_id}`

## Sending messages

`send(chat_id, text)`:
- Calls `sendMessage` Bot API method
- If `topic_id` present on session, sets `message_thread_id`
- Messages longer than 4096 chars split into multiple sends

## Interface implementation

```
start()       — begin long polling
stop()        — stop polling, cancel pending getUpdates
send()        — sendMessage via Bot API
on_message()  — register callback for incoming messages
```
