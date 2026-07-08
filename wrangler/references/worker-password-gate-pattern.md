# Cloudflare Worker Password-Gate Pattern

A reusable pattern for serving a static site behind a server-side password gate
on a custom domain. No SSO, no accounts — single shared password, HMAC-signed
HttpOnly cookie, 30-day expiry.

## When to use

- Internal tools, dashboards, wikis that need a shared password but not user accounts
- Sites where you already control the domain via Cloudflare DNS
- When you want the gate enforced at the edge (content never reaches the browser unauthenticated)

## Architecture

```
Browser → Cloudflare Worker (custom domain) → password check → ASSETS binding
```

The Worker intercepts all requests. Unauthenticated requests get a login page.
POST to `/__auth` with the correct password sets a signed HttpOnly cookie.
Subsequent requests with a valid cookie pass through to `env.ASSETS.fetch(req)`.

## Worker code (worker.js)

```javascript
const COOKIE = 'site_auth';
const MAX_AGE = 60 * 60 * 24 * 30; // 30 days

function enc(s) { return new TextEncoder().encode(s); }
function hex(buf) {
  return [...new Uint8Array(buf)].map((b) => b.toString(16).padStart(2, '0')).join('');
}

async function hmac(secret, msg) {
  const key = await crypto.subtle.importKey('raw', enc(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
  return hex(await crypto.subtle.sign('HMAC', key, enc(msg)));
}

async function makeToken(secret) {
  const exp = String(Math.floor(Date.now() / 1000) + MAX_AGE);
  return `${exp}.${await hmac(secret, exp)}`;
}

async function validToken(token, secret) {
  if (!token || token.indexOf('.') < 0) return false;
  const [exp, sig] = token.split('.');
  if (!exp || !sig) return false;
  if (Number(exp) < Math.floor(Date.now() / 1000)) return false;
  const expect = await hmac(secret, exp);
  if (sig.length !== expect.length) return false;
  let diff = 0;
  for (let i = 0; i < sig.length; i++) diff |= sig.charCodeAt(i) ^ expect.charCodeAt(i);
  return diff === 0;
}

function readCookie(req, name) {
  const raw = req.headers.get('Cookie') || '';
  for (const part of raw.split(';')) {
    const [k, ...v] = part.trim().split('=');
    if (k === name) return v.join('=');
  }
  return null;
}

function loginPage(error) {
  const msg = error ? `<p class="err">Incorrect password — try again</p>` : '';
  return new Response(
    `<!doctype html>...login form HTML...`,
    { status: error ? 401 : 200, headers: { 'content-type': 'text/html; charset=utf-8', 'cache-control': 'no-store' } }
  );
}

export default {
  async fetch(req, env) {
    const url = new URL(req.url);
    const secret = env.GATE_SECRET || env.GATE_PASSWORD;

    // Login submission
    if (req.method === 'POST' && url.pathname === '/__auth') {
      const form = await req.formData();
      const pw = (form.get('password') || '').toString().trim().toLowerCase();
      const expected = (env.GATE_PASSWORD || '').toString().trim().toLowerCase();
      if (expected && pw === expected) {
        const token = await makeToken(secret);
        return new Response(null, {
          status: 302,
          headers: {
            Location: '/',
            'Set-Cookie': `${COOKIE}=${token}; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=${MAX_AGE}`,
          },
        });
      }
      return loginPage(true);
    }

    // Authenticated? serve the static site.
    if (await validToken(readCookie(req, COOKIE), secret)) {
      return env.ASSETS.fetch(req);
    }

    // Everything else -> login.
    return loginPage(false);
  },
};
```

## wrangler.jsonc

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-site",
  "main": "worker.js",
  "compatibility_date": "2025-06-01",
  "assets": {
    "directory": "./dist-cf",
    "binding": "ASSETS",
    "run_worker_first": true,
    "not_found_handling": "single-page-application"
  },
  "routes": [
    { "pattern": "my-site.example.com", "custom_domain": true }
  ]
}
```

## Deploy steps

```bash
# 1. Create project dir with worker.js, wrangler.jsonc, and dist-cf/ (static files)
# 2. Set secrets (password + HMAC signing secret)
echo "mypassword" | wrangler secret put GATE_PASSWORD --name my-site
echo "random-hex-secret" | wrangler secret put GATE_SECRET --name my-site

# 3. Deploy
wrangler deploy
```

## Adding API endpoints behind the gate

The Worker can also handle JSON API endpoints (e.g., D1 database) behind the same
auth gate. Just add route matching before the `env.ASSETS.fetch(req)` fallback:

```javascript
// After auth check, before ASSETS fallback:
if (url.pathname === '/api/visits' && req.method === 'GET') {
  const result = await env.DB.prepare('SELECT * FROM visits ORDER BY timestamp DESC').all();
  return new Response(JSON.stringify({ visits: result.results }), {
    headers: { 'content-type': 'application/json' }
  });
}

if (url.pathname === '/api/visits' && req.method === 'POST') {
  const body = await req.json();
  await env.DB.prepare('INSERT INTO visits (id, data) VALUES (?, ?)')
    .bind(crypto.randomUUID(), JSON.stringify(body)).run();
  return new Response(JSON.stringify({ success: true }), {
    status: 201, headers: { 'content-type': 'application/json' }
  });
}
```

Add D1 binding to `wrangler.jsonc`:
```jsonc
"d1_databases": [
  { "binding": "DB", "database_name": "my-db", "database_id": "<DB_ID>" }
]
```

## Verification

```bash
# Login page
curl -s https://my-site.example.com | grep '<title>'

# Auth flow
curl -s -X POST https://my-site.example.com/__auth -d "password=mypassword" -c cookies.txt -o /dev/null -w "%{http_code}"
# Should be 302

# Authenticated content
curl -s https://my-site.example.com -b cookies.txt | grep '<title>'
# Should be the site title, not the login page

# Wrong password
curl -s -X POST https://my-site.example.com/__auth -d "password=wrong" -o /dev/null -w "%{http_code}"
# Should be 401
```

## Known deployments

- `echo-springs.jkre.is` — camp wiki (original, built by Pi with Claude Opus 4.8)
- `aiec.jkre.is` — AI Exec Circle repos dashboard (built by Hermes)