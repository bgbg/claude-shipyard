---
description: Execute the macOS 'say' command to speak text aloud. When composing the spoken text yourself (e.g. progress updates), keep it to 15 words or fewer unless the user requests otherwise.
---

# Say Command

Execute the macOS `say` command to speak the provided text aloud.

## Behavior

When invoked with `/say <text>`, extract the text from the command arguments and execute:
```bash
say "<text>"
```

The text should be passed directly to the `say` command. If no text is provided, inform the user that text is required.

## Message length

Unless the user requests otherwise, keep spoken messages to 15 words or fewer. Spoken alerts should be short and to the point; trim or summarize longer text rather than reading it verbatim.

## Examples

- `/say Hello, world!` - Speaks "Hello, world!"
- `/say Task completed` - Speaks "Task completed"
- `/say Testing one two three` - Speaks "Testing one two three"

## Notes

- Works on macOS systems only (requires macOS `say` command)
- Text is passed as a single argument to `say`
