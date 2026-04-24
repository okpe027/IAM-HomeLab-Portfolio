# IAM Home Lab Portfolio

**Opeyemi Ogundimu** &nbsp;|&nbsp; IAM Analyst &nbsp;|&nbsp; Security Engineer  
**GitHub:** github.com/okpe027/My_First_Project  
**Certifications:** CompTIA A+ · CompTIA Security+ · Cisco CCNA · SC-300 (in progress)

---

## About This Portfolio

This portfolio documents a 20-lab hybrid IAM program I built to formalize and deepen the identity and access management experience I work with daily in my current role as a Technical Support Engineer at Emerson Electric Canada.

The production environment at Emerson runs Okta as the identity provider for 25+ enterprise applications — Salesforce, AWS, Microsoft 365, ServiceNow, Snowflake, Okta Privileged Access, and others. This lab replicates that architecture at a controlled scale using the same tools and the same patterns: Active Directory on-premises, Microsoft Entra ID in the cloud, Azure AD Connect bridging the two, and Okta handling SSO federation across applications.

Every lab is documented as a case study — not a step-by-step tutorial. The focus is on why decisions were made, what risks each control addresses, and what the real-world parallel looks like in a production environment.

---

## Lab Environment

| Component | Detail |
|---|---|
| Hypervisor | VMware Workstation |
| Domain Controller | DC01 · Windows Server 2022 · 192.168.10.10 |
| Domain | FictionalCorp.local |
| Cloud Tenant | fictionalcorp.onmicrosoft.com · Microsoft Entra ID P2 |
| Sync | Azure AD Connect · Password Hash Sync · Password Writeback |
| Phase 3 (Lab 09+) | Okta Developer Tenant |
| Ticketing | ServiceNow PDI (mock) |

---

## Program Structure

| Phase | Labs | Focus |
|---|---|---|
| Phase 1 | 01 – 04 | Active Directory & On-Premises Identity |
| Phase 2 | 05 – 08 | Microsoft Entra ID & Hybrid Identity |
| Phase 3 | 09 – 12 | SSO & Okta Integration |
| Phase 4 | 13 – 16 | Zero Trust & Privileged Access |
| Phase 5 | 17 – 20 | End-to-End Scenarios & Interview Simulations |

---

## Progress

| Lab | Title | Status |
|---|---|---|
| 01 | Active Directory — On-Premises Identity Foundation | ✅ Complete |
| 02 | RBAC Design, Role Engineering & Access Governance | ✅ Complete |
| 03 | Employee Offboarding & Full Access Revocation | ✅ Complete |
| 04 | Account Lockout Investigation & Security Event Analysis | ✅ Complete |
| 05 | Azure AD Connect — Building the Hybrid Identity Bridge | ✅ Complete |
| 06 | Conditional Access Policies | ⏳ In Progress |
| 07 | Hybrid Offboarding — Cloud Session Revocation | ⏳ Upcoming |
| 08 | Privileged Identity Management & SSPR | ⏳ Upcoming |
| 09 | SAML SSO — Okta to Salesforce | ⏳ Upcoming |
| 10 | Okta Lifecycle Management | ⏳ Upcoming |
| 11 – 20 | Continuing... | ⏳ Upcoming |

---
---

# Lab 01 — Active Directory: On-Premises Identity Foundation

**Completed:** April 2026 &nbsp;|&nbsp; **Phase:** 1 &nbsp;|&nbsp; **Environment:** VMware · Windows Server 2022 · Active Directory

---

## The Problem This Lab Solves

Most enterprise environments are hybrid — Active Directory managing the core identity layer on-premises, with Microsoft 365 and cloud platforms sitting on top. Before any of that works, someone has to build the foundation correctly.

This lab builds that foundation for FictionalCorp. The domain, OU structure, security groups, and provisioning workflow established here carry forward through all 20 labs. When Azure AD Connect is configured in Lab 05, every decision made in Lab 01 becomes visible in the cloud. The OU structure determines sync scope. The naming conventions carry across. The group design drives cloud access policy from Lab 06 onwards.

The goal was not to complete a setup checklist. It was to make decisions that hold up under scrutiny — the kind an auditor or a security-conscious hiring manager would ask about.

---

## Environment

| Setting | Value | Why it matters |
|---|---|---|
| Server | DC01 | Standard naming for Domain Controller 01 |
| Domain | FictionalCorp.local | .local — internal non-routable suffix |
| IP | 192.168.10.10 (static) | DCs must have a fixed IP. DHCP breaks AD. |
| DNS | 127.0.0.1 (self-referencing) | Server becomes its own DNS after AD promotion |
| Network 1 | Host-only (192.168.10.x) | AD traffic isolated on private network |
| Network 2 | NAT (internet) | Required for Azure AD Connect download in Lab 05 |

---

## Organisational Unit Design

Two rules drove the OU structure: departments get their own OUs for targeted Group Policy, and account states get their own OUs so the environment is readable at a glance.

```
FictionalCorp.local
├── HR
├── Finance
├── IT
├── Sales
├── Contractors        ← All time-limited accounts — separate GPO baseline
├── Disabled_Users     ← Terminated accounts — 30-day retention, excluded from cloud sync
└── Service_Accounts   ← Non-human accounts — no interactive logon
```

The `Disabled_Users` OU is excluded from Azure AD Connect sync scope. When someone is offboarded, their account is disabled and moved here — and it disappears from Entra ID automatically. Active in AD means active in the cloud. Terminated in AD means absent from the cloud.

---

## Security Groups

Permissions go on groups. Users go into groups. Direct user-to-resource permissions are not used anywhere in this environment. The groups below were created here and expanded significantly in Lab 02.

| Group | OU | Purpose |
|---|---|---|
| GRP-HR-Users | HR | Standard HR access |
| GRP-Finance-Users | Finance | Standard Finance access |
| GRP-Finance-InvoiceCreate | Finance | Invoice creation — SoD control |
| GRP-Finance-InvoiceApprove | Finance | Invoice approval — SoD control |
| GRP-IT-Users | IT | Standard IT access |
| GRP-Sales-Users | Sales | Standard Sales access |

The two Finance invoice groups were created deliberately at this stage. Lab 02 demonstrates a SoD violation using them. Designing the structure correctly from the start is the point.

---

## Provisioning

**INC-0001 — Marcus Williams — Sales Representative — Permanent**  
Placed in the Sales OU. Assigned to GRP-Sales-Users only. Password set to force change at first logon. The provisioning password belongs to the analyst — not the user.

**INC-0002 — Jessica Park — IT Support Contractor — 90-day contract**  
Placed in the Contractors OU. Assigned to GRP-IT-Users. Account expiry set at provisioning — not at offboarding. When the contract end date arrived, AD disabled the account automatically. No ticket required. No human memory required. Username: `j.park.contractor` — the suffix carries information without opening the account.

---

## Recommendations

**Set contractor expiry at provisioning, not offboarding.** Offboarding is a reactive process that depends on someone remembering to raise a ticket. Expiry is automatic and proactive.

**Exclude Disabled_Users from Azure AD Connect sync scope.** This needs to be configured before the first sync runs, not retrofitted afterwards.

**Separate contractors from permanent staff at the OU level, not just the group level.** Group separation handles access. OU separation handles policy and cloud sync scope. You need both.

---

## What I Would Do Differently in Production

Add a routable UPN suffix (`fictionalcorp.com`) as an alternative UPN before the first Azure AD Connect configuration. Users then sync with clean cloud UPNs. Doing this after accounts exist requires a UPN migration — a disruptive operation that often gets deferred indefinitely. The right time is before the sync is configured.

---

## Personal Observation

The thing that became clear doing this lab is how much the early decisions constrain everything that follows. OU structure, naming conventions, the decision to separate contractors — none of these feel significant in week one. By Lab 05 when users are syncing to Entra ID, every one of those decisions is visible in the cloud. Getting them right in Lab 01 meant Lab 05 was straightforward. Getting them wrong would have meant rework before the sync could even be configured cleanly.

---
---

# Lab 02 — RBAC Design: Building the Access Control Model

**Completed:** April 2026 &nbsp;|&nbsp; **Phase:** 1 &nbsp;|&nbsp; **Environment:** VMware · Windows Server 2022 · Active Directory · Excel

---

## The Problem This Lab Solves

Most access control failures do not happen because someone broke through a security control. They happen because the access was never designed properly in the first place. A user who can both create and approve invoices. An engineer who still has Finance access six months after moving to a different team. A contractor who accumulated permissions across three role changes and nobody noticed.

