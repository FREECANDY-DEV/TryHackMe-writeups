<div align="center">

# 🚩 TryHackMe Write-ups Collection

Welcome to my comprehensive repository for **TryHackMe (THM)** CTF challenges, event walkthroughs, and machine write-ups.

This repository serves as a structured, detailed collection of penetration testing, cloud security, and vulnerability exploitation guides. Each write-up focuses on the **"why"** and **"how"**, including full execution logs, methodologies, and remediation advice.

---

## 📚 What's in this Repository?

The write-ups are organized by events and overarching themes. You will find hands-on exploitation techniques covering:

- 🌐 **Web Application Exploitation:** Command Injection, SQLi, NoSQLi, SSTI, API Abuse, BOLA/IDOR
- ☁️ **Cloud Security:** AWS (Cognito, DynamoDB), Azure (SAS Leaks, Key Vaults), IAM Misconfigurations
- 🐧 **Boot2Root & Privilege Escalation:** Initial access vectors, port forwarding, and Linux/Windows local privilege escalation
- 🕵️ **OSINT & Forensics:** Information gathering, decoding, and tracing digital footprints
- 🛠️ **Business Logic Flaws:** Race conditions (TOCTOU) and state manipulation

---

## 🌴 Featured Event: Hacker Holidays 2026 - The Byte Lotus

<img width="2822" height="1508" alt="image" src="https://github.com/user-attachments/assets/2ef52b59-4de0-43f9-a5e0-848d542fea13" />

**"A five-star resort with a zero-star security posture."**

A massive 14-day gamified cyber security campaign hosted by TryHackMe. The difficulty scales up as you progress through the resort, uncovering mysteries, bypassing AI concierges, and exploiting critical hotel infrastructure.

### 📝 Completed Rooms
| Room Name | Vulnerability Focus | Write-Up Link |
| :--- | :--- | :--- |
| **Complimentary** | AWS Cognito Unauthenticated Role & DynamoDB BOLA | [Read Here](./hackerholidays/Complimentary.md) |
| **Do Not Disturb** | Node.js NoSQLi, SSTI, PrivEsc via Disk Group | [Read Here](./hackerholidays/Do%20Not%20Disturb.md) |
| **Towel on the Sunbed** | Business Logic, TOCTOU Race Condition, API Abuse | [Read Here](./hackerholidays/Towel%20on%20the%20Sunbed.md) |
| **CryptoCabana** | Azure Storage SAS Leak, Key Vault Secret Versioning | [Read Here](./hackerholidays/CryptoCabana.md) |
| **Infinity Pool** | OS Command Injection, SSH Port Forwarding, Token Leak | [Read Here](./hackerholidays/Infinity_Pool.md) |
| **Easter Egg** | OSINT, Base64 Decoding, Lore | [Read Here](./hackerholidays/Easter_Egg.md) |

> 📁 *For a full breakdown of the event, visit the [Hacker Holidays Directory](./hackerholidays/).*



---

## ⚖️ Disclaimer

All write-ups, scripts, and commands documented in this repository are created strictly for educational purposes and authorized security research on CTF platforms. Please respect the rules of engagement for each platform and do not use these techniques against targets you do not have permission to test.

**Documented and exploited by FREECANDY-DEV**

</div>
