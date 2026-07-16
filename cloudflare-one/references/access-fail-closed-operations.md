# Cloudflare Access fail-closed operations

Use this runbook when putting a public-hostname origin behind Cloudflare Access without allowing an unprotected cutover window.

## Security-system paths

### 1. Scoped API credential from Bitwarden

Prefer an existing Cloudflare API token with the minimum account permission required by the current endpoint schema.

- Keep `BW_SESSION` out of process arguments, shell history, logs, and assistant output. Consume it from an inherited environment or a mode-`0600` file and delete the file immediately after use.
- Verify candidates with `GET /client/v4/user/tokens/verify`, but do not equate `200` there with authorization for Access.
- Access application and policy endpoints require account-level **Access: Apps and Policies Write** (the dashboard may label the permission **Edit**).
- A valid token limited to Workers, Pages, and KV typically returns `403` / Cloudflare code `10000` for `/accounts/{account_id}/access/apps`.
- Never print token values while identifying or testing credentials. Return only source label, length, validation status, API status, and Cloudflare error codes.

### 2. Official Cloudflare API MCP OAuth fallback

If no stored token has Access authority, use the official Cloudflare API MCP rather than broadening an unrelated token or weakening Keychain ACLs:

```bash
codex mcp add cloudflare-api --url https://mcp.cloudflare.com/mcp
codex mcp login cloudflare-api
```

Preserve a timestamped `~/.codex/config.toml` backup before adding the server.

#### Keychain pitfall

An old `Codex MCP Credentials` item can remain ACL-bound to a removed executable such as `/Applications/Codex.app/Contents/Resources/codex`. `security find-generic-password` may then show a macOS approval dialog but still fail (observed exit status `24`). Do **not** weaken the item's ACL or extract the encrypted Keychain database. Re-register the official MCP and complete OAuth with the current Codex CLI instead.

#### MCP execution pitfall

The MCP `search` tool can retrieve the live OpenAPI schema, while the generic `execute` tool performs account API calls. In noninteractive `codex exec`, `execute` may be cancelled as `user cancelled MCP tool call` even with `approval_policy=never`.

For an explicitly authorized, tightly bounded call, the proven fallback is:

```bash
codex exec \
  --dangerously-bypass-approvals-and-sandbox \
  --skip-git-repo-check \
  --ephemeral \
  -C "$HOME" \
  --json \
  '<bounded prompt>'
```

Because this disables Codex safeguards, all of these controls are mandatory:

1. Restrict the prompt to `cloudflare-api` MCP for Cloudflare operations.
2. Name the exact account, hostname, app, policy, and allowed identity.
3. Forbid browser use, shell commands, and local-file modification in the prompt.
4. Retrieve the current endpoint schema before constructing request bodies.
5. Read current state immediately before mutation; abort on duplicates, unexpected policies, or API errors.
6. Make the smallest idempotent mutation. Never delete an app or policy as an automatic repair.
7. Follow the mutation with a fresh independent readback and direct HTTP verification.
8. Never expose OAuth or API tokens in prompts or output.

## Fail-closed Access cutover

### Preconditions

- The origin is healthy on a private path.
- The public hostname's tunnel ingress is temporarily `http_status:403`.
- The tunnel configuration validates and its service manager is healthy.
- The account has an identity provider appropriate for the policy. One-time PIN is sufficient for an exact-email allow rule.

### Procedure

1. **Keep the public origin closed.** Confirm public `/`, health, and sensitive API routes return the tunnel's static denial.
2. **Verify private health.** Confirm the connector host and any private/Tailscale forward return `200` from the real origin.
3. **Inventory Access.** Exact-domain list the hostname. Abort if more than one app matches.
4. **Create or inspect the app.** For a browser app, use `type: self_hosted`; set the exact domain, session duration, launcher visibility, and allowed identity provider. `auto_redirect_to_identity: true` is valid only when exactly one IdP is allowed. Keep `options_preflight_bypass: false` unless explicitly required.
5. **Inspect all app policies.** Abort on `bypass`, `non_identity`, `everyone`, broad groups/domains, or other unexpected selectors.
6. **Create the minimum allow policy.** Prefer one exact email or a verified IdP group; use precedence `1`; keep `exclude` and `require` empty unless requirements say otherwise.
7. **Fresh API readback.** Require one exact-domain app, the intended settings, and exactly the intended policies. All calls must return 2xx, `success: true`, and `errors: []`.
8. **Prove Access interception before opening the origin.** Anonymous requests to `/` and sensitive API routes must redirect to the account's `/cdn-cgi/access/login/<hostname>` surface. Spoofed identity headers must still redirect. Follow the redirect only far enough to prove the login page loads.
9. **Open only the intended ingress.** Back up the tunnel config, replace the hostname's `http_status:403` service with the private origin, validate the custom config, and restart its service manager. If validation or restart fails, restore the backup and restart the fail-closed config.
10. **Post-cutover verification.** Repeat anonymous redirect, spoofed-header denial, login-page load, private-origin health, and connector-origin health tests.
11. **Authorized verification.** Have the user complete authentication. Query `GET /accounts/{account_id}/access/logs/access_requests` with exact email and time bounds; require a matching app UID/domain record with `action: login` and `allowed: true`.

### Useful commands

Validate a local tunnel config:

```bash
cloudflared tunnel --config /path/to/config.yml ingress validate
```

Restart a macOS LaunchAgent after validation:

```bash
launchctl kickstart -k "gui/$(id -u)/<launchd-label>"
```

Do not place command substitution in a Hermes terminal argument when the environment mangles `$()`; compute the UID separately or use a literal value discovered from `id -u`.

### HTTP checks

Use a no-redirect client and require:

- Anonymous `/`, health, and sensitive API paths: `302` to the account's Cloudflare Access login host.
- Spoofed `Cf-Access-Authenticated-User-Email` / forwarded-email headers: still `302`.
- Access login surface: `200` with the expected IdP form.
- Private/Tailscale origin and connector loopback: `200`.

Do not enter passwords, API keys, or one-time codes on the user's behalf. The user completes identity authentication; confirm it through Access authentication logs.

## Rollback

At any failure after the Access mutation:

1. Restore the tunnel ingress to `http_status:403`.
2. Validate the config.
3. Restart cloudflared and confirm public denial.
4. Leave the Access app/policy intact for inspection unless the user explicitly approves deletion.

## Verified example: Paperclip, 2026-07-16

- Hostname: `paperclip.jack.digital`
- Access app: `7bd4df8b-9c47-4e8c-bfc9-b531cd7a7976`
- Policy: `a5926f1c-d445-4bd6-88d6-b443f50b13f0`
- Identity: exact email `jackareis@gmail.com` through one-time PIN
- App settings: self-hosted, 24-hour session, launcher hidden, one IdP, auto-redirect enabled, preflight bypass disabled
- Tunnel gate: `http_status:403` until app/policy readback and anonymous login redirect both passed
- Final tunnel origin: `http://127.0.0.1:3100`
- Verification: anonymous and spoofed-header requests redirected to Access; public login form loaded; private and connector origins returned `200`; Access log recorded `action: login`, `connection: onetimepin`, `allowed: true`.