This lab builds the role matrix that prevents all of that — before a single account is created. The discipline is designing before building. Every group in AD, every user assignment, every permission decision traces back to a deliberate row in a documented role matrix.

---

## Role Matrix

Eleven distinct roles mapped across six systems. Three access levels: Allow, Deny, Read-Only.

| Role | AD | Email | FinanceCore | PeopleHub | SalesPro | ITConsole | Inv.Create | Inv.Approve | Admin |
|---|---|---|---|---|---|---|---|---|---|
| HR-Manager | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| HR-Coordinator | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Finance-Manager | ✅ | ✅ | ✅ | 👁 | ❌ | ❌ | ❌ | ✅ | ❌ |
| Finance-Analyst | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Finance-Approver | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| IT-Manager | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ |
| IT-Support | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| IT-Contractor | ✅ | ❌ | ❌ | ❌ | ❌ | 👁 | ❌ | ❌ | ❌ |
| Sales-Manager | ✅ | ✅ | ❌ | 👁 | ✅ | ❌ | ❌ | ❌ | ❌ |
| Sales-Rep | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Service-Account | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

*✅ Allow · ❌ Deny · 👁 Read-Only*

---

## SoD Policy

Three conflict pairs defined before any account was created:

**SOD-001:** Finance-Analyst + Finance-Approver — one person creating and approving their own invoices enables fraud. SOX Section 404.

**SOD-002:** IT-Support + Admin Rights — privilege escalation risk. ISO 27001 A.9.2.3.

**SOD-003:** Human role + Service-Account — non-auditable access. NIST SP 800-53 AC-6.

---

## Privilege Creep — Detected and Remediated

Alex Chen was promoted from Finance Analyst to Finance Manager three months ago. The role change added his new groups but never removed the old ones. Result: Alex could create invoices and approve them simultaneously — a CRITICAL SoD violation sitting on top of a HIGH privilege creep finding.

**Before:**

| User | Finance-Analysts | Finance-Managers | InvoiceCreate | InvoiceApprove | SoD |
|---|---|---|---|---|---|
| a.chen | ✅ | ✅ | ✅ | ✅ | ❌ CRITICAL |

**Findings raised:** AF-001 (Privilege Creep — HIGH) and AF-002 (SoD Violation — CRITICAL). Both remediated same day.

**After:**

| User | Finance-Analysts | Finance-Managers | InvoiceCreate | InvoiceApprove | SoD |
|---|---|---|---|---|---|
| a.chen | ❌ | ✅ | ❌ | ✅ | ✅ Resolved |

---

## Role Change — CHG-0001

David Osei promoted from IT Support to IT Manager. A role change is always a swap — remove the old role before assigning the new one. Adding without removing is how privilege creep starts. GRP-IT-Support removed and GRP-IT-Managers plus GRP-IT-Admins added in the same change window. Clean state, no overlap.

---

## Access Certification — Q1

11 accounts reviewed. 3 access rights removed. 1 SoD violation remediated. Zero open findings at close.

---

## Recommendations

**Run access certifications quarterly, not annually.** Annual reviews catch problems twelve months too late.

**Design the role matrix before creating any accounts.** Retrofitting an access control model onto an environment where accounts already exist is significantly more work than designing it correctly from the start.

**Treat SoD violations as security incidents, not compliance findings.** The difference in urgency matters. AF-002 was remediated within hours of detection.

---

## Personal Observation

Privilege creep is the default outcome in any environment where role changes are processed as additions rather than swaps. Nobody intends to leave stale access in place. It happens because the removal step is easy to skip when you are under time pressure to provision the new access quickly. Building the habit of treating every role change as a full swap — remove first, add second, verify after — prevents an entire category of access control failures.

---
---

# Lab 03 — Offboarding: When Someone Leaves, Everything Leaves With Them

**Completed:** April 2026 &nbsp;|&nbsp; **Phase:** 1 &nbsp;|&nbsp; **Environment:** VMware · Windows Server 2022 · Active Directory

---

## The Problem This Lab Solves

Provisioning gets the attention. Offboarding is where organisations get breached.

An ex-employee with active credentials is not a theoretical risk. The gap between "this person no longer works here" and "this person no longer has access" should be measured in minutes. Most organisations measure it in weeks, if they measure it at all.

