# Lab 02 — RBAC Design: Building the Access Control Model

**Phase:** 1 of 5 &nbsp;|&nbsp; **Lab:** 02 of 20 &nbsp;|&nbsp; **Completed:** April 2026  
**Environment:** VMware Workstation · Windows Server 2022 · Active Directory · Excel  
**Domain:** FictionalCorp.local · DC01 · 192.168.10.10

---

## The Problem This Lab Solves

Most access control failures do not happen because someone broke through a security control. They happen because the access control was never designed properly in the first place. A user who can both create and approve invoices. An engineer who still has Finance system access six months after moving to a different team. A contractor who accumulated permissions across three role changes and nobody noticed.

This lab builds the role matrix that prevents all of that — before a single account is created.

The discipline here is designing before building. Every group in Active Directory, every user assignment, every permission decision traces back to a deliberate row in a documented role matrix. If it is not in the matrix, it does not get built. If it is built but not in the matrix, that is an audit finding.

---

## What Was Built

### Role Inventory

Eleven distinct roles identified across FictionalCorp's four departments. A role is not just a job title — it is a unique combination of access requirements. Two job titles that need identical access share a role. Two that need different access get different roles.

| Role | Department | Type | Access characteristic |
|---|---|---|---|
| HR-Manager | HR | Permanent | Full PeopleHub access |
| HR-Coordinator | HR | Permanent | Standard HR operations |
| Finance-Manager | Finance | Permanent | FinanceCore + invoice approve only |
| Finance-Analyst | Finance | Permanent | FinanceCore + invoice create only |
| Finance-Approver | Finance | Permanent | FinanceCore + invoice approve only |
| IT-Manager | IT | Permanent | Full ITConsole + admin rights |
| IT-Support | IT | Permanent | ITConsole — no admin rights |
| IT-Contractor | IT | Contractor | ITConsole read-only — time-limited |
| Sales-Manager | Sales | Permanent | SalesPro + HR reports read access |
| Sales-Rep | Sales | Permanent | SalesPro only |
| Service-Account | N/A | Non-human | App-to-app only — no interactive login |

### Role Matrix

The matrix maps every role to its permitted access across six FictionalCorp systems. Three access levels: Allow, Deny, Read-Only. Every cell is a decision.

| Role | AD | Email | FinanceCore | PeopleHub | SalesPro | ITConsole | Inv.Create | Inv.Approve | Admin |
|---|---|---|---|---|---|---|---|---|---|
| HR-Manager | Allow | Allow | Deny | Allow | Deny | Deny | Deny | Deny | Deny |
| HR-Coordinator | Allow | Allow | Deny | Allow | Deny | Deny | Deny | Deny | Deny |
| Finance-Manager | Allow | Allow | Allow | Read-Only | Deny | Deny | Deny | Allow | Deny |
| Finance-Analyst | Allow | Allow | Allow | Deny | Deny | Deny | Allow | Deny | Deny |
| Finance-Approver | Allow | Allow | Allow | Deny | Deny | Deny | Deny | Allow | Deny |
| IT-Manager | Allow | Allow | Deny | Deny | Deny | Allow | Deny | Deny | Allow |
| IT-Support | Allow | Allow | Deny | Deny | Deny | Allow | Deny | Deny | Deny |
| IT-Contractor | Allow | Deny | Deny | Deny | Deny | Read-Only | Deny | Deny | Deny |
| Sales-Manager | Allow | Allow | Deny | Read-Only | Allow | Deny | Deny | Deny | Deny |
| Sales-Rep | Allow | Allow | Deny | Deny | Allow | Deny | Deny | Deny | Deny |
| Service-Account | Deny | Deny | Allow | Deny | Deny | Deny | Deny | Deny | Deny |

### Security Groups Built in Active Directory

14 security groups across five OUs, each mapping directly to a role in the matrix.

```
HR/               GRP-HR-Managers, GRP-HR-Coordinators
Finance/          GRP-Finance-Managers, GRP-Finance-Analysts, GRP-Finance-Approvers
                  GRP-Finance-InvoiceCreate, GRP-Finance-InvoiceApprove
IT/               GRP-IT-Managers, GRP-IT-Support, GRP-IT-Contractors, GRP-IT-Admins
Sales/            GRP-Sales-Managers, GRP-Sales-Reps
Service_Accounts/ GRP-ServiceAccounts
```

### Users Provisioned

11 accounts across four departments plus one service account — all processed from tickets, all logged in the provisioning spreadsheet before creation.

---

## The SoD Policy

Segregation of Duties exists because some combinations of access create fraud risk without any technical safeguard. If one person can create an invoice and approve it, they can authorize payments to themselves. The SoD policy documents which role combinations are prohibited before they get built into the environment.

Three conflict pairs defined:

**SOD-001 — Finance-Analyst + Finance-Approver**  
One person creating and approving their own invoices. Enables financial fraud. Regulatory reference: SOX Section 404.

**SOD-002 — IT-Support + Admin Rights**  
IT Support holding administrative privileges. Privilege escalation risk. Reference: ISO 27001 A.9.2.3.

