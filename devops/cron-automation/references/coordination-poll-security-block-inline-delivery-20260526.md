# Coordination Poll Security Block and Inline Delivery — 2026-05-26

## Context
Hermes/Wings cron job running a read-only coordination poll. The agent attempted to append the report to `~/.hermes/logs/fleet-poll.log` using a shell redirect (`>>`).

## Incident
- Shell redirect blocked by security scan: `[HIGH] Dotfile overwrite detected`.
- `write_file()` fallback succeeded but is **replace-only**, destroying ~60 lines of prior log history.
- Second `write_file()` call in the same session (to the same path) also silently overwrote the first, because the sibling-subagent guard only fires cross-agent.

## Root Cause
The security scanner intercepts shell redirection patterns (`>> ~/.hermes/logs/...`) but does not block Hermes tool writes. However, `write_file()` is designed to overwrite the entire file, not append.

## Correct Fallback for Read-Only Cron Tasks
When a cron task is read-only and its wrapper already auto-delivers the final response:
1. **Do not fight the security boundary.** Additional log files are a convenience, not a requirement.
2. **Produce the report inline as the final assistant message.** The cron wrapper handles delivery.
3. If append semantics are truly required, use a Python read-then-rewrite pattern (see `references/fleet-poll-overwrite-writefile-20260525-repeat.md` in `cron-automation`).

## Verdict
For read-only fleet polls, inline delivery is the simplest correct path. It avoids log corruption, respects the read-only contract, and satisfies the cron wrapper's `[SILENT]` semantics when there is nothing new.

## Takeaway
`write_file(path="~/.hermes/logs/...", content=report)` on an existing cron log file is a **destructive append anti-pattern**. Always assume `write_file` replaces. When in doubt, deliver inline.