This lab builds the offboarding procedure that closes that gap — for a standard resignation, a for-cause termination requiring legal hold, and a contractor account that auto-expired. Three different scenarios. One discipline: the moment the termination ticket arrives, the clock starts.

---

## Three Terminations Processed

### TERM-0001 — Fatima Al-Hassan — Voluntary Resignation
Standard offboarding. Account disabled first, groups removed, moved to Disabled_Users, description updated with ticket reference and deletion date, 30-day deletion reminder set. Under 20 minutes from ticket receipt to closure.

**Risk: LOW**

### TERM-0002 — Priya Sharma — For-Cause Termination
Different situation entirely. The first action on a for-cause termination is not to touch the account. It is to notify IT Security. If the employee is under investigation, the security team may need forensic evidence before any changes are made. Disabling first could destroy that evidence.

IT Security notified before any account changes. CISO verbal approval documented. Account disabled. Groups removed. Moved to Disabled_Users. Description flagged: `TERMINATED — FOR CAUSE — LEGAL HOLD — Do NOT delete before [90-day date]`. 90-day legal hold set. 85-day reminder set to contact Legal before the hold expires — five days of buffer so Legal has time to respond.

**Risk: CRITICAL — escalated, legal hold active**

### TERM-0003 — Jessica Park — Contractor Account Expiry
The expiry date set at provisioning in Lab 01 fired automatically. AD disabled the account on the correct date without any human intervention. Manual cleanup completed the offboarding — group removal, move to Disabled_Users, deletion reminder set.

This is why contractor expiry dates are set at provisioning, not at offboarding.

---

## The Hybrid Sync Gap

Azure AD Connect syncs every 30 minutes. When an account is disabled in AD, the cloud sessions remain active for up to 30 minutes. For a standard resignation that window is acceptable. For Priya Sharma's for-cause termination it is not.

The fix — implemented from Lab 07 onwards:

```
Step 1: Disable in AD                  → on-premises access cut immediately
Step 2: Revoke Entra ID sessions       → cloud access cut immediately
Step 3: Remove Azure licenses          → cloud entitlements removed
Step 4: Sync propagates disable state  → background, up to 30 minutes
```

Steps 2 and 3 do not wait for the sync cycle. They happen in parallel with Step 1.

---

## Recommendations

**Disable before anything else — every time.** Disabling cuts all active sessions immediately. Removing groups while the account is still enabled creates a window of partial access. Disable first, clean up second.

**Never delete immediately — always retain.** 30 days standard. 90 days for for-cause. The audit trail, group membership history, and mailbox content may all be needed for legal proceedings. The cost of retaining a disabled account is zero. The cost of not having it when Legal asks is significant.

**Set the legal hold reminder five days before expiry.** A reminder on day 90 of a 90-day hold is too late to act on.

---

## Personal Observation

This lab changed how I think about the IAM analyst's role in a for-cause termination. Until working through this scenario, I thought of offboarding as an access management task. The for-cause procedure made it clear that offboarding is sometimes the opening step of a security investigation — and the IAM analyst is the first responder. The sequence matters: notify Security, then act. Not the other way around.

---
---

# Lab 04 — Account Lockout Investigation: Reading Logs Like a Security Analyst

**Completed:** April 2026 &nbsp;|&nbsp; **Phase:** 1 &nbsp;|&nbsp; **Environment:** VMware · Windows Server 2022 · Event Viewer · PowerShell

---

## The Problem This Lab Solves

Account lockout tickets are the most common IAM helpdesk request in any enterprise. Most are innocent. But occasionally a lockout is the first visible sign of something more serious — multiple accounts failing authentication in rapid succession from the same source, service accounts hit with repeated attempts outside business hours, admin accounts targeted with automated credential spraying.

The lockout looks the same. The cause is completely different. Knowing which is which — and knowing how to prove it — is what separates an IAM analyst from someone who just unlocks accounts.

---

## Event IDs That Matter

| Event ID | What it records | Key fields |
|---|---|---|
| 4625 | Failed logon | Account, Logon Type, Failure Reason, Source Address |
| 4740 | Account locked out | Account, Caller Computer Name |
| 4648 | Explicit credential use | Account, Target, Network address |
| 4771 | Kerberos pre-auth failed | Account, IP, Failure Code |
| 4776 | NTLM credential validation | Account, Error Code, Workstation |

