# Wrangler multi-account wrapper + SOPS dotenv quirks — 2026-06-14

## Trigger

Use this reference when a Cloudflare Workers deployment or secret step depends on a local multi-account Wrangler wrapper and SOPS-encrypted dotenv files.

## Lessons

### 1. `cf-wrangler` may not be the account wrapper

On Jack's Mac, two different things can be named `cf-wrangler`:

- `/opt/homebrew/bin/cf-wrangler` — Wrangler's internal delegate binary. It only accepted `dev` in this session and returned `Usage: cf-wrangler dev [args]` for account-style commands.
- `~/.hermes/bin/cf-wrangler` — Jack's multi-account wrapper. It accepts account aliases like `osrh` and `jack`, loads Cloudflare token/account IDs from SOPS, then execs `wrangler`.

**Rule:** For account-specific Cloudflare work, call the wrapper explicitly:

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
~/.hermes/bin/cf-wrangler osrh whoami
~/.hermes/bin/cf-wrangler osrh secret list
```

Do not trust `command -v cf-wrangler` when the account alias matters.

### 2. `SOPS_AGE_KEY_FILE` must be exported for headless SOPS decrypts

If `sops -d ~/.secrets/llm-providers.env` fails with no usable master key, first try:

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
sops -d ~/.secrets/llm-providers.env >/dev/null
```

This is a decrypt setup fix, not a secret value fix.

### 3. `sops exec-env` can silently produce empty vars from bad dotenv shape

In this session, the decrypted `llm-providers.env` had:

- `export KEY=...` prefixes
- duplicate identical `OLLAMA_API_KEY` lines

`SOPS_AGE_KEY_FILE` made `sops -d` succeed, but `sops exec-env` still produced missing env vars. The safe repair was:

1. Back up the encrypted file.
2. Decrypt without printing values.
3. Rewrite lines as plain `KEY=value`.
4. Drop exact duplicate keys only when the value is identical.
5. Re-encrypt using `--filename-override` so SOPS uses the original file's creation rule.
6. Verify with presence checks, not values.

Skeleton:

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
FILE="$HOME/.secrets/llm-providers.env"
cp "$FILE" "$FILE.bak-pre-repair-$(date +%Y%m%d-%H%M%S)"

python3 repair-dotenv-shape.py \
  | sops --input-type dotenv --output-type dotenv \
      --filename-override "$FILE" \
      --encrypt /dev/stdin > "$FILE.tmp"
mv "$FILE.tmp" "$FILE"
chmod 600 "$FILE"

sops exec-env "$FILE" 'python3 - <<"PY"
import os
for k in ["OLLAMA_API_KEY", "OPENROUTER_API_KEY"]:
    print(k + "_PRESENT=" + ("yes" if os.environ.get(k) else "no"))
PY'
```

Never print token values, lengths, prefixes, or hashes in user-visible output unless explicitly needed for a secret-shape verifier; prefer `PRESENT=yes/no`.

## Verifier checklist

| Check | Command shape | Expected |
|---|---|---|
| Correct wrapper | `~/.hermes/bin/cf-wrangler osrh whoami` | Authenticated account row for the expected account ID |
| Secrets present | `~/.hermes/bin/cf-wrangler osrh secret list` | Secret names only; no values |
| SOPS decrypt | `SOPS_AGE_KEY_FILE=... sops -d FILE >/dev/null` | exit 0 |
| dotenv exec-env | `sops exec-env FILE 'python3 presence_check.py'` | each required key reports `PRESENT=yes` |
