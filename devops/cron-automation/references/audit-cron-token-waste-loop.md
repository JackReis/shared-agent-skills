# Audit Cron Token Waste Loop

## Discovered: 2026-06-01 (dialectical-fleet-audit v5-v8)

## Pattern

A cron job that **detects** problems but cannot **remediate** them runs on a high-frequency schedule (e.g., every 10 minutes). Over days/weeks, it burns model tokens producing identical reports while zero remediation occurs.

## Concrete Case

The `dialectical-fleet-audit` cron job (ID: `fbda357b007a`) was set to `*/10 * * * *` (144 runs/day). Over 9 days it ran 1,172+ times, each time reporting the same 8 CRITICAL + 5-6 HIGH findings. Total token cost was significant. Zero findings were remediated between any two consecutive runs.

## Root Cause

Audit/detection crons are **diagnostic tools**, not remediation tools. They produce findings that require human action. Running them faster than humans can act creates pure waste.

## Detection Signals

1. The cron output has a version number (v5, v6, v7, v8...) and each version is identical to the previous
2. A `completed` count in the hundreds with `last_status: ok` but no evidence of findings being acted on
3. The MEMORY.md entry for the audit hasn't changed between runs
4. The findings reference "unchanged since vN" or "zero remediated"

## Remediation

1. **Reduce frequency**: After 2-3 consecutive identical findings, reduce to `0 */4 * * *` (6×/day) or `0 */6 * * *` (4×/day)
2. **Add a circuit breaker**: If the cron can compare against its last output, include a `[SILENT]` response when findings are unchanged
3. **Pause until acted on**: For critical findings, pause the cron entirely until a human confirms remediation has started
4. **Separate detection from reporting**: Run detection at high frequency, but only deliver a report when findings change

## Cost Impact

With `glm-5.1:cloud` (or any cloud model), 144 runs/day × ~8K input tokens × ~4K output tokens = significant daily burn. Over 9 days at 1,172 runs, this was estimated at several hundred thousand tokens of pure waste.