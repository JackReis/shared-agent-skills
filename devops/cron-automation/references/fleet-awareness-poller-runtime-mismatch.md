# Fleet Awareness Poller Runtime Mismatch

Session note: the live container for a cron poll may not contain the user-facing coordination tree even when prior sessions reference it.

Observed facts from this session:
- `HOME` in the active container was `/root`.
- Canonical user paths such as `/root/Documents/Coordination/README.md` and `/root/Documents/=notes/INFRASTRUCTURE-TASKS.md` were absent.
- `/root/.hermes/cubby/scripts/poll-fleet-awareness.sh` existed and logged repeated polls to `/root/.hermes/cubby/runs/fleet-awareness-poller.log`.
- The poller state file showed null sources:
  - `git: null`
  - `handoffs: null`
  - `presence: null`
  - `readme: null`
- The log repeatedly reported:
  - `presence directory not found`
  - `session-handoffs directory not found`
  - `Coordination README: not found`
  - `no git repo found in candidate roots`

Operational takeaway:
- If the expected coordination tree is missing, do not treat that as a clean no-change result without cross-checking prior session history.
- Use `session_search` to reconstruct earlier polls and known canonical paths before reporting silence.
