# Fail-closed hostname decommission for Cloudflare Tunnel / Zero Trust

Use this when a hostname must stop serving immediately, but DNS-record or Zero Trust Tunnel permissions are missing or delayed.

## Pattern

1. **Verify live behavior first**
   - Check the forbidden hostname directly with a cache-busting query string.
   - Check a known-good canonical hostname in the same sweep so you do not break the service while decommissioning aliases.

2. **Try authoritative cleanup**
   - Preferred: remove the DNS record, Worker custom domain, Pages custom domain, or Zero Trust public hostname route.
   - If using the Cloudflare API, distinguish capabilities:
     - Zone lookup may succeed while DNS record reads/edits return `403 Authentication error`.
     - Tunnel config reads/edits may also need separate account/Zero Trust permissions.
   - Do not conclude a token is generally valid just because `/zones?name=...` succeeds.

3. **If authoritative cleanup is blocked, fail closed at the serving layer**
   - Add an application guard that rejects forbidden host markers with 404/410.
   - Check all likely host markers, not only `Host`:
     - `Host`
     - `X-Forwarded-Host`
     - `X-Original-Host`
     - RFC `Forwarded: host=...`
   - Cloudflare Tunnel/local origins may present `Host: localhost:<port>` while preserving the requested public hostname in a forwarded header.

4. **If the hostname is in a local `cloudflared` config, map it to an explicit status**
   - Example:
     ```yaml
     ingress:
       - hostname: forbidden.example.com
         service: http_status:404
       - service: http_status:404
     ```
   - Back up the config before editing.
   - Restart only the relevant connector/service.

5. **Verify both external and origin-level behavior**
   - External:
     ```bash
     curl -sS -L --max-time 12 -H 'Cache-Control: no-cache' \
       -o /tmp/body -w '%{http_code}\n' \
       "https://forbidden.example.com/?offline_check=$(date +%s)"
     ```
   - Origin/header simulation:
     ```bash
     curl -sS --max-time 8 \
       -H 'Host: localhost:8080' \
       -H 'X-Forwarded-Host: forbidden.example.com' \
       -o /tmp/body -w '%{http_code}\n' \
       http://127.0.0.1:8080/
     ```
   - Canonical hostname must still return healthy 200.

## Pitfalls

- A public 200 after a local `Host` guard often means Cloudflare Tunnel is forwarding the original public hostname in `X-Forwarded-Host` or `Forwarded`, not in `Host`.
- A `HEAD` request may return 405 even when `GET` returns the actual page/status. Use `GET` for decommission verification unless the service explicitly supports `HEAD`.
- This pattern is mitigation, not full cleanup. Record the remaining Cloudflare DNS / custom-domain / Zero Trust deletion work for a better-scoped token or dashboard session.
