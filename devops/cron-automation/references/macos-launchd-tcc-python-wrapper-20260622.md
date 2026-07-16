# macOS launchd TCC Python Wrapper Workaround

## Date
2026-06-22

## Problem
On macOS Sequoia (15.7.5), a launchd plist that runs `/bin/bash /path/to/restricted/dir/script.sh` fails with exit code 126:
```
shell-init: error retrieving current directory: getcwd: cannot access parent directories: Operation not permitted
/bin/bash: /Users/jack.reis/Documents/=notes/claude/scheduled-tasks/vault-ingest/run-obn-sync.sh: Operation not permitted
```

The `=notes` directory (and likely other `~/Documents/` subdirectories) is TCC-restricted. `/bin/bash` when launched by launchd does not have Full Disk Access to these directories.

## Diagnosis
- `launchctl list | grep <label>` shows exit code 126
- The script runs fine when executed manually from Terminal
- `xattr -l` shows `com.apple.provenance` on the script file
- Clearing xattrs does NOT fix the issue — the TCC restriction is on `/bin/bash` itself, not the script

## Fix: Python Wrapper
The venv Python binary (`/Users/jack.reis/Documents/=notes/.venv/bin/python3`) already has TCC permission because the user ran it from Terminal. Use it as the launchd executor:

### Wrapper script (`~/.local/bin/obn-sync-launchd.py`)
```python
#!/usr/bin/env python3
import subprocess, sys, os
os.chdir("/Users/jack.reis")  # safe directory
result = subprocess.run(
    ["/bin/bash", "/Users/jack.reis/Documents/=notes/claude/scheduled-tasks/vault-ingest/run-obn-sync.sh"],
    capture_output=False, timeout=3600,
)
sys.exit(result.returncode)
```

### Plist ProgramArguments
```xml
<key>ProgramArguments</key>
<array>
    <string>/Users/jack.reis/Documents/=notes/.venv/bin/python3</string>
    <string>/Users/jack.reis/.local/bin/obn-sync-launchd.py</string>
</array>
<key>WorkingDirectory</key>
<string>/Users/jack.reis</string>
```

### Installation
```bash
chmod +x ~/.local/bin/obn-sync-launchd.py
xattr -cr ~/.local/bin/obn-sync-launchd.py
cp com.jackreis.obn-sync.plist ~/Library/LaunchAgents/
xattr -cr ~/Library/LaunchAgents/com.jackreis.obn-sync.plist
launchctl bootout gui/$(id -u)/com.jackreis.obn-sync 2>/dev/null || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.jackreis.obn-sync.plist
```

## Verification
```bash
launchctl list | grep obn-sync
# Should show: <PID>  0  com.jackreis.obn-sync
```

## Permanent Fix
Add `/bin/bash` to **System Settings > Privacy & Security > Full Disk Access**. This requires user interaction and cannot be done programmatically.

## Scope
This affects ALL launchd jobs that need to access `~/Documents/` subdirectories via `/bin/bash`. The Python wrapper approach works because the Python binary inherits TCC permissions from its first interactive use.