# Personal assistant

This repo drives a personal-assistant routine. A run starts in one of two ways:

- **API trigger**: the fire text has a task or question in it. It usually comes from a
  Paperboy `claude_task` action and names an `evt_…` ID.
- **Schedule**: the fire text is just the standing prompt, with no task in it.

Every run does this:

1. **Classify the trigger.** Read the fire text. Note any `evt_…` ID and the device that
   started the task. If there is no task, check for pending action items (see
   "Scheduled runs" below). If there are none, end the run quietly.
2. **Pick the flow** from `flows/` that matches the trigger and follow it.
3. **Close the loop.** If the run came from a Paperboy task, end with
   `mcp__Paperboy__finish_task` on that `evt_…` ID. Call it exactly once, even when the
   run failed or the user stopped answering.

## Flows

| Flow | When | File |
|------|------|------|
| Pebble conversation | A trigger brings a question or task and the user is reachable on the Pebble | `flows/pebble-conversation.md` |

## Devices (Paperboy)

- `pebble`: Pebble watch. It has voice; answers come back as text from speech-to-text
  on the phone. This is the main conversation channel.
- `sticky-…`: e-ink Sticky screen. Use it for longer answers through `finish_task`
  `page` or `quick_page`. It is often offline (`linkUp: false`).

Call `mcp__Paperboy__get_devices` to check whether a device is reachable before
starting a conversation.

## Other tools at hand

- **Sous**: the household's weekly food cycle. Follow its own instructions: call
  `week_status` first and speak Norwegian.
- **Oda**: the grocery cart. Never check out or pay.
- **Folgpris**: price tracking.

## Scheduled runs (no task in the prompt)

Nothing is wired up yet. Later this will check for pending action items, such as unread
Folgpris price drops or the Sous `week_status` next step. Until then, end the run
without sending a notification.

## Rules

- Git: never use branches. Commit and push directly to `main`.
- Never take outward-facing or irreversible actions (purchases, checkouts, deletions,
  messages to other people) on the strength of a spoken answer alone. Read the action
  back on the watch and get an explicit "yes" first.
- Spoken answers are speech-to-text and may be garbled. If an answer is ambiguous,
  ask again rather than guess.
