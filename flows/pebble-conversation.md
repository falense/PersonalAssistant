# Flow: Pebble conversation

A trigger brings a question or task. Claude handles it, then keeps the conversation
going on the Pebble with open-ended follow-up questions. The conversation ends when the
user stops answering before the timeout.

## Sequence

```
trigger (API / claude_task, evt_…)
   │
   ▼
1. Parse task ──► 2. Do the work ──► 3. ask_user(pebble, answer + open follow-up)
                        ▲                         │
                        │        answer text      │
                        └─────────────────────────┤
                                                  │ timeout / refusal / "nei takk"
                                                  ▼
                                   4. finish_task(evt_…, summary, page?)
```

## Steps

1. **Parse the task.** Pull out the request, the `evt_…` ID and the starting device. If
   the trigger names no device, use `pebble`. Call `get_devices` and confirm
   `linkUp: true` and `voiceAvailable: true`. If the Pebble is unreachable, do the work
   anyway, then go to step 4. The summary and the push notification still reach the
   user.

2. **Do the work** with the tools you need (Sous, Oda, Folgpris, web, the repo). Keep
   the result short enough to read on a watch.

3. **Converse.** Call `mcp__Paperboy__ask_user`:
   - `device`: `pebble`
   - `question`: at most about 300 characters. Give a one- or two-line answer or status,
     then an **open-ended** follow-up question, such as "Want me to … or something
     else?" or "What should I do next with this?". Avoid yes/no questions unless you
     need confirmation before an action.
   - `timeoutSeconds`: `55`. The MCP client drops any call after 60 seconds, even though
     Paperboy allows up to 230. A longer timeout makes the call fail on the client side
     while the question stays open on the watch.

   **Late answers.** If the call fails with an MCP timeout, or a later `ask_user` returns
   `busy`, the earlier question is probably still on the watch. Long-poll
   `read_events(type="voice.input", since=<last cursor>, wait=30)` for up to about
   4 minutes in total. Treat a `voice.input` from `pebble` as the answer. Keep the cursor
   from your first `read_events` call so you don't pick up old answers.

   Then handle the result:
   - **Answer text**: treat it as the user's next instruction. Do that work, then go
     back to step 3 with a new answer and a new follow-up.
   - **`text` is null and `audio` is set** (stt is off): fetch the clip with
     `get_event_audio`. If you can't transcribe it, say so in the summary and end.
   - **`timeout`**: the user has left. End the conversation and go to step 4.
   - **`device_unreachable`, `voice_off`, `voice_unavailable`, `cancelled_by_user`**:
     end and go to step 4.
   - **`busy`**: an earlier question is still open. Wait for its answer with
     `read_events`, as described under "Late answers".
   - **`no_speech`**: the user pressed the button but said nothing that could be heard.
     Ask again once with a shorter question. If that also returns `no_speech`, end.
   - **The user ends it** ("that's all", "nei takk", "stopp"): go to step 4.

   Guard rails: stop after **10 turns**. If a follow-up would lead to a purchase, a
   deletion or a message to someone else, read the exact action back and ask for an
   explicit yes first.

4. **Close.** If the run has an `evt_…`, call `mcp__Paperboy__finish_task` with:
   - `summary`: one or two sentences covering what was asked, what was done and any
     open thread.
   - `page`: optional. Use it only when the result is too long for the watch. It is
     Sticky HTML in rem units with greys only (see `paperboy://guide/sticky`).

   If the conversation ended on a timeout with an open thread, also send a
   `PushNotification` so it isn't lost.

## Turn log

Keep a running log in the session: for each turn, the question asked, the answer
received and the action taken. Use it to build the `finish_task` summary.
