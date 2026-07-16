# Reference: set-u-cron-home-unbound-mac-failure-20260525
## Cron environment strips HOME, causing `set -u` to abort the script before any logging occurs

**Symptom:** Cron appears to run (`fleet_awareness.log` shows timestamps at expected 5-minute intervals), but every line shows only:
- presence: OK (reads `/Users/jack.reis/...` absolute path, no `$HOME` used)
- handoffs: EMPTY
- README: OK (fixed absolute path, no `$HOME` used)
- git: `branch=?, 0 uncommitted file(s)` (git silently fails)

The crontab redirect log `fleet_awareness_cron.log` is **0 bytes** (completely empty) because the script dies before writing anything to it.

**Root cause:** macOS cron strips `$HOME` from the environment. The script uses `set -uo pipefail` (unbound variables are fatal). The first use of `$HOME` is at line 19 for `PRESENCE_DIR="$HOME/.hermes/presence"`. The script immediately exits with:
```
/Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh: line 19: HOME: unbound variable
```
The redirect log captures this error — BUT only if the redirect itself works. In this case, the error was somehow NOT captured in the redirect log (0 bytes), possibly because the shell itself failed during setup.

**Verification:**
Simulate the exact failure:
```bash
env -i PATH=/Library/Developer/CommandLineTools/usr/bin:/usr/local/bin:/usr/bin:/bin \
  /usr/bin/lockf -t 0 ~/.hermes/cubby/logs/.fleet-awareness.lock \
  /bin/bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh \
  >> /tmp/cron_test.log 2>&1
# Contents: "... line 19: HOME: unbound variable"
```
With `HOME=/Users/jack.reis` added to the environment, the same command succeeds completely.

**Fix (two layers of defense):**
1. **Crontab:** Explicitly set `HOME` in the cron line:
   ```cron
   */5 * * * * PATH=/... HOME=/Users/jack.reis /usr/bin/lockf ...
   ```
2. **Script (defensive):** Add a fallback `HOME="${HOME:-/Users/jack.reis}"` immediately after `set -u`, before any `$HOME` usage. This guards against any future environment where `HOME` might also be missing (e.g., custom launchd, systemd timers, CI containers).

**Key lessons:**
- `set -u` in scripts that run under cron is dangerous if any variable expansion relies on environment variables
- Always simulate cron's stripped environment (`env -i PATH=... /bin/bash script`) when debugging cron failures
- A partially working script (some absolute paths working, others failing) is a strong signal of environment variable issues under `set -u`
- An empty redirect log with `set -u` scripts often means the script died so early that even stdout/stderr capture failed to complete
