# BlueBubbles timeout behavior

BlueBubbles can deliver an iMessage and then fail to return an HTTP response before Hermes' client timeout. In logs, this often appears as a `ReadTimeout` around 30 seconds after Hermes generated a response.

This is not a markdown formatting failure. The message may already be in Messages.app.

## Bad behavior

Retrying after this timeout can send duplicate iMessages.

Sending the fallback text below is misleading and can also duplicate the response:

```text
(Response formatting failed, plain text:)
```

## Correct behavior

Treat read/write timeouts as delivery-ambiguous:

- log the attempted send details
- do not retry
- do not send formatting fallback
- let the user send another prompt if the message truly did not arrive

Treat connect timeouts as retryable because the request did not reach BlueBubbles.

## Avoiding slow chat creation

When BlueBubbles only gives Hermes a phone/email address, Hermes may call `/api/v1/chat/new`. For existing 1:1 chats this is slow and can hang in Messages.app.

The better path is:

1. Query existing chats with `/api/v1/chat/query`.
2. Match the inbound `reply_to` message GUID against `lastMessage` fields.
3. Cache aliases for chat identifier and participant addresses.
4. Use `/api/v1/message/text` with the resolved native `chatGuid`.

## Useful tests

```bash
source venv/bin/activate
python -m pytest tests/gateway/test_send_retry.py tests/gateway/test_bluebubbles.py tests/gateway/test_bluebubbles_send_path.py -q
```
