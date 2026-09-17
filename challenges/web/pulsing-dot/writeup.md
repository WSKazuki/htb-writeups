# Pulsing Dot — HTB Challenge Writeup

**Category:** Web  
**Difficulty:** Very Easy  
**Flag:** `HTB{C0SM1C-BYP4SS}`

---

## Overview

A Go frontend service forwards requests to an internal Python backend. The Go service blocks the `getSecureCode` action, but a JSON key case-sensitivity difference between Go and Python allows bypassing the restriction entirely.

---

## Architecture

Two services communicate internally:

- **Go service** (port 8080, public) — receives user requests and routes them
- **Python service** (port 8081, internal only) — handles the actual logic, including the flag endpoint

The Go router blocks `getSecureCode` with "Access denied". The Python router returns the FLAG for `getSecureCode`. Port 8081 is not publicly accessible.

---

## The Vulnerability: JSON Key Case Sensitivity

Go's `encoding/json` unmarshals JSON into structs using **case-insensitive** key matching:

```go
type RequestData struct {
    Action string `json:"action"`
}
```

Both `"action"` and `"Action"` match the same struct field. When both appear, the **last one wins**.

Python's `json` module uses a **dict** — `"action"` and `"Action"` are two completely different keys.

---

## Exploit

```bash
curl http://<IP>:<PORT>/execute -X POST \
  -H "Content-Type: application/json" \
  -d '{"action": "getSecureCode", "Action": "getcosmic"}'
```

**Go:** reads `"action"` = `getSecureCode`, then `"Action"` = `getcosmic` (case-insensitive match, overwrites). Final value: `getcosmic` → forwards full body to Python.

**Python:** `data['action']` → exact key → `"getSecureCode"` → returns FLAG.

---

## Why it works

The Go service was supposed to use JWT to authenticate inter-service calls (`github.com/dgrijalva/jwt-go` is imported but never used). Without that, the only protection is the Go-level action check — bypassed by the parser difference.

Root causes: missing inter-service authentication + Go case-insensitive JSON parsing vs. Python case-sensitive dict keys.

---

## Key Takeaway

When two services share a JSON payload but parse it with different languages/libraries, edge cases (duplicate keys, case variations) may be interpreted differently. Never pass raw client input through unchecked — re-validate at every trust boundary.
