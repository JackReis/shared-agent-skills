# Fleet Awareness Poller Runtime Notes

Session-specific notes for the silent fleet awareness poller.

## What happened
The poller was invoked as:

```bash
bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
```

In this runtime, the command failed with:

```text
bash: /root/.hermes/cubby/scripts/poll-fleet-awareness.sh: No such file or directory
```

## Practical takeaway
Before treating a scheduled poll as broken, verify the live execution environment:

```bash
uname -a && whoami && pwd && printf 'HOME=%s\n' "$HOME"
```

Then search for the script in the actual container/workspace rather than assuming the authoring machine's `~/.hermes` layout exists here.

## Best practice for future runs
- Prefer absolute paths once confirmed.
- If the script is absent, locate or recreate it before scheduling.
- Keep the poll silent: no notifications, just log capture.
