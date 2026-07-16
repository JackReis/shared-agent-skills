# Fleet awareness silent poller: TCC/read-baseline mismatch (2026-05-11)

## Situation
A scheduled fleet-awareness cron body requested `bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh` with no Discord notifications and `[SILENT]` when nothing material changed.

The installed crontab used macOS `lockf` correctly:

```cron
*/5 * * * * /usr/bin/lockf -t 0 /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.lock /bin/bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh >> /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.log 2>&1
```

The script-owned artifacts were fresh:

- `~/.hermes/logs/fleet-awareness.log`
- `~/.hermes/logs/.fleet-awareness-state`

## Observed pattern
Cron ticks sometimes logged:

- `Handoff directory exists but is not readable`
- `Coordination README exists but is not readable`
- `Presence changed since last check`
- `Git status changed since last check`

A manual locked run from the agent context could read the same handoff tree and README successfully:

- handoffs: 264
- Coordination README: 231 lines
- active agents: 2 (`codex`, `hermes`)
- hermes-agent git status: 5 changed paths

This means some log "changed" lines were artifacts of running the same script from contexts with different read permissions/baselines. The manual run updates the state file to the readable baseline; the next cron tick may update it back to a denial/unreadable baseline and log spurious changes.

## Handling rule
For silent read-only monitors, do not escalate these as user-facing findings when all of the following are true:

1. The schedule exists and uses the expected lock + absolute script path.
2. The script exits 0 under the same lock wrapper.
3. Script-owned log/state timestamps are fresh.
4. Direct inspection shows stable fleet content (same agent count, handoff count, README line count, expected git status count).
5. The only anomalies are known macOS cron/TCC read denials or baseline hash flips.

Return exactly `[SILENT]` in that case. Report only if there is a new material fleet change, a scheduler/log freshness failure, or a new operational regression not explained by the permission-context mismatch.

## Pitfall
Manual verification can perturb the state file because the agent context may see files cron cannot. If you need to inspect direct content, separate "material content changed" from "state hash changed after a manual readable run." Prefer fresh timestamps and direct content counts over hash-change lines alone.