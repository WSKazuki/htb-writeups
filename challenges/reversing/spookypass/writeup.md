# SpookyPass — HTB Challenge Writeup

**Category:** Reversing  
**Difficulty:** Very Easy  
**Flag:** `HTB{un0bfu5c4t3d_5tr1ng5}`

---

## Overview

A binary asks for a password to let you into the "spookiest party of the year." The password is stored in plaintext inside the executable — `strings` is all you need.

---

## Steps

### 1. Unzip the challenge

HTB challenge ZIPs use the standard password `hackthebox`:

```bash
unzip -P hackthebox a12c739e-dddf-43d7-bbf0-c4389ea79b09.zip
cd rev_spookypass
```

### 2. Identify the file

```bash
file pass
# pass: ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped
```

### 3. Extract readable strings

```bash
strings pass
```

Key output:

Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password:
s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
You're not a real ghost; clear off!


The password sits in plaintext — the binary uses `strcmp` against this hardcoded value.

### 4. Run the binary

```bash
chmod +x pass
./pass
# Enter: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
```

---

## Why it works

The password is a string literal compiled into the `.rodata` section with no encryption or obfuscation. `strings` reads it verbatim. The binary was also compiled without stripping symbols, making the `strcmp` call fully visible.

---

## Tools used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the password-protected ZIP |
| `file` | Identify the binary format |
| `strings` | Dump printable text from the binary |

---

## Key Takeaway

`strings` is always the first tool on any reversing binary. For harder challenges where secrets are obfuscated, the next steps are `ltrace` (intercept `strcmp` at runtime) or Ghidra/Radare2 for static decompilation.
