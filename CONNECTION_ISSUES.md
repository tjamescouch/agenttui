Connection issues and quick mitigations

Symptoms:
- Websocket disconnects and reconnects cause recent messages to be echoed back to clients, appearing as duplicates.
- Users report ~50% of messages "bouncing back" during unstable periods.
- Rapid reconnect timers can cause overlapping connection attempts, increasing noise.

Root causes (likely):
- Server may replay recent events to rehydrate clients on connect; without a local dedupe, clients can show duplicates.
- Client reconnect logic can schedule multiple timers or reconnect attempts which overlap.

Mitigations applied in this branch:
1) Short-lived client-side dedupe cache (3s) keyed by message id/from/to/content to ignore quick duplicates.
   - Implemented in index.js inside createChatClient.
   - This reduces visible duplicate messages during immediate reconnect/echo windows.

2) Clear scheduled reconnect timer on successful connect to avoid overlapping reconnect attempts.
   - Prevents duplicate connect attempts piling up when network flaps.

Suggested server-side improvements:
- Ensure messages have stable, unique IDs and do not re-broadcast messages that the server previously acknowledged to the same client.
- Add server-side dedupe or per-connection watermarking to avoid re-sending already-delivered messages.

How to test locally:
- Start a local AgentChat server at ws://127.0.0.1:6667 and run agenttui.
- Simulate a network flap (e.g., stop the server and start it quickly) and observe whether duplicates appear.
- With this branch, duplicates within ~3s should be suppressed.

If you'd like, I can:
- Extract the dedupe helper into a small module and add unit tests.
- Add a CLI flag to tune the dedupe window and reconnect delay.
- Attempt to push this branch and open a PR, or produce a patch bundle for maintainers to apply.

Bundle path created: /tmp/oai_fuzzy_filter.bundle
Branch: oai/fuzzy-filter-enter-chat
Commit: f8fd7cc (latest)
