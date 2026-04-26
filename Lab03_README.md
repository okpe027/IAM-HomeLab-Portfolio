# Lab 03 — Offboarding: When Someone Leaves, Everything Leaves With Them

**Phase:** 1 of 5 &nbsp;|&nbsp; **Lab:** 03 of 20 &nbsp;|&nbsp; **Completed:** April 2026  
**Environment:** VMware Workstation · Windows Server 2022 · Active Directory  
**Domain:** FictionalCorp.local · DC01 · 192.168.10.10

---

## The Problem This Lab Solves

Provisioning gets the attention. Offboarding is where organisations get breached.

An ex-employee with active credentials is not a theoretical risk — it is one of the most consistent findings in security audits and one of the most common vectors for data theft. The gap between "this person no longer works here" and "this person no longer has access" should be measured in minutes, not days. Most organisations measure it in weeks, if they measure it at all.

This lab builds the offboarding procedure that closes that gap — for a standard resignation, for a for-cause termination that requires legal hold, and for a contractor account that auto-expired. Three different scenarios. Three different procedures. One discipline: the moment the termination ticket arrives, the clock starts.

---

## Three Terminations Processed

---

### TERM-0001 — Fatima Al-Hassan — Voluntary Resignation

```
Priority:      Standard
Employee:      Fatima Al-Hassan  |  f.alhassan  |  Sales Representative
Last day:      April 2026
Access held:   GRP-Sales-Reps, SalesPro CRM, Email
Legal hold:    No — 30-day retention before deletion
```

Standard offboarding. No complications. The kind of ticket that arrives twenty times a month in a large organisation and needs to be processed consistently every time without cutting corners.

**Pre-offboarding audit confirmed:** one group membership, one CRM system, email access. Nothing unusual.

**Execution sequence:**
1. AD account disabled — first action, always
2. Removed from GRP-Sales-Reps
3. Moved to Disabled_Users OU
4. Account description updated with termination date and ticket reference
5. 30-day deletion reminder set
6. Provisioning log updated

Total time: under 20 minutes from ticket receipt to closure.

---

### TERM-0002 — Priya Sharma — For-Cause Termination

```
Priority:      IMMEDIATE
Employee:      Priya Sharma  |  p.sharma  |  HR Coordinator
Effective:     Immediately
Access held:   GRP-HR-Coordinators, PeopleHub HR System, Email
Legal hold:    YES — 90 days minimum — do NOT delete
Reason:        Policy violation — confidential
```

Different situation entirely. For-cause terminations require a different mindset before they require different steps.

The first action on a for-cause termination is not to touch the account. It is to notify IT Security. If the employee is under investigation, the security team may need to capture forensic evidence from the account in its current state. Disabling first could destroy evidence before it is collected.

**Notification sent to IT Security before any account changes.**

**Execution sequence:**
1. IT Security notified — documented with timestamp
2. CISO verbal approval confirmed and logged
3. Pre-offboarding audit run — one group, PeopleHub access, no undisclosed delegated permissions
4. AD account disabled
5. Removed from GRP-HR-Coordinators
6. Delegated permissions audited — none found
7. Moved to Disabled_Users OU
8. Description updated: `TERMINATED — FOR CAUSE — LEGAL HOLD — Do NOT delete before [90-day date]`
9. 90-day legal hold set — not the standard 30 days
10. 85-day calendar reminder set to contact Legal before hold expires
11. Activity logs preserved — confirmed with IT Security
12. Provisioning log updated with all timestamps and notifications

The 85-day reminder is deliberate. Waiting until day 90 to contact Legal about the hold expiry leaves no response time. Five days of buffer is the professional standard.

---

### TERM-0003 — Jessica Park — Contractor Account Expiry

```
Priority:      Scheduled
Contractor:    Jessica Park  |  j.park.contractor  |  IT Support Contractor
Contract end:  90 days from Lab 01 provisioning date
```

This one required the least effort because the hard part was already done in Lab 01. When the contract end date arrived, Active Directory disabled the account automatically. No ticket. No human memory required. The expiry date set at provisioning did its job.

Manual cleanup completed the offboarding:
1. Auto-disable confirmed — account already disabled by AD
2. Removed from GRP-IT-Contractors
3. Moved to Disabled_Users OU
4. Description updated with expiry date and ticket reference
5. 30-day deletion reminder set

This is exactly why contractor account expiry dates are set at provisioning, not at offboarding.

---

## Environment State After All Three Offboardings

**Disabled_Users OU — three accounts confirmed:**

| Username | Type | Legal Hold | Delete Date |
|---|---|---|---|
| f.alhassan | Voluntary resignation | No | 30 days from termination |
| p.sharma | For cause | YES — 90 days | Not before legal review |
| j.park.contractor | Contract expiry | No | 30 days from expiry |

