# Lab 01 — Active Directory: On-Premises Identity Foundation

**Phase:** 1 of 5 &nbsp;|&nbsp; **Lab:** 01 of 20 &nbsp;|&nbsp; **Completed:** April 2026  
**Environment:** VMware Workstation · Windows Server 2022 · Active Directory  
**Domain:** FictionalCorp.local · DC01 · 192.168.10.10

---

## The Problem This Lab Solves

Every hybrid identity environment starts with the same question: how do you build an on-premises identity layer that works cleanly today and extends into the cloud without needing to be rebuilt? Most environments get this wrong — not because the technology is complicated, but because the early decisions (OU structure, naming conventions, group design) are made quickly and then lived with for years.

This lab builds that foundation deliberately. The domain, OU structure, security groups, and provisioning workflow created here carry forward through all 20 labs. When Azure AD Connect is configured in Lab 05, everything here syncs directly to Microsoft Entra ID. The decisions made in week one shape the entire hybrid environment.

---

## Environment

| Setting | Value | Reasoning |
|---|---|---|
| Server | DC01 | Standard naming convention for Domain Controller 01 |
| Domain | FictionalCorp.local | Internal .local suffix — non-routable, correct for a private lab domain |
| IP Address | 192.168.10.10 (static) | Domain Controllers must have a fixed IP. DHCP breaks AD. |
| DNS | 127.0.0.1 (self-referencing) | Server becomes its own DNS authority after AD promotion |
| Network Adapter 1 | Host-only (192.168.10.x) | Keeps AD traffic on a private isolated network |
| Network Adapter 2 | NAT (internet access) | Required for Azure AD Connect download in Lab 05 |

---

## Organisational Unit Design

The structure below follows two rules: departments get their own OUs for targeted Group Policy, and account states get their own OUs so the environment is readable at a glance.

```
FictionalCorp.local
├── HR
├── Finance
├── IT
├── Sales
├── Contractors        ← All time-limited accounts, separated from permanent staff
├── Disabled_Users     ← Terminated accounts, 30-day retention before deletion
└── Service_Accounts   ← Non-human accounts, separate GPO baseline
```

The `Disabled_Users` OU is excluded from Azure AD Connect sync scope in Lab 05. Terminated accounts never appear in Entra ID — active in AD means active in the cloud, disabled in AD means absent from the cloud. Clean separation makes access reviews straightforward and eliminates confusion during audits.

The `Contractors` OU exists because group-level separation alone is not enough. A dedicated OU allows a stricter Group Policy baseline — shorter lockout thresholds, no local admin, restricted logon hours — applied to contractors specifically without affecting permanent staff. It also makes contractor accounts immediately visible in any access review without needing to open individual records.

---

## Security Groups

Permissions are assigned to groups. Users are assigned to groups. Direct user-to-resource permissions are not used anywhere in this environment.

| Group | OU | Controls |
|---|---|---|
| GRP-HR-Users | HR | Standard access for HR staff |
| GRP-Finance-Users | Finance | Standard access for Finance staff |
| GRP-Finance-InvoiceCreate | Finance | Invoice creation — SoD control |
| GRP-Finance-InvoiceApprove | Finance | Invoice approval — SoD control |
| GRP-IT-Users | IT | Standard access for IT staff |
| GRP-Sales-Users | Sales | Standard access for Sales staff |

The two Finance invoice groups were created at this stage, not in Lab 02 where the SoD violation is demonstrated. Designing the structure correctly from the start is always easier than retrofitting it after accounts are already in production.

---

## Provisioning

Both accounts were processed from mock tickets. In production, an account with no ticket behind it has no audit trail — which becomes a problem the first time someone asks why that account exists.

---

### INC-0001 — Marcus Williams — Sales Representative

```
Requested by:  Sarah Johnson — HR Manager
New hire:      Marcus Williams  |  Sales Representative  |  Permanent
Username:      m.williams
Access:        Standard Sales access
```

Placed in the `Sales` OU. Assigned to `GRP-Sales-Users` only. No Finance, no HR, no IT. Password set to force a change at first logon — the provisioning password belongs to the analyst, not the user.

---

### INC-0002 — Jessica Park — IT Support Contractor

