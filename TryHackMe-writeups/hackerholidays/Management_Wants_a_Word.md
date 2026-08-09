# Hacker Holidays: Day 14 - Management Wants a Word

**Category:** Forensics
**Difficulty:** Hard

## 🛎️ Concierge Briefing
> Management wants "a word" — and it's buried somewhere Vera didn't want anyone looking.
> 
> A full KAPE acquisition of Vera's workstation has been handed to you. On the surface it's a quiet, ordinary Windows profile. But sitting in `Documents` is a 100 MiB file called `backup` that doesn't behave like a backup at all. Somebody went to a lot of trouble to make sure the "word" could only be read by someone willing to follow a very long chain.

---

## 🕵️‍♂️ Initial Analysis & Identifying the Files

When I first downloaded and extracted the challenge files, I was presented with a **KAPE collection** — a triaged copy of the live machine's `C:` drive:

```text
management-wants-a-word-forensics-hh-day-14/
└── KAPE/C/
    ├── Windows/System32/config/SAM, SYSTEM, SECURITY      ← registry hives
    └── Users/vera/
        ├── Documents/backup                               ← 100 MiB, the target
        ├── AppData/Roaming/Microsoft/Protect/<SID>/…      ← DPAPI master key folder
        └── AppData/Local/Google/Chrome For Testing/…      ← browser profile
```

> ⚠️ **The silent extraction bug.** The very first `unzip` **silently skipped** the entire `AppData/Roaming/Microsoft/Protect/<SID>/` directory — the folder holding the DPAPI master key. I only noticed when the DPAPI chain appeared to be completely dead. Before you ever conclude an artifact is "missing", re-list the archive and compare it against the extracted tree, then re-extract. That one folder is the linchpin of everything that follows.

First look at the mystery file:

```bash
$ file backup
backup: data

$ ls -l backup
-rw-r--r-- 1 vera vera 104857600 Jul 20 14:02 backup
```

Exactly 100 MiB, no magic bytes at all, a multiple of 512 — this has *VeraCrypt container* written all over it. The question is: what's the password? Nothing in the visible filesystem hints at it, so I went hunting where Windows hides its secrets.

---

## 🛠️ Step 1: Recovering the Windows Password from LSA Secrets

Windows stores the plaintext logon password in **LSA secrets** on many single-user builds. With the three offline hives, `impacket`'s `secretsdump` pulls it out in seconds:

```bash
python3 impacket/examples/secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL
```

Relevant output:

```text
[*] Target system bootKey: 0x0f6f73ce89c8cda52d06fcc5131e040f
[*] Dumping LSA Secrets
[*] DefaultPassword: minivera
[*] DPAPI_SYSTEM: 01000000d08c9ddf0115d1118c7a00c04fc297eb01000000...
[*] vera:1000:aad3b435b51404eeaad3b435b51404ee:1241186a4aac4f34f4bf7ace71b396a8:::
```

Two things immediately mattered:

1. **`DefaultPassword: minivera`** — the plaintext Windows password for user `vera`.
2. **`vera`'s NTLM hash** `1241186a4aac4f34f4bf7ace71b396a8` — quick sanity check by MD4-ing the password:

```python
import hashlib
h = hashlib.new('md4', 'minivera'.encode('utf-16le')).hexdigest()
assert h == '1241186a4aac4f34f4bf7ace71b396a8'   # ✓ password confirmed
```

`minivera` is not the answer to the challenge — it's the key that unlocks the next lock.

---

## 🔐 Step 2: Unwrapping the DPAPI Master Key

Anything Windows protects on behalf of a user — Chrome's `Local State`, saved credentials, `Credential Manager` — is encrypted with **DPAPI**. The scheme is layered:

```text
user password ─► master key (Protect/<SID>/<GUID>) ─► per-app blobs
```

Offline, I reproduced that with **`dpapick3`**:

```python
import dpapick3.masterkeyfile

data = open('Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a', 'rb').read()
mk = dpapick3.masterkeyfile.MasterKeyFile(data)
mk.decryptWithPassword('minivera')          # unwraps with the recovered logon password
print(mk.get_key().hex())                   # → 64-byte master key
```

With the master key in hand, I could decrypt any DPAPI blob from this user's context — including the one Chrome uses to wrap its master AES key.

---

## 🔑 Step 3: Pulling Vera's Vault Password from Chrome

The Chrome profile in the acquisition is **"Chrome For Testing" v126**. Chrome protects saved passwords in two stages:

1. **`Local State` → `os_crypt.encrypted_key`** — a base64, DPAPI-encrypted blob that decrypts (using the Step 2 master key) to the browser's 32-byte **master AES key**.
2. **`Login Data`** (SQLite) — each saved password's `password_value` column holds an encrypted blob. Modern Chrome versions use the **`v10`** format:

```text
v10 || nonce (12 B) || AES-256-GCM ciphertext || tag (16 B)
```

Decrypting the blob for the target origin:

```python
import base64, json
from Crypto.Cipher import AES

# 1) unwrap the browser master key from Local State
enc = base64.b64decode(json.load(open('Local State'))['os_crypt']['encrypted_key'])
aes_key = dpapi_decrypt(enc[len(b'DPAPI'):])          # → 32-byte key

# 2) decrypt the saved credential blob (Login Data → password_value)
blob  = bytes.fromhex('763130 ...')                    # 'v10' header, 56 B total
nonce, ct, tag = blob[3:15], blob[15:-16], blob[-16:]
pw = AES.new(aes_key, AES.MODE_GCM, nonce=nonce).decrypt_and_verify(ct, tag)
print(pw.decode())
```

> 💡 **Don't assume the older scheme.** Pre-2022 Chrome used `v10`-style **AES-128-CBC + HMAC-SHA1**. Older *and* newer profiles often slip back into plain **AES-256-GCM** (`v10`). Read the header byte before picking your decryptor — running a GCM blob through the PBKDF2/HMAC routine just produces garbage.

The recovered credential:

| Field | Value |
| :--- | :--- |
| Origin / URL | `bytelotus.thm:8080` |
| Username | `VeraSecretVault` |
| **Password** | `Wh4t1sV3raD0inG0nTh1sH0st` |

The username says it all — *Vera's secret vault*. And "What is Vera doing on this host" is a lot less cryptic once you know where the vault lives.

---

## 🗄️ Step 4: Cracking the VeraCrypt Header

Back to `backup`. VeraCrypt stores its volume header (512 bytes) at a known offset, so I probed for the magic:

```bash
$ xxd -l 16 backup
00000000  56 45 52 41 5a 2b 2a 21 08 07 05 01 ...
```

`VERA` confirmed — and the header carries its own integrity check, so a password can be validated *without* mounting anything: derive the KEK from the candidate (`SHA-512`, 500,000 iterations), unwrap the AES-XTS keys, and recompute the header CRC. I built a tiny cracker (`vc_crack`) and threw the vault password at it:

```text
Header @ 0x000000: magic=VERA, version=0x0105, hdr_crc=acba3b6c   (✓ valid)
salt             = 6c1d06e4… (64 B)
cipher           = AES, mode = XTS
kdf              = SHA-512, iterations = 500,000
[+] MATCH(sha512@500000, Wh4t1sV3raD0inG0nTh1sH0st)
```

No dictionary attack required — the password had been sitting in Chrome the whole time.

> 💡 **Hidden-volume gotcha.** There is a second 512-byte blob at offset `0x10000` (65,536) with salt `31ddb99e…`. That is the **hidden volume header slot** (`TC_HIDDEN_VOLUME_HEADER_OFFSET`), not the encrypted data. Treating it as the real header is a classic rabbit hole — the primary header lives at offset 0.

---

## 💽 Step 5: Decrypting the Container & Extracting the Filesystem