**All active staff accounts confirmed unaffected.** Eight remaining accounts — Marcus Williams, James Adeyemi, Linda Okafor, Alex Chen, Rachel Thompson, Michael Obi, David Osei, Karen Nwosu — all enabled, all in correct OUs, all group memberships intact.

---

## The Hybrid Sync Gap

This lab surfaced a security consideration that matters more in a hybrid environment than a purely on-premises one.

Azure AD Connect syncs on a 30-minute cycle. When an account is disabled in on-premises AD, that disable state takes up to 30 minutes to propagate to Entra ID. During that window, the terminated employee's cloud sessions — email, SharePoint, Teams, any cloud application — remain fully active.

For a standard resignation, that window is acceptable. For Priya Sharma's for-cause termination, it is not.

The fix, implemented from Lab 07 onwards: immediately after disabling in AD, manually revoke all Entra ID sessions without waiting for the sync cycle.

```
Step 1: Disable in AD                    → cuts on-premises access immediately
Step 2: Revoke Entra ID sessions         → cuts cloud access immediately  
Step 3: Remove Azure licenses            → removes cloud service entitlements
Step 4: AD Connect sync propagates       → background process, up to 30 minutes
```

Steps 2 and 3 do not wait for the sync. They happen in parallel with Step 1.

---

## Threat Connections

**Lingering access after termination is the most common source of insider threat incidents.**  
It is rarely malicious intent from the outset. It is an ex-employee who discovers they still have access and makes a decision about what to do with it. The window between "no longer employed" and "no longer has access" is the risk. This lab closes that window to minutes.

**For-cause terminations are an active forensic scenario, not just an access management task.**  
The reason notification to IT Security precedes any account changes is that account state is evidence. An account that was actively being used for malicious activity at the time of termination may contain information critical to an investigation. Disabling it before the security team has captured what they need destroys that information permanently.

**The 30-minute hybrid sync gap is an exploitable window.**  
An employee who knows they are about to be terminated and who has cloud access can exfiltrate data, send emails, or access sensitive documents during the time between their on-premises account being disabled and their cloud sessions being revoked. For for-cause terminations, this window should be treated as a security risk and closed manually, not left to the sync schedule.

---

## Recommendations

**Disable before anything else — every time, without exception.**  
The temptation when processing a termination under time pressure is to start removing group memberships while the account is still active. Do not. Disable first. Disabling cuts all active authentication sessions immediately. Removing groups while the account is still enabled means there is a window where the user has fewer permissions but is still authenticated — which is messier than just cutting access cleanly at the start.

**Never delete — retain.**  
Immediate deletion destroys the audit trail. The account's activity history, group membership log, and any associated mailbox content may all be needed for legal proceedings, access reviews, or incident investigations. Thirty days of retention for standard terminations. Ninety days for for-cause. The cost of retaining a disabled account is essentially zero. The cost of not having it when Legal asks for it can be significant.

**Set the legal hold reminder five days before the hold expires, not on the expiry date.**  
This gives Legal time to respond before the hold date passes. A reminder on day 90 of a 90-day hold is a reminder that the decision should have already been made.

**Document the for-cause notification chain as it happens, not after.**  
Who was notified, at what time, through what channel. This record may be needed in a legal or regulatory context. Writing it from memory afterwards is less reliable and less credible than timestamped contemporaneous notes.

---

## What I Would Do Differently in Production

The offboarding checklist in this lab is a manual spreadsheet. In a production environment at scale, this should be an automated workflow — a termination event in the HR system triggers a sequence of actions across AD, Entra ID, Okta, and downstream applications without requiring a human to work through a checklist under time pressure.

The checklist is the right tool for understanding what the workflow should contain. Automation is the right tool for executing it reliably at scale. SailPoint, ServiceNow, and similar platforms handle this — the HR system fires a termination event, the IAM platform executes the deprovisioning sequence, and the audit trail is generated automatically.

---

## Personal Observation

The for-cause termination scenario changed how I think about the relationship between IAM and security operations. Until this lab, I thought of offboarding as an access management task — technical steps to revoke credentials and group memberships. The for-cause procedure made it clear that offboarding is sometimes the first step in a security investigation, and the IAM analyst is the first responder.

That reframe matters. It means the first question on a for-cause termination is not "what do I need to revoke?" It is "what does the security team need before I touch anything?" Getting that sequence right is the difference between preserving evidence and destroying it.

---

## Connection to Azure

The offboarding checklist built in this lab has placeholder steps for Entra ID session revocation and license removal — marked as Lab 07 items. When Azure AD Connect is configured in Lab 05, those steps become active parts of every offboarding procedure. The hybrid sync gap documented here is addressed directly in Lab 07 with the manual session revocation workflow.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*