```
Requested by:  Tom Richards — IT Manager
Contractor:    Jessica Park  |  IT Support Contractor
Username:      j.park.contractor
Contract end:  90 days from provisioning
Access:        IT support only  |  No admin rights
```

Placed in the `Contractors` OU. Assigned to `GRP-IT-Users`. Account expiry set at provisioning — not flagged for follow-up, not relying on anyone's memory. When the date arrives, AD disables the account automatically.

The `.contractor` suffix in the username carries information without requiring anyone to open the account. This is a small detail that makes a real difference at scale.

---

## Threat Connections

Every control in this lab exists because of a specific risk. Here is what each one is actually defending against:

**Contractor account expiry →** Eliminates the most common source of lingering access. Contractor accounts that stay active after a contract ends — because an offboarding ticket was never raised, or was raised and not actioned — are one of the most frequent findings in access audits. Automation removes the human dependency entirely.

**Disabled_Users OU excluded from cloud sync →** Prevents terminated employee accounts from appearing in Entra ID after offboarding. Without this exclusion, a disabled on-premises account would still show as a cloud identity — visible to applications, potentially accessible through token-based authentication until the sync propagates. Exclusion at the OU level is cleaner and more reliable than relying on the disable state to sync quickly enough.

**Group-based RBAC →** Eliminates permission sprawl. When a user changes roles, one group membership change adjusts all associated permissions across every system the group has access to. Direct user permissions require individual auditing of every access point — which does not scale and creates the conditions for privilege creep.

**Naming conventions that carry information →** Reduces the cognitive overhead of managing the environment and makes misconfigurations visible faster. `j.park.contractor` and `GRP-Finance-InvoiceApprove` tell you what you need to know before you open anything. In an environment with hundreds of accounts and dozens of groups, this matters.

---

## Recommendations

**Set contractor account expiries at provisioning, not at offboarding.**  
Offboarding is a reactive process that depends on someone remembering to raise a ticket. Expiry dates are proactive and automatic. Every contractor account should have an expiry date set the moment it is created, matching the contract end date exactly.

**Keep the Disabled_Users OU out of your Azure AD Connect sync scope.**  
This is a design decision that needs to be made before Azure AD Connect is configured, not after. Retrofitting it requires a sync reconfiguration and a cleanup of any terminated accounts that have already appeared in Entra ID.

**Separate contractors from permanent staff at the OU level.**  
Group separation handles access control. OU separation handles policy, visibility, and cloud sync scope. You need both.

**Use naming conventions that carry information.**  
A naming convention is documentation that maintains itself. Every time someone searches AD, runs a script, or reads an access review, the naming convention either helps them or makes them do extra work. Design it to help.

---

## What I Would Do Differently in Production

The lab domain is `FictionalCorp.local` — an internal non-routable suffix. In a production environment I would add a routable UPN suffix (such as `fictionalcorp.com`) as an alternative UPN in AD before configuring Azure AD Connect. This gives cloud users a cleaner UPN (`m.williams@fictionalcorp.com` instead of `m.williams@fictionalcorp.onmicrosoft.com`) and avoids the UPN suffix mismatch warning that appears during Azure AD Connect configuration.

It is a small thing to set up before the first account is created. It is a meaningful amount of work to change after hundreds of accounts exist.

---

## Personal Observation

The thing that became clear doing this lab is how much the early decisions constrain everything that comes after. The OU structure, the naming conventions, the decision to separate contractors — none of these feel significant in week one. By Lab 05 when Azure AD Connect is running and users are syncing to Entra ID, every one of those decisions is visible in the cloud. Getting them right in Lab 01 meant Lab 05 was straightforward. Getting them wrong would have meant rework before the sync could even be configured cleanly.

Identity infrastructure is the kind of thing where the cost of a bad early decision compounds over time. That is worth understanding before you touch a production environment.

---

## Connection to Azure

In Lab 05, Azure AD Connect is configured on DC01 to sync `FictionalCorp.local` to the Entra ID tenant at `fictionalcorp.onmicrosoft.com`. The OU structure and sync scope decisions made in this lab directly determine which users and groups appear in the cloud. Labs 06 through 20 build on that hybrid foundation.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*
