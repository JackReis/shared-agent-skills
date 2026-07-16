# Fleet Awareness Poller: Runtime Mismatch (2026-05-10)

Observed in a cron-style fleet poll session:

- Live container ran as `root` with `HOME=/root`.
- Canonical coordination sources were absent in the container:
  - `~/Documents/Coordination/README.md`
  - `~/Documents/=notes/INFRASTRUCTURE-TASKS.md`
  - `claude/mcp-coordination/state/session-handoffs/`
- Local poller artifacts still existed under `~/.hermes/cubby/`:
  - `scripts/poll-fleet-awareness.sh`
  - `runs/fleet-awareness-poller.log`
  - `runs/fleet-awareness-poller.state.json`
- Poll state showed null sources:
  - `git: null`
  - `handoffs: null`
  - `presence: null`
  - `readme: null`
- Poll log repeatedly reported:
  - `presence directory not found`
  - `session-handoffs directory not found`
  - `Coordination README: not found`
  - `no git repo found in candidate roots`

Follow-up observed later in the same class of run:
- The poll state stayed all-null but advanced to a fresh baseline timestamp (`2026-05-10T06:59:26+00:00`).
- The log still showed the same missing-tree signature, so the correct interpretation was "unmounted coordination tree / no live delta", not "confirmed silence from the monitored files".

Operational lesson:
- Do not treat missing coordination paths as a clean no-change result.
- First reconstruct context from prior session history with `session_search` and compare against known canonical paths.
- If the state file is all-null, treat it as a baseline marker until the live tree is actually reachable.
- Only then decide whether the right final response is a substantive report or `[SILENT]`.