**SOD-003 — Human role + Service-Account**  
Service accounts shared with or assigned to human users. Non-auditable access. Reference: NIST SP 800-53 AC-6.

---

## Privilege Creep — Detected and Remediated

This is where the lab gets interesting.

Alex Chen was promoted from Finance Analyst to Finance Manager three months ago. Whoever processed the role change added him to the Finance Manager group — correctly — but never removed him from the Finance Analyst group. They also added Invoice Approve access for the manager role but left Invoice Create access from the analyst role in place.

The result: Alex could create invoices and approve them simultaneously. That is a CRITICAL SoD violation sitting on top of a HIGH privilege creep finding.

**Before remediation:**

| User | Finance-Analysts | Finance-Managers | InvoiceCreate | InvoiceApprove | SoD violation |
|---|---|---|---|---|---|
| a.chen | YES | YES | YES | YES | CRITICAL |

**Findings raised:**

- **AF-001 — Privilege Creep (HIGH):** Stale Finance Analyst group membership retained after promotion. User holds permissions from both old and new role simultaneously.
- **AF-002 — SoD Violation (CRITICAL):** Member of both InvoiceCreate and InvoiceApprove groups. Direct breach of SOD-001. SOX exposure.

**After remediation:**

| User | Finance-Analysts | Finance-Managers | InvoiceCreate | InvoiceApprove | SoD violation |
|---|---|---|---|---|---|
| a.chen | No | YES | No | YES | Resolved |

Both findings closed same day.

---

## Role Change — CHG-0001

A role change ticket arrived for David Osei — promoted from IT Support to IT Manager following Karen Nwosu's departure.

The rule on role changes: it is always a swap, never an addition. Remove the old role before assigning the new one. Adding only the new groups without removing the old ones is how privilege creep starts. David's old IT Support membership was removed in the same change window the new IT Manager and IT Admins groups were added. Clean state, no overlap.

---

## Access Certification — Q1

A quarterly access review was run across all FictionalCorp accounts. Every user's current group membership was reviewed and either certified as appropriate or flagged for remediation.

**Result:** 11 accounts reviewed. 3 access rights removed (privilege creep on a.chen). 1 SoD violation remediated. Zero open findings at close.

---

## Threat Connections

**Privilege creep is one of the most common paths to insider threats.**  
It rarely involves a malicious actor deliberately accumulating access. It happens through incomplete role changes, forgotten contractor permissions, and access granted for a project that was never removed when the project ended. The fix is not better intentions — it is a quarterly access certification cycle that catches what daily operations miss.

**SoD violations create conditions for fraud that no other control can compensate for.**  
If one person controls an entire transaction from creation to approval, technical security controls become irrelevant. They can authorize anything. SoD is not a compliance box to tick — it is the structural control that makes fraud require collusion rather than just opportunity.

**Group-based RBAC contains the blast radius of a compromised account.**  
When an account is compromised, the attacker inherits exactly the permissions assigned to that account — nothing more. A Sales Rep account that is compromised gives access to SalesPro and email. It does not give access to FinanceCore or ITConsole. Least privilege through group-based access control limits what any single compromised identity can reach.

---

## Recommendations

**Run access certifications quarterly, not annually.**  
Annual reviews catch problems twelve months too late. A quarterly cycle catches privilege creep within ninety days of it occurring — usually before it becomes an audit finding or a security incident.

**Design the role matrix before creating any accounts.**  
The temptation is to start building and document it afterwards. Resist it. Retrofitting an access control model onto an environment where accounts already exist is significantly more work than designing it correctly from the start. This lab proved that — the matrix took two hours to design and saved what would have been days of cleanup.

**Treat SoD violations as security incidents, not compliance findings.**  
The difference in urgency matters. A compliance finding gets scheduled for remediation in the next sprint. A security incident gets resolved the same day. SoD-001 was remediated within hours of detection. That is the right response cadence.

---

## What I Would Do Differently in Production

The role matrix was built manually in a spreadsheet. In a production environment with hundreds of job titles and dozens of systems, this approach does not scale. A mature IGA platform — SailPoint, Saviynt, or similar — handles role mining automatically, surfacing existing access patterns and suggesting role definitions based on what users actually have rather than what policy says they should have.

Manual role design is the right starting point for understanding the methodology. Automated role mining is the right tool for maintaining it at scale.

---

## Personal Observation

The privilege creep scenario was simulated deliberately in this lab — but the pattern it represents is not theoretical. It is the default outcome in any environment where role changes are processed as additions rather than swaps. Nobody intends to leave stale access in place. It happens because the removal step is easy to skip when you are under time pressure to get the new access provisioned quickly.

Building the habit of treating every role change as a full swap — remove first, add second, verify after — is one of those small disciplines that prevents a category of problems entirely.

---

## Connection to Azure

The 14 security groups built in this lab sync to Entra ID in Lab 05 via Azure AD Connect. In Entra ID, these groups become the targets for Conditional Access policy assignment. Members of GRP-IT-Admins will be required to activate PIM before any admin action. Members of GRP-Finance-Users will be subject to MFA requirements for FinanceCore access. The role matrix designed here drives cloud access policy from Lab 06 onwards.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*
