---
name: imessage-reliability
description: BlueBubbles/iMessage reliability guidance for Hermes Gateway: avoid duplicate sends on ambiguous timeouts, resolve existing chats before creating new ones, and keep iMessage replies plain.
version: 1.0.0
author: Mihir Penugonda / Hermes Agent
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [iMessage, BlueBubbles, gateway, reliability, messaging]
    related_skills: [imessage]
---

# iMessage Reliability

Use this when configuring, debugging, or modifying Hermes Agent's BlueBubbles/iMessage gateway.

## Core rule

A BlueBubbles read/write timeout after an outbound send is delivery-ambiguous. The iMessage may already have been delivered even though Hermes did not receive the HTTP response.

Do not blindly retry and do not send the `(Response formatting failed, plain text:)` fallback after these timeouts. That can duplicate messages and mislabel a BlueBubbles delivery timeout as a markdown issue.

Connect timeouts are different: if Hermes never connected to BlueBubbles, the message did not reach the server and normal retry behavior is safer.

## Preferred send path

For replies to existing 1:1 iMessage conversations:

1. Use the inbound message GUID (`reply_to`) to find the owning BlueBubbles chat with `/api/v1/chat/query`.
2. Cache the chat GUID against the sender phone/email, chat identifier, and participant addresses.
3. Send through `/api/v1/message/text` with the resolved `chatGuid`.
4. Only use `/api/v1/chat/new` when there is no existing chat to resolve.
5. Give `/api/v1/chat/query` a short timeout, around 10 seconds.
6. Give real `/api/v1/chat/new` creation a longer timeout, around 90 seconds, because Messages.app can block.

## Logging requirements

When send failures happen, logs should include:

- failure stage: initial send, retry, fallback, or delivery-failure notice
- chat_id
- reply_to
- metadata
- exact attempted content
- SendResult.error
- SendResult.raw_response
- retryable flag

In the BlueBubbles `_api_post` helper, wrap failures so the error includes the endpoint path and exception type. Raw `ReadTimeout` strings can be empty.

Good example:

```text
POST /api/v1/chat/new failed before response: ReadTimeout:
```

## Session identity

For BlueBubbles DMs, prefer the sender phone/email as the Hermes session chat ID.

Keep the raw BlueBubbles chat GUID as `chat_id_alt` so replies can still use the stable native chat GUID when needed.

This prevents duplicate sessions when BlueBubbles emits one webhook as `any;-;+1...` and another as `+1...` for the same conversation.

## iMessage style

Use plain conversational text.

Avoid markdown-heavy replies over iMessage:

- no bold formatting
- no bullet-heavy lists unless the user explicitly asks
- no headers
- no asterisks for emphasis
- no fallback messages that mention formatting unless the actual failure is a formatting parse failure

## Verification

Run targeted tests from the Hermes checkout:

```bash
source venv/bin/activate
python -m pytest tests/gateway/test_send_retry.py tests/gateway/test_bluebubbles.py tests/gateway/test_bluebubbles_send_path.py -q
```

Then restart and watch logs:

```bash
hermes gateway restart
tail -f ~/.hermes/logs/gateway.log
```

Check that normal DM replies do not repeatedly call `/api/v1/chat/new` and that read/write timeouts do not trigger duplicate fallback sends.
