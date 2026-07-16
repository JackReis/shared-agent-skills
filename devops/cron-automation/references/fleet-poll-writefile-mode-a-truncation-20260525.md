# `write_file(mode='a')` Truncates Existing Log Files — 2026-05-25

## Symptom
A fleet-poll cron task attempted to append to an existing log file using `write_file(path="~/.hermes/logs/fleet-poll.log", mode="a", content=report)`. The file already contained ~60 lines of historical entries. After the tool call succeeded, `read_file` showed only **2 lines** (a header separator + timestamp line) — all prior history was destroyed.

## Expected Behavior
`mode='a'` should append content to the end of the file without altering existing data.

## Actual Behavior
The write_file tool appeared to treat `mode='a'` as a replacement operation, keeping only the final portion of the content (2 lines). The returned `"bytes_written"` was consistent with the full report size, but the file on disk was truncated.

## Immediate Workaround
When appending to an existing log file, **prefer shell append via `terminal()`**:
```bash
# Safe on this environment — verified in prior polls
echo "$content" >> ~/.hermes/logs/fleet-poll.log
```

If shell redirection is blocked by a security scanner, and `write_file(mode='a')` is the only remaining option, read the full file first, concatenate in-memory, and write with `mode='w'` (explicit replace):
```python
from hermes_tools import read_file, write_file
old = read_file(path="/path/to/log")["content"]
new = old + "\n" + report
write_file(path="/path/to/log", content=new)
```

## Verification Pattern
After any file modification that should preserve history:
1. `read_file` → check `total_lines` or `file_size`
2. Compare to expected count. If lines dropped, you overwrote instead of appended.

## Cross-Reference
- `references/fleet-poll-writefile-sibling-overwrite-20260525.md` — sibling-subagent cross-agent overwrite
- `references/fleet-poll-overwrite-writefile-20260525.md` — earlier `write_file` replace-only overwrite (no mode specified)
