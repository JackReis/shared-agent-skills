# Detached Launcher Fallback for Missing Cron

Session note for environments where a recurring read-only monitor is requested but `crontab` is unavailable.

## What worked here
- `crontab` was absent in the runtime.
- A plain shell background loop did not prove durable enough to trust.
- Launching a child process via Python `subprocess.Popen(..., start_new_session=True)` created a detachable monitor that could be verified independently.

## Verification pattern
1. Start the recurring loop from a small launcher process.
2. Write the child PID to a pidfile under `~/.hermes/cubby/runs/`.
3. Confirm the PID with `ps`.
4. Confirm the poller log is growing, not just that the launch command returned success.

## Notes
- Do not use shell-level `nohup`, `disown`, or `setsid` wrappers in foreground commands when the tooling forbids them.
- Treat `terminal(background=true)` as a launch mechanism only; it is not proof that the loop survives.
- If the runtime lacks cron, report the limitation clearly unless a detached monitor is explicitly verified.
