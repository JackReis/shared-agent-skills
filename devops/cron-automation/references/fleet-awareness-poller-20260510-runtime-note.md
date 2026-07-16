# Fleet awareness poller runtime note (2026-05-10)

Observed in the live cron/runtime container:

- `HOME=/root`, `whoami=root`, `pwd=/root`
- Poll script exists at `/root/.hermes/cubby/scripts/poll-fleet-awareness.sh`
- The live monitored tree was absent in this runtime:
  - `~/.hermes/presence/` missing
  - `session-handoffs/` missing
  - coordination `README.md` missing
  - no git repo found in candidate roots

Verification pattern that worked:

1. Run the poll script directly.
2. Confirm the output ends with `Poll complete`.
3. Check `/root/.hermes/cubby/runs/fleet-awareness-poller.log` for a fresh timestamp.
4. Check `/root/.hermes/cubby/runs/fleet-awareness-poller.state.json` for the latest snapshot.
5. Do not trust pidfiles alone; stale `*.pid` files may remain even when `ps` shows nothing.

Useful outputs seen in this session:

- `Active agents: presence directory not found`
- `Handoffs: session-handoffs directory not found`
- `Coordination README: not found`
- `Git status: no git repo found in candidate roots`

The monitor was silent because the runtime lacked the coordination tree, not because the poller failed.