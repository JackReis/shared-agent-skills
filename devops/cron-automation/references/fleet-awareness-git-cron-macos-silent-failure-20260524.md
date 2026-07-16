# Fleet awareness poller: macOS cron silently breaks git via xcrun shim (2026-05-24)

## Scenario

A fleet-awareness poll script runs every 5 minutes via cron on macOS:

```cron
*/5 * * * * /bin/bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh \
  >> /Users/jack.reis/.hermes/cubby/logs/fleet_awareness_cron.log 2>&1
```

The script is self-contained and read-only. It writes its own structured log and
exits 0. The script contains these lines:

```bash
branch=$(git -C "$VAULT" branch --show-current 2>/dev/null || echo "?")
dirty=$(git -C "$VAULT" status --porcelain 2>/dev/null | wc -l | tr -d ' ')
```

## Observation

- **Interactive run:** `bash poll-fleet-awareness.sh` (same script, same paths):
  - `branch=main, 7 uncommitted file(s)`
- **Cron-log entries (script-owned log):**
  - `branch=?, 0 uncommitted file(s)`
  - No stderr captured because redirected via `2>/dev/null`
- **No errors in cron redirect log** — `git` does not crash; it simply produces
  empty/wrong output silently.
- **Script exits 0**, cron fires on schedule, `crontab -l` confirms the entry.

## Root Cause

On macOS, `/usr/bin/git` is an `xcrun` shim, not the real binary. Under cron's
minimal environment, `xcrun` fails to resolve the developer tools path, so `git`
returns empty output or exits non-zero. Because the script suppresses stderr
with `2>/dev/null || echo "?"`, the failure is completely silent.

The same issue affects any tool installed via Xcode Command Line Tools that relies
on `xcrun` shimming: `clang`, `make`, `swift`, `xcodebuild`, etc.

## Impact

- Git-dependent monitoring (branch, dirty count, status) silently underreports
  under cron even though:
  - The repository exists
  - The script works interactively
  - No stderr is emitted
- Other operators (or future agents) inspecting only the cron log may see
  `branch=?, 0 uncommitted file(s)` and assume the vault is clean or not a git
  repo, when it is neither.

## Verification Recipe

1. **Check the cron-log vs interactive output side-by-side:**
   ```bash
   # Interactive (your current shell environment)
   bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
   tail -3 ~/.hermes/cubby/logs/fleet_awareness.log

   # Simulated cron environment
   env -i HOME=/Users/jack.reis SHELL=/bin/bash PATH=/usr/bin:/bin \
     bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
   tail -3 ~/.hermes/cubby/logs/fleet_awareness.log
   ```
2. **Look for the pattern:**
   - Interactive: `branch=main, N uncommitted file(s)`
   - Simulated/minimal: `branch=?, 0 uncommitted file(s)`
3. **Confirm xcrun behavior:**
   ```bash
   env -i HOME=/Users/jack.reis /usr/bin/git --version
   ```
   May show `xcode-select: error:` or similar, or may silently return empty.

## Fixes

1. **Use the real git path.** Find the path that xcrun resolves interactively:
   ```bash
   xcrun --find git
   ```
   Then hardcode that in the script (e.g., `/Applications/Xcode.app/Contents/Developer/usr/bin/git`
   or `/Library/Developer/CommandLineTools/usr/bin/git`).

2. **Set `PATH` in the cron line** to include the developer tools bin directory:
   ```cron
   */5 * * * * PATH=/usr/local/bin:/usr/bin:/bin:/Applications/Xcode.app/Contents/Developer/usr/bin ... bash ...
   ```

3. **Explicitly source xcrun initialization** in the script before calling git:
   ```bash
   if [ "$(uname)" = "Darwin" ]; then
       # Initialize Developer PATH for cron contexts
       if [ -x "/usr/libexec/path_helper" ]; then
           eval $(/usr/libexec/path_helper -s)
       fi
       # Or explicitly add developer tools to PATH
       if [ -d "/Library/Developer/CommandLineTools/usr/bin" ]; then
           export PATH="/Library/Developer/CommandLineTools/usr/bin:$PATH"
       fi
   fi
   ```

## Takeaway

On macOS, the presence of `/usr/bin/git` in cron does **not** guarantee that git
works. The `xcrun` shim can silently fail to resolve the real binary in minimal
environments. When a script works interactively but produces silent failures under
cron, suspect xcrun/Developer Tools path issues.

Always compare interactive output with simulated minimal-environment output
(`env -i HOME=... SHELL=... PATH=...`) when debugging cron tool failures on macOS.
