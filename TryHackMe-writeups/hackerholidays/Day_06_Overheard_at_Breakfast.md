# Hacker Holidays: Day 6 - Overheard at Breakfast

**Category:** OSINT
**Difficulty:** Easy

## 🛎️ Concierge Briefing
> A screenshot of a breakfast conversation contains two throwaway details: an email address and a free profile tool that starts with "G". Together they point at an account that was never meant to be found.

---

## 🕵️‍♂️ Recon

The task file is a single `conversation.png` — a screenshot of a chat between two guests, `Ponzi` and `Lambo`. The hint from `@0xMia` is blunt:

```text
"the breakfast crowd really said the quiet part out loud this morning
y'all need to actually READ what they said, not just skim it"
```

So I read it word by word. The relevant part of the exchange:

> Lambo: "I used to use this free tool that let me upload my profile and link other media accounts... Started with a G if I remember correctly."
> Lambo: "... this is my best way of communication: lambobytelotushotel@gmail.com"

Two clues:

- A free service starting with **G**, described as a place to host a profile and link other media accounts
- The email `lambobytelotushotel@gmail.com`

## 🔎 Identifying the Platform

"Free tool starting with G that hosts a profile and links your other accounts" = **Gravatar**. A Gravatar profile is bound to an email address and can be looked up from it. That profile-to-email binding is exactly what makes Gravatar useful for OSINT.

## 🕸️ Finding the Profile

A Gravatar profile URL is the SHA-256 of the normalized email address:

```text
https://gravatar.com/{sha256(email)}
```

No need to compute the hash by hand — Gravatar's email lookup accepts the address directly. Submitting `lambobytelotushotel@gmail.com` resolves to the hidden profile `https://gravatar.com/cheerfullysongf28e3c3716`: display name `Lambo`, location `Byte Lotus Hotel`, and a bio that looks like Base64.

## 🧬 Decoding the Bio

The bio string has the classic Base64 shape: alphanumerics with optional `=` padding. One operation in CyberChef:

```text
From Base64
```

and the bio turns into the room flag. Base64 is encoding, not encryption — every correct reader produces the same output.

## 🚩 The Flag

**Flag:** `THM{REDACTED}`

## 💡 Why This Works

1. **An email is an identifier.** We treat addresses as private, but an email is one of the strongest identity indexes online and can be tied to many services.
2. **A "G" + email is enough.** Knowing the platform collapses the search space to a single profile.
3. **Base64 is not encryption.** Anyone who copies the string and decodes it reaches the flag.

## 🛡️ Protecting Yourself

- Don't drop personal email addresses into public conversations
- An email hash is an identifier, not anonymization
- Audit old Gravatar / link-in-bio profiles and remove ones you no longer use
- Chat screenshots are public material — assume they'll be used in an OSINT investigation

## 📝 Final Thoughts

Pure OSINT: two casual details in a screenshot, one hidden profile, one Base64 string. People leak more in normal conversation than in any breach.

**Documented and tested for TryHackMe — Hacker Holidays 2026**