**4740 tells you what locked and what caused it. 4625 tells you why and from where. Cross-referencing both gives you the full picture.**

---

## Three Scenarios Investigated

### Scenario A — Marcus Williams — Simple Password Mistake
Single account. Single source. Logon Type 3 (network). Six failed attempts. No unusual pattern in surrounding logs.

**Resolved at helpdesk. Risk: LOW**

### Scenario B — David Osei — Stale Cached Credentials
Rapid repeated attempts — the pattern of a service retrying, not a human typing. Event ID 4648 present alongside the lockout, confirming explicit credential use. Root cause: a scheduled task or Windows service configured with personal account credentials after a recent password change.

**Resolved. IT Manager advised to migrate automated processes to a dedicated service account. Risk: MEDIUM**

### Scenario C — Linda Okafor, Rachel Thompson, Michael Obi — Spray Pattern
Three accounts locked out within minutes of each other. Same password. Same source. Automated cadence.

```powershell
$failedLogons | Group-Object AccountName |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Output confirmed identical failed attempt counts across three accounts in a compressed time window. Pattern consistent with a password spray attack — one password attempted across many accounts, staying below the per-account lockout threshold.

**This did not go to helpdesk. It went to IT Security. SEC-0001 opened. Risk: CRITICAL**

---

## Escalation Decision Framework

| Signal | Routine | Escalate |
|---|---|---|
| Accounts affected | 1 | 3+ in short window |
| Source machine | Known internal | Unknown or unusual |
| Time | Business hours | Off-hours or weekend |
| Logon type | 2 (interactive), 7 (unlock) | 3 (network), 8 (cleartext), 10 (RDP) |
| Attempt pattern | Spread over time | Rapid, automated |
| Account type targeted | Standard user | Admin or service account |
| Event ID 4648 present | No | Yes |

Three or more escalation signals → open a security incident alongside the helpdesk ticket.

---

## On-Prem vs Cloud Log Comparison

| Capability | Event Viewer | Entra ID Sign-in Logs |
|---|---|---|
| Location data | Workstation name / IP | Country, city, coordinates |
| Risk scoring | None | Built-in (Low/Medium/High) |
| Impossible travel | No | Yes |
| Leaked credential detection | No | Yes — dark web feeds |
| MFA result | No | Yes |
| Conditional Access outcome | No | Yes |

On-premises Event Viewer gives you machine-level authentication detail. Entra ID gives you context-rich identity intelligence. In a hybrid environment you need both.

---

## Recommendations

**Configure advanced audit policies before you need them.** Default Windows Server settings do not capture everything required. An environment that has not been generating the right events cannot be investigated retroactively.

**Use PowerShell for pattern analysis, not just Event Viewer.** The spray pattern in Scenario C was not visible by browsing individual events. It was visible by running a group-by query across all failed logons. Build the habit of scripting log analysis for anything beyond a single account investigation.

**Document your escalation framework before an incident.** The decision about whether a lockout is a helpdesk ticket or a security incident should be made against a documented standard, not improvised under pressure.

---

## Personal Observation

Account lockouts have a shape. A single account locking out once has one shape. Three accounts locking out in the same window with the same pattern has a completely different shape. Recognising that difference does not come from memorising Event IDs. It comes from knowing what the logs look like when everything is normal — so that when something is not normal, it stands out immediately.

---
---

# Lab 05 — Azure AD Connect: Building the Hybrid Identity Bridge

**Completed:** April 2026 &nbsp;|&nbsp; **Phase:** 2 &nbsp;|&nbsp; **Environment:** VMware · Windows Server 2022 · Microsoft Entra Connect · Azure Entra ID P2

---

## The Problem This Lab Solves

Labs 01 through 04 built a solid on-premises identity foundation. All of it existed in isolation. Users were in Active Directory. The cloud was empty.

This lab closes that gap. Azure AD Connect bridges the on-premises FictionalCorp.local domain to the Entra ID tenant at fictionalcorp.onmicrosoft.com. Once configured, a user created in AD exists in Entra ID. A password changed on-premises propagates to the cloud. A group membership updated in AD reflects in cloud application assignments. The two environments stop being separate things and become one coherent identity system.

Every lab from here onwards depends on that bridge being in place.

---

## What Was Configured

**Sign-in method: Password Hash Synchronization**  
AD password hashes sync to Entra ID. FictionalCorp users authenticate to cloud services with their on-premises credentials. Also enables leaked credential detection — Microsoft compares synced hashes against breach databases and flags matches as high-risk sign-ins.

**OU Filtering:**

| OU | Synced | Reason |
|---|---|---|
| HR, Finance, IT, Sales, Contractors, Service_Accounts | ✅ | Active identities |
| **Disabled_Users** | ❌ | Terminated accounts must not appear in Entra ID |
| Domain Controllers, Computers | ❌ | Computer objects — not relevant |

**Password Writeback: Enabled**  
Allows SSPR password resets in Entra ID to write back to on-premises AD. Required for Lab 08.

---

## Obstacles Encountered

Three issues came up during installation — all worth documenting because they are common in real enterprise deployments.

**TLS 1.2 not configured.**  
The installer blocked immediately. Windows Server 2022 does not enable TLS 1.2 for .NET Framework by default. Fixed by setting `SchUseStrongCrypto` in the 32-bit and 64-bit .NET Framework registry paths and enabling TLS 1.2 in SCHANNEL protocols. Server restart required.

**IE Enhanced Security Configuration blocking the Microsoft login page.**  
Azure AD Connect uses an embedded browser for Azure authentication. IE Enhanced Security Configuration blocked the login page on first attempt. Fixed by disabling IE Enhanced Security Configuration in Server Manager.

**JavaScript blocked in Internet Explorer.**  
After the Enhanced Security fix, the login page loaded but threw a JavaScript error. Fixed by enabling Active Scripting in IE security settings. In production this is resolved by setting Edge as the default browser on the sync server before installation — IE should not be handling modern authentication flows.

---

## Verification

After the initial sync, 13 FictionalCorp users appeared in Entra ID — every one showing **On-premises sync enabled: Yes** and **On-premises domain name: FictionalCorp.local**.

Live sync verified by updating Marcus Williams' department attribute in AD, forcing a delta sync, and confirming the change appeared in Entra ID within two minutes:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

---

## The Hybrid Sync Gap

Azure AD Connect syncs on a 30-minute cycle. When an account is disabled in AD, cloud sessions remain active for up to 30 minutes. This gap is acceptable for standard offboarding. For for-cause terminations it is a security exposure. The fix — manually revoking Entra ID sessions immediately alongside the AD disable — is implemented in Lab 07.

---

## Recommendations

**Set Edge as the default browser on any Azure AD Connect server before installation.** Every obstacle in this installation traces back to IE being the embedded authentication browser. One setting change before starting would have saved an hour of troubleshooting.

**Monitor sync health, not just sync results.** Most teams check whether users appeared in Entra ID and consider the job done. The more important thing to monitor is whether the sync service is running and completing successfully:

```powershell
Get-ADSyncScheduler | Select-Object LastSyncTime, SyncCycleEnabled
Get-ADSyncConnectorRunStatus
```

---

## What I Would Do Differently in Production

Add a routable UPN suffix before configuring Azure AD Connect. The `.local` domain creates a UPN mismatch warning during installation and means synced users authenticate with the tenant domain rather than a clean company UPN. This is a straightforward fix before the first sync runs and a meaningful amount of work to change after hundreds of accounts exist.

---

## Personal Observation

The obstacles in this lab — TLS, IE Enhanced Security, JavaScript — are not interesting problems. They are friction. But they are exactly the kind of friction that consumes hours in production when engineers are not expecting it. Documenting each one, understanding why it occurred, and knowing how to prevent it is the actual value of working through it manually rather than scripting around it.

The moment the sync completed and 13 FictionalCorp users appeared in Entra ID with On-premises sync: Yes — that is when the two environments became one. Everything built in Labs 01 through 04 is now a hybrid identity system.

---
---

## What Comes Next

**Lab 06 — Conditional Access Policies**  
With FictionalCorp users synced to Entra ID and Entra ID P2 active, Conditional Access policies can now be applied. Lab 06 builds three policies that mirror real enterprise security requirements: blocking legacy authentication protocols, enforcing MFA for privileged roles, and restricting access by location and device compliance state.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*  
*Last updated: April 2026*
