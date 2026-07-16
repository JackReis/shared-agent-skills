# Fleet heartbeat script path mismatch

Session takeaway:
- Requested command: `bash ~/.hermes/cubby/scripts/heartbeat-update.sh`
- Runtime was a Linux container with `HOME=/root`
- The requested script did **not** exist at `/root/.hermes/cubby/scripts/heartbeat-update.sh`
- A filesystem search also found no `heartbeat-update.sh` anywhere in the container
- Nearby scripts existed, including:
  - `/root/.hermes/cubby/scripts/run-fleet-awareness-monitor.sh`
  - `/root/.hermes/cubby/scripts/poll-fleet-awareness.sh`

Rules learned:
- Do not assume the requested heartbeat filename exists just because related fleet scripts do.
- Search the live runtime before substituting a nearby script.
- If the requested script is absent, report the mismatch clearly rather than claiming the heartbeat ran.
- Use `session_search` to recover prior known-good locations before deciding the absence is real.