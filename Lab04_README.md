# Lab 04 — Account Lockout Investigation: Reading Logs Like a Security Analyst

**Phase:** 1 of 5 &nbsp;|&nbsp; **Lab:** 04 of 20 &nbsp;|&nbsp; **Completed:** April 2026  
**Environment:** VMware Workstation · Windows Server 2022 · Active Directory · Event Viewer · PowerShell  
**Domain:** FictionalCorp.local · DC01 · 192.168.10.10

---

## The Problem This Lab Solves

Account lockout tickets are the most common IAM helpdesk request in any enterprise environment. Most are innocent — a user typed the wrong password, their phone is still authenticating with credentials they changed last week, a scheduled task is running under an account with an expired password. Routine.

But occasionally a lockout is the first visible sign of something more serious. Multiple accounts failing authentication in rapid succession from the same source. Service accounts being hit with repeated attempts outside business hours. Admin accounts targeted with automated credential spraying. The lockout is the same. The cause is completely different. Knowing which is which — and knowing how to prove it — is what separates an IAM analyst from someone who just unlocks accounts.

This lab builds that investigation capability from scratch.

---

## Event IDs That Matter

These five Event IDs are the core of any account lockout investigation on a Windows domain. Knowing them without looking them up is a baseline expectation for any IAM or security analyst role.

| Event ID | What it records | Key fields |
|---|---|---|
| 4625 | Failed logon attempt | Account name, Logon Type, Failure Reason, Source Address, Workstation |
| 4740 | Account locked out | Account name, Caller Computer Name |
| 4648 | Logon using explicit credentials | Account, Target account, Network address |
| 4771 | Kerberos pre-authentication failed | Account, IP Address, Failure Code |
| 4776 | NTLM credential validation | Account, Error Code, Workstation |

**Event ID 4740 tells you the account that locked out and the machine that caused it.**  
**Event ID 4625 tells you why the authentication failed and from where.**  
Cross-referencing them gives you the full picture.

**Logon Type codes from Event ID 4625:**

| Type | Meaning | Why it matters |
|---|---|---|
| 2 | Interactive — user at the keyboard | Normal user activity |
| 3 | Network — file share, mapped drive | Common for service failures |
| 7 | Unlock — screen unlock attempt | Usually a forgotten PIN |
| 8 | NetworkCleartext — legacy authentication | Flags legacy protocols — security concern |
| 10 | RemoteInteractive — RDP | Warrants closer attention if unexpected |

---

## Audit Policy Configuration

Before Event Viewer captures anything useful, the domain controller needs to be configured to generate the right events. Default Windows Server audit settings do not log everything needed for a lockout investigation.

**Advanced Audit Policy configured via Group Policy:**

| Category | Policy | Setting |
|---|---|---|
| Account Logon | Credential Validation | Success and Failure |
| Account Logon | Kerberos Authentication | Success and Failure |
| Account Management | User Account Management | Success and Failure |
| Logon/Logoff | Logon | Success and Failure |
| Logon/Logoff | Account Lockout | Success and Failure |

Verified with:
```powershell
auditpol /get /category:*
```

This step matters in production. An environment that has not been configured to generate the right audit events cannot be investigated after the fact. The logs either exist or they do not.

---

## Three Scenarios Investigated

---

### Scenario A — Marcus Williams — Simple Password Mistake

**What happened:** Marcus locked his account attempting to authenticate to a network share with the wrong password six times in quick succession.

**Investigation:**

Event ID 4740 → Caller Computer Name: DC01  
Event ID 4625 → Logon Type 3 (Network) · Failure: Unknown username or bad password · 6 attempts

**Assessment:** Routine. Single account, single source machine, straightforward failure reason, no unusual pattern in surrounding log entries.

**Resolution:** Account unlocked. User advised on credential management.

**Risk rating: LOW**

---

### Scenario B — David Osei — Stale Cached Credentials

**What happened:** David's account locked out from rapid, repeated authentication attempts — the kind of pattern that does not come from a human typing a password. It comes from a service or application retrying with credentials that are no longer valid.

**Investigation:**

Event ID 4740 → Caller Computer Name: DC01  
Event ID 4625 → Logon Type 3 (Network) · 6 attempts in rapid succession  
Event ID 4648 → Explicit credential use detected

The presence of Event ID 4648 alongside the rapid repeated attempts pointed to a process using cached credentials — most likely a Windows service, scheduled task, or mapped drive configured with David's personal account credentials after a recent password change.

**Assessment:** Not a security incident. A configuration gap — automated processes should not be using personal account credentials. The immediate cause is a stale cached password. The underlying cause is an account management practice that should be corrected.

**Resolution:** Account unlocked. IT Manager advised to audit scheduled tasks and services running under personal account credentials. Recommendation made to use a dedicated service account (svc.financecore pattern) for any automated processes.

**Risk rating: MEDIUM — configuration gap, not malicious**

---

### Scenario C — Linda Okafor, Rachel Thompson, Michael Obi — Spray Pattern

**What happened:** Three separate accounts locked out within a short time window. Same password attempted against each. Rapid, automated pattern.

**Investigation:**

Event ID 4740 → Three lockout events within minutes of each other  
Event ID 4625 → Same source, same failure pattern, three different accounts  
PowerShell analysis confirmed the pattern:

```powershell
$failedLogons | Group-Object AccountName |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

Output showed l.okafor, r.thompson, and m.obi each with identical failed attempt counts in a compressed time window from the same source.

**Assessment:** This pattern is consistent with a password spray attack — an attacker who has obtained a list of valid usernames and is attempting a common password against all of them, staying below the per-account lockout threshold while spreading attempts across multiple accounts. In a production environment this would be treated as a confirmed security incident.

**This did not go to helpdesk. This went to IT Security.**

**Escalation:** Security incident SEC-0001 opened. IT Security notified. Enhanced monitoring enabled. Source address investigation initiated. All three accounts unlocked only after IT Security confirmed the source was the lab simulation.

**Risk rating: CRITICAL — escalated as security incident**

---

## PowerShell Log Analysis

Event Viewer is the starting point. PowerShell is where the real analysis happens — especially when you need to identify patterns across hundreds of events rather than reading them one at a time.

**Extract all lockout events from today:**
```powershell
$events = Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4740
    StartTime = (Get-Date).Date
} | Select-Object TimeCreated,
    @{N='LockedAccount'; E={$_.Properties[0].Value}},
    @{N='CallerMachine';  E={$_.Properties[1].Value}}

$events | Export-Csv -Path "C:\Lockout_Report_$(Get-Date -Format 'yyyyMMdd').csv" -NoTypeInformation
```

**Identify accounts with the most failed attempts:**
```powershell
$failedLogons = Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    Id        = 4625
    StartTime = (Get-Date).Date
} | Select-Object TimeCreated,
    @{N='AccountName';   E={$_.Properties[5].Value}},
    @{N='LogonType';     E={$_.Properties[10].Value}},
    @{N='SourceAddress'; E={$_.Properties[19].Value}}

$failedLogons | Group-Object AccountName |
    Select-Object Name, Count |
    Sort-Object Count -Descending
```

The spray pattern in Scenario C was identified by running this second script and seeing three accounts with identical counts in a compressed window. That output was the evidence that escalated this from a helpdesk ticket to a security incident.

---

## Escalation Decision Framework

Not every lockout is the same. This framework determines whether a lockout gets resolved at the helpdesk or escalated to the security team.

| Signal | Routine | Escalate |
|---|---|---|
| Accounts affected | 1 | 3 or more in short window |
| Source machine | Known internal | Unknown, external, or unusual |
| Time of lockout | Business hours | Off-hours, weekend, holiday |
| Logon type | 2 (interactive), 7 (unlock) | 3 (network), 8 (cleartext), 10 (RDP) |
| Attempt pattern | Spread over time | Rapid, automated cadence |
| Account type targeted | Standard user | Admin, service account, or executive |
| Same password tried | No | Yes — same pattern across multiple accounts |
| Event ID 4648 present | No | Yes |

**Rule of thumb:** Three or more escalation signals in a single investigation → open a security incident alongside the helpdesk ticket.

---

## On-Prem vs Cloud Log Comparison

One of the more important comparisons this lab surfaced — what on-premises Event Viewer captures versus what Entra ID Sign-in Logs capture for the same type of event.

| Capability | Event Viewer (On-Prem) | Entra ID Sign-in Logs |
|---|---|---|
| Location data | Workstation name / internal IP | Country, city, coordinates |
| Risk scoring | None | Built-in (Low / Medium / High) |
| Impossible travel detection | No | Yes |
| Leaked credential detection | No | Yes — dark web feeds |
| MFA result visibility | No | Yes |
| Conditional Access outcome | No | Yes |
| Retention | Depends on log size settings | 30 days free / 90 days P1+ |

On-premises Event Viewer gives you machine-level authentication detail. Entra ID Sign-in Logs give you context-rich identity intelligence. In a hybrid environment you need both. The Scenario C spray pattern was visible in Event Viewer. In Entra ID Identity Protection, a similar attack would also generate a risk alert with an automated response — blocking the sign-in before accounts even lock out.

---

## Threat Connections

**Account lockouts are a lagging indicator — the attack may already have succeeded.**  
A lockout means the attacker attempted more than the threshold allows. It does not mean they failed. If they attempted five passwords and the lockout threshold is six, they may have found a valid one before the lockout fired. An account lockout investigation should always include a check for successful authentications alongside the failed ones.

**Password spray attacks are designed to evade per-account lockout thresholds.**  
The attacker does not try ten passwords against one account — they try one password against ten accounts. Each account only sees one or two failed attempts, well below the lockout threshold. The pattern is only visible when you look across all accounts simultaneously. This is why the PowerShell analysis that groups by account and sorts by count is more valuable than reading individual Event Viewer entries.

**Cached credentials after a password change are a configuration vulnerability, not a user error.**  
When a user changes their password but a service or application is still authenticating with the old one, the lockouts will continue until someone finds and updates every location where the old credential is stored. The underlying problem is that personal account credentials were used for automated processes. The solution is dedicated service accounts with managed credentials — not password reset loops.

---

## Recommendations

**Configure advanced audit policies before you need them.**  
Default Windows Server audit settings do not capture everything required for a thorough investigation. The time to configure `auditpol` and verify the output is before an incident, not during one. An environment that has not been generating the right events for the past six months cannot be retroactively investigated.

**Use PowerShell for log analysis, not just Event Viewer.**  
Event Viewer is adequate for investigating a single event. It is not adequate for identifying patterns across hundreds of events in a compressed time window. The spray pattern in this lab was not visible by browsing Event Viewer — it was visible by running a group-by query in PowerShell and seeing three accounts with matching counts. Build the habit of scripting log analysis rather than reading logs manually.

**Build an escalation framework before an incident forces you to improvise one.**  
The decision about whether a lockout is a helpdesk ticket or a security incident should be made against a documented framework, not in the moment under pressure. The framework in this lab gives a consistent answer regardless of who is handling the ticket or what time of day it arrives.

---

## Personal Observation

The most useful thing this lab taught me is that account lockouts have a shape. A single account locking out has one shape. Three accounts locking out in the same time window with the same pattern has a completely different shape. Reading that shape correctly — and acting on the difference between the two — is a skill that does not come from knowing what Event IDs mean. It comes from having looked at enough logs to recognise when something does not fit the routine pattern.

The spray scenario was simulated. But the pattern it produced in the logs was identical to what a real password spray attack produces. Recognising it required looking at the data the right way — grouped by account, sorted by count, compared against a time window. That is the investigation discipline this lab was designed to build.

---

## Connection to Azure

From Lab 05 onwards, the on-premises Event Viewer investigation capability built here runs in parallel with Entra ID Sign-in Log analysis. A lockout investigation in the hybrid environment covers both layers — Event Viewer for on-premises authentication failures, Entra ID for cloud authentication context, risk signals, and Conditional Access outcomes. Lab 15 extends this further with Entra ID Identity Protection, which automates the risk detection that this lab performs manually.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*
