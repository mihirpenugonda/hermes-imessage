# hermes-imessage

Patches and a Hermes skill for making Hermes Agent's BlueBubbles/iMessage gateway safer and less annoying in real iMessage conversations.

This repo is modeled after benjaminsehl/hermes-imessage, but focuses on the reliability fixes from Mihir's Hermes setup: no duplicate sends after BlueBubbles timeouts, better chat GUID resolution, better failure logs, and stable DM session IDs.

## What it fixes

BlueBubbles can deliver an iMessage but time out before returning an HTTP response to Hermes. Hermes used to treat that as a send failure, retry, and then try the misleading fallback message:

(Response formatting failed, plain text:)

That could create duplicate texts or make a delivery-timeout look like a markdown/formatting problem.

This patch changes Hermes so BlueBubbles read/write timeouts are treated as ambiguous delivery results. Hermes logs the attempted payload and returns without retrying or sending the plain-text fallback.

It also avoids the slow `/api/v1/chat/new` path for existing 1:1 conversations by resolving and caching the existing BlueBubbles chat GUID from inbound message context.

## Changes included

Message delivery:
- Treat BlueBubbles read/write timeouts as ambiguous delivery, not formatting failures.
- Do not retry or send fallback text after ambiguous delivery timeouts.
- Keep connect timeouts retryable because no message reached BlueBubbles.
- Log the exact attempted payload, reply target, metadata, raw response, and retryability when sends fail.

Chat resolution:
- Cache BlueBubbles chat GUID aliases for chat identifiers and participant addresses.
- Use the inbound reply message GUID to find the existing BlueBubbles chat via `/api/v1/chat/query`.
- Prefer existing chats over `/api/v1/chat/new` for normal replies.
- Use a longer timeout for true new-chat creation because Messages.app can block.

Session stability:
- Prefer sender phone/email as the BlueBubbles DM session ID.
- Store the raw BlueBubbles chat GUID as `chat_id_alt`.
- Avoid duplicate sessions for the same 1:1 chat when BlueBubbles alternates between address and raw GUID identifiers.

Tests:
- Adds coverage for ambiguous timeouts, send-failure logging, reply-guid chat lookup, and DM session ID normalization.

## Installation

From a Hermes Agent checkout:

```bash
cd ~/.hermes/hermes-agent
git apply /path/to/hermes-imessage/patches/bluebubbles-reliability.patch
hermes gateway restart
```

To install the skill:

```bash
cp -r /path/to/hermes-imessage/skill/imessage-reliability ~/.hermes/skills/imessage-reliability
```

## Verification

```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate
python -m pytest tests/gateway/test_send_retry.py tests/gateway/test_bluebubbles.py tests/gateway/test_bluebubbles_send_path.py -q
hermes gateway restart
tail -f ~/.hermes/logs/gateway.log
```

Expected test result for the patch snapshot in this repo:

```text
86 passed
```

In live logs, normal replies to existing iMessage DMs should use `/api/v1/message/text`, not repeatedly fall back to `/api/v1/chat/new`.

## Files

```text
patches/bluebubbles-reliability.patch
skill/imessage-reliability/SKILL.md
references/bluebubbles-timeouts.md
```

## Requirements

- Hermes Agent with BlueBubbles gateway support
- BlueBubbles server running on a Mac
- Python test environment for Hermes if you want to run the test suite

## Related work

- benjaminsehl/hermes-imessage: iMessage UX patches for shorter bubbles, acknowledgments, debouncing, and platform notes.
- This repo: delivery correctness and BlueBubbles timeout/session reliability.

## License

MIT
