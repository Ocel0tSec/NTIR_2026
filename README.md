# NTIR_2026
Lab documents and guidance for the Intro to Cloud Pen Testing Workshop

# MergerMayhem

**An introductory cloud penetration testing workshop built on Microsoft Entra ID.**

MergerMayhem simulates a post-merger corporate environment where two companies — Ocelot Security and Revolver Industries — have consolidated into a single Entra ID tenant. Misconfigurations introduced during the IT migration create an exploitable attack path that students follow from initial reconnaissance through lateral movement and data exfiltration.

The workshop is designed for security professionals with little or no cloud pentesting experience. No prior Azure or Entra ID knowledge is required. All tools used are free and open source, and the lab environment runs entirely on the Entra ID Free tier at zero cost.

**Duration:** 45 minutes  
**OSINT Target:** [https://ocelotsecurity.com/ntir-workshop/](https://ocelotsecurity.com/ntir-workshop/)

---

## Attack Path Overview

The lab follows a five-phase kill chain. Each phase introduces a core cloud pentesting concept and a corresponding tool.

| Phase | Objective | Technique | Tool |
|-------|-----------|-----------|------|
| 1 | Discover employee names and email format | OSINT, user enumeration | Company website, o365spray |
| 2 | Compromise a user account | Password spraying | MSOLSpray or o365spray |
| 3 | Explore the tenant and find sensitive files | Post-compromise enumeration via Microsoft Graph API | GraphRunner |
| 4 | Access resources belonging to a different business unit | Credential reuse, lateral movement | Browser or Graph API |
| 5 | Document findings and discuss defenses | Reporting | — |

---

## Prerequisites

Before starting the lab, ensure you have the following installed on your machine.

### Required Software

| Software | Purpose | Install |
|----------|---------|---------|
| Python 3.8+ | Run o365spray | [python.org](https://www.python.org/downloads/) |
| Git | Clone tool repositories | [git-scm.com](https://git-scm.com/) |
| PowerShell 5.1+ or 7+ | Run GraphRunner and MSOLSpray | Pre-installed on Windows; [Install on macOS/Linux](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell) |

### Required Tools

Clone each of these repositories to your local machine.

**o365spray** — User enumeration and password spraying for Microsoft 365:
```bash
git clone https://github.com/0xZDH/o365spray.git
cd o365spray
pip3 install -r requirements.txt
```

**MSOLSpray** — Password spraying tool for Microsoft Online (PowerShell):
```bash
git clone https://github.com/dafthack/MSOLSpray.git
```

**GraphRunner** — Post-exploitation toolset for Entra ID using Microsoft Graph API:
```bash
git clone https://github.com/dafthack/GraphRunner.git
```

Verify each tool loads without errors before the workshop begins.

---

## Lab Walkthrough

### Phase 1 — OSINT and User Enumeration

Your engagement starts with open-source intelligence gathering. Navigate to the company website and gather information about the organization and its employees.

**Step 1.** Open the OSINT target page in your browser:

```
https://ocelotsecurity.com/ntir-workshop/
```

**Step 2.** Review the page carefully. Look for employee names, job titles, and organizational structure. Pay close attention to any email addresses — they reveal the username format used by the organization.

**Step 3.** Based on the naming convention you identified, build a text file called `userlist.txt` with one potential username per line. For example, if you see an email formatted as `f.last@domain.com`, apply that pattern to every employee name on the page.

**Step 4.** Add decoy usernames that don't exist. This makes the enumeration exercise more realistic — in a real engagement, not every guess will be valid.

**Step 5.** Run the enumeration. Replace `TARGET_DOMAIN` with the domain identified on the company page:

```bash
python3 o365spray.py --enum -U userlist.txt -d TARGET_DOMAIN --enum-module oauth2
```

The tool will report which usernames are **VALID** and which are **INVALID**. Record the valid usernames — you will use them in the next phase.

> **What is happening here?** Microsoft's OAuth2 authentication endpoint responds differently to valid and invalid usernames. This is a known behavior that allows unauthenticated user enumeration against any Entra ID tenant.

---

### Phase 2 — Password Spraying

Password spraying tests a single common password against many accounts simultaneously. Unlike brute force, spraying avoids account lockouts by limiting the number of attempts per user.

**Step 1.** Create a file called `valid_users.txt` containing only the usernames confirmed valid in Phase 1.

**Step 2.** Choose a password to spray. Common patterns during organizational transitions include:
- `Welcome2024!` — default temporary passwords
- `Season+Year` — `Summer2024!`, `Winter2024!`
- `CompanyName+Year` — `Ocelot2024!`

**Step 3.** Execute the spray:

```bash
python3 o365spray.py --spray -U valid_users.txt -p "Welcome2024!" -d TARGET_DOMAIN
```

Or using PowerShell:

```powershell
Import-Module .\MSOLSpray.ps1
Invoke-MSOLSpray -UserList .\valid_users.txt -Password "Welcome2024!"
```

**Step 4.** Review the output. A successful result indicates the password is valid for that account. Note the compromised username and password.

> **Why does this work?** New employees, recent mergers, and IT transitions frequently result in accounts with temporary passwords that were never rotated. In production environments, Multi-Factor Authentication and Conditional Access policies prevent this attack.

---

### Phase 3 — Post-Compromise Enumeration

With valid credentials, you can now authenticate to the tenant and enumerate its resources using the Microsoft Graph API.

#### Option A — Browser (Recommended for Beginners)

**Step 1.** Open a browser and navigate to [https://portal.office.com](https://portal.office.com).

**Step 2.** Sign in with the compromised credentials.

**Step 3.** Explore the environment. Click through the app launcher and note what applications are available.

**Step 4.** Navigate to **SharePoint**. Browse the sites accessible to this account. Open any document libraries and review their contents.

**Step 5.** If you find files of interest, download them and examine their contents.

#### Option B — GraphRunner (Recommended for Practitioners)

**Step 1.** Launch PowerShell and import GraphRunner:

```powershell
Import-Module .\GraphRunner.ps1
```

**Step 2.** Authenticate. Enter the compromised credentials when prompted:

```powershell
Get-GraphTokens
```

**Step 3.** Enumerate the tenant:

```powershell
Invoke-GraphRecon -Tokens $tokens -PermissionEnum
Get-AzureADUsers -Tokens $tokens -OutFile users.txt
Get-SecurityGroups -AccessToken $tokens.access_token
Get-SharePointSiteURLs -Tokens $tokens
```

**Step 4.** Search SharePoint and OneDrive for sensitive content:

```powershell
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm "credentials"
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm "password"
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm "migration"
```

**Step 5.** Review any files discovered. Look for credentials, connection strings, API keys, or references to other systems.

> **What is happening here?** Every Entra ID user has default read permissions across the directory. GraphRunner wraps the Microsoft Graph API (`/v1.0/users`, `/v1.0/groups`, `/v1.0/sites`) to enumerate these resources programmatically.

---

### Phase 4 — Lateral Movement

If you discovered credentials for a different user in Phase 3, you can now attempt to authenticate as that user and access resources the original compromised account could not reach.

**Step 1.** Open an **incognito or private browser window** (this prevents session conflicts with your previous login).

**Step 2.** Navigate to [https://portal.office.com](https://portal.office.com) and sign in with the newly discovered credentials.

**Step 3.** Explore this user's environment. Compare what this account can access versus the first compromised account. Check the following:
- **SharePoint** — Are there new sites visible that weren't accessible before?
- **Power Automate** ([https://flow.microsoft.com](https://flow.microsoft.com)) — Are there shared automation flows?
- **Power Apps** ([https://make.powerapps.com](https://make.powerapps.com)) — Are there shared applications?

**Step 4.** Examine any Power Automate flows carefully. Look for hardcoded credentials, database connection strings, API keys, or personally identifiable information (PII) embedded in the flow definitions.

> **What is happening here?** Credential reuse across business units is a common finding in post-merger environments. When migration teams store credentials in shared documents, a single compromised account can provide access to an entirely separate segment of the organization.

---

### Phase 5 — Reporting

Document the complete attack path you followed. In a real penetration test, this becomes the deliverable that drives remediation.

Record the following:

1. **Initial Access** — How did you gain entry? What credentials were compromised and how?
2. **Data Discovered** — What sensitive information was accessible from each account?
3. **Lateral Movement** — How did you move between accounts? What enabled the pivot?
4. **Business Impact** — What is the worst-case scenario if an adversary followed this path?
5. **Remediation** — What are your top three recommendations to prevent this attack?

---

## Defensive Recommendations

After completing the lab, consider how each phase could have been prevented:

| Attack Phase | Defense |
|-------------|---------|
| User enumeration | Entra ID Smart Lockout, rate limiting (cannot fully prevent) |
| Password spraying | Multi-Factor Authentication, Conditional Access policies, banned password lists |
| Tenant enumeration | Restrict default user permissions, audit directory read access |
| Credential in SharePoint | Store secrets in Azure Key Vault or a PAM solution, never in documents |
| Power Automate data exposure | Power Platform DLP policies, governance controls for embedded secrets |
| Cross-BU lateral movement | Least-privilege group memberships, separate service accounts per business unit |

---

## Workshop Materials

| Resource | Description |
|----------|-------------|
| [Student Handout (DOCX)](./MergerMayhem_Student_Handout.docx) | Printable lab guide with step-by-step instructions and space to record findings |
| [Instructor Guide (DOCX)](./MergerMayhem_Instructor_Guide.docx) | Full instructor manual with setup instructions, answer key, timeline, and talking points |
| [OSINT Target Page](https://ocelotsecurity.com/ntir-workshop/) | Simulated company website used for initial reconnaissance |

---

## Additional Resources

| Resource | Link |
|----------|------|
| GraphRunner (BHIS) | [github.com/dafthack/GraphRunner](https://github.com/dafthack/GraphRunner) |
| Microsoft Graph API Documentation | [learn.microsoft.com/en-us/graph/api/overview](https://learn.microsoft.com/en-us/graph/api/overview) |
| Graph Explorer (browser-based API testing) | [developer.microsoft.com/graph/graph-explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) |
| HackTricks Cloud — Azure Pentesting | [cloud.hacktricks.xyz/pentesting-cloud/azure-security](https://cloud.hacktricks.xyz/pentesting-cloud/azure-security) |
| BadZure (Vulnerable Entra ID lab) | [github.com/mvelazc0/BadZure](https://github.com/mvelazc0/BadZure) |
| Breaching the Cloud (Antisyphon Training) | [antisyphontraining.com](https://www.antisyphontraining.com/on-demand-courses/breaching-the-cloud-w-beau-bullock/) |
| CARTP — Azure Red Team Certification | [alteredsecurity.com/cartp](https://www.alteredsecurity.com/cartp) |

---

## Disclaimer

This workshop and its associated lab environment are intended for **authorized security education and training only**. The techniques and tools demonstrated should only be used against systems and accounts you have explicit written permission to test. Unauthorized access to computer systems is illegal.

---

## License

MIT
