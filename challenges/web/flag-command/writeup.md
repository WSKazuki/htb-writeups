# Flag Command — HTB Challenge Writeup

**Category:** Web  
**Difficulty:** Very Easy  
**Flag:** `HTB{D3v310p3r_t0015_4r3_b35t__t0015_wh4t_d0_y0u_Th1nk??}`

---

## Overview

A browser-based text adventure game hides a secret command. The game presents 4 choices per step, but a fifth "secret" command is validated server-side too — and the full list of commands, including the secret one, is exposed by an unauthenticated API endpoint.

---

## Recon

Spawn the instance and open the web app. It's a terminal-style game: type `start`, pick directions, try to escape the forest. Playing it normally leads to dead ends ("Game over").

The real recon is in the source code.

---

## Exploit

### Step 1 — Read the JavaScript source

Open DevTools (**F12**) → **Sources** → open `commander.js`.

Two key lines:

```js
const fetchOptions = () => {
    fetch('/api/options')
        .then((data) => data.json())
        .then((res) => { availableOptions = res.allPossibleCommands; })
}

if (availableOptions[currentStep].includes(currentCommand) ||
    availableOptions['secret'].includes(currentCommand)) {
    // send command to /api/monitor
}
```

The game accepts a hidden `secret` command — and fetches all options including it from `/api/options` with no authentication.

### Step 2 — Call the API directly

```bash
curl http://<IP>:<PORT>/api/options
```

Response reveals the secret array:

```json
"secret": ["Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"]
```

### Step 3 — Use the secret command

Type it into the game terminal:

Flag returned in the server response.

---

## Why it works

The developer hid the secret command from the UI but served it from a public unauthenticated endpoint. Anyone who reads the JS source sees the `/api/options` call and can fetch the full command list directly. Security through obscurity is not access control.

---

## Tools used

| Tool | Purpose |
|------|---------|
| Browser DevTools (F12) | Read JS source, find API endpoint |
| `curl` | Call `/api/options` directly |

---

## Key Takeaway

Always read the JavaScript. Client-side code is fully visible to anyone in DevTools. If the frontend fetches from an API, that endpoint is reachable by anyone — secret logic belongs server-side, behind authentication, never in a public endpoint response.