With a validated password, decrypting the volume is just careful AES-XTS bookkeeping. No root, no `veracrypt` binary, no OpenCL — so I rolled a small C decryptor (`vc_vol`) using the format constants:

- Data region begins at `TC_VOLUME_DATA_OFFSET = 131,072`
- Data unit number = `byte_offset / 512` (first data unit is unit `256`)
- Key material at decrypted-header offset `256` → bytes `[192:448]`: first 32 = **data key**, next 32 = **tweak key** (AES-XTS-256)
- Header CRC fields are stored **big-endian** (validated: `CRC32(keydata) == 0x049a2d8d`)

```bash
$ ./vc_vol backup vol_raw.img
[+] wrote 104726528 bytes
$ file vol_raw.img
vol_raw.img: DOS/MBR boot sector … FAT32
```

The decrypted image is a **FAT32** filesystem. `7z` reads it directly:

```bash
$ 7z x vol_raw.img -ofat_out
```

```text
secret_financial_documents/
    important_invoice_byte_lotus.pdf      26,747 B
    transactions_q3.csv                      427 B
$RECYCLE.BIN/desktop.ini
System Volume Information/WPSettings.dat
```

`transactions_q3.csv` is a boring list of vendor payments (Byte Lotus Catering, Sunrise Transport, Lotus Printworks, Byte Lotus Resorts…). The interesting document is the PDF.

---

## 🚩 Step 6: Reading the Flag from the Invoice

`important_invoice_byte_lotus.pdf` has **no text layer** — it's a single rasterized image of an invoice (636×724, 8-bit RGB, FlateDecode). Extract the raw pixels, render to a PNG, and OCR it:

```bash
python3 render_invoice.py important_invoice_byte_lotus.pdf invoice.png
tesseract invoice.png -      # plus RapidOCR as a second opinion
```

OCR reads a fake "Byte Lotus Resorts" invoice (INVOICE NR `2122/9090/5050`, BILL TO `Hotel Cleaning LLC`) whose **DESCRIPTION** line contains the flag:

```text
Flag: THM{REDACTED}
```

> 🔎 **Glyph verification.** The flag line is only ~8 px tall, and OCR engines disagree on several characters (`1` vs `l`, `0` vs `O`, `{` vs `[`). When the flag is tiny, don't trust a single OCR pass — crop the line, binarize it, and compare each glyph against the font's shapes at the pixel level. That's what disambiguates the leetspeak characters here.

Once decoded, the "word" management wanted is the phrase the whole event has been building toward: **"It was Vera all along."**

**Flag:** `THM{REDACTED}`

---

## 📝 Final Thoughts

This challenge was a complete forensic chain, and every single link depended on the previous one:

1. **LSA secrets** gave up the plaintext logon password (`minivera`) with zero cracking.
2. That password **unwrapped the DPAPI master key**.
3. The master key **decrypted Chrome's Local State**, which gave the AES key for the saved-credential blob.
4. The credential blob contained **the VeraCrypt password**.
5. The VeraCrypt password **unlocked the container** and the invoice inside.

A few takeaways:

- **Verify your acquisitions.** The ZIP silently dropped the DPAPI `Protect` folder on first extraction — one un-trusted extraction pass and the whole DPAPI chain looks "unsolvable". Always diff the archive manifest against the extracted tree.
- **`DefaultPassword` is gold.** LSA secrets routinely hold the plaintext logon password; check it before reaching for hash cracking.
- **The browser vault is a password dump.** `Local State` + `Login Data` are trivially decryptable offline with the master key — and people store *everything* in them, including VeraCrypt container passwords.
- **Encryption is only as strong as its weakest key holder.** The container was protected with SHA-512 at 500,000 iterations — effectively un-bruteforceable — but the password was recoverable in minutes because it lived inside the Chrome vault.

---

**Documented and tested for TryHackMe — Hacker Holidays 2026**
