# Lab 05 — Azure AD Connect: Building the Hybrid Identity Bridge

**Phase:** 2 of 5 &nbsp;|&nbsp; **Lab:** 05 of 20 &nbsp;|&nbsp; **Completed:** April 2026  
**Environment:** VMware Workstation · Windows Server 2022 · Microsoft Entra Connect · Azure Entra ID  
**Domain:** FictionalCorp.local → fictionalcorp.onmicrosoft.com  
**Sync Method:** Password Hash Synchronization · Password Writeback enabled

---

## The Problem This Lab Solves

Labs 01 through 04 built a solid on-premises identity foundation — a properly structured domain, a role-based access control model, offboarding procedures, and audit log investigation. All of it existed in isolation. Users were in Active Directory. The cloud was empty.

That gap is the problem this lab closes.

Most organisations do not run purely on-premises or purely in the cloud. They run both — Active Directory managing the core identity layer, Microsoft 365 and cloud platforms sitting on top. Azure AD Connect is the bridge between those two worlds. Once it is configured, a user created in AD exists in Entra ID. A password changed on-premises propagates to the cloud. A group membership updated in AD reflects in cloud application assignments. The two environments stop being separate things and become one coherent identity system.

Every lab from here onwards depends on that bridge being in place.

---

## What Was Configured

### Sync Method: Password Hash Synchronization

Password hashes are synced from on-premises AD to Entra ID. This means FictionalCorp users can authenticate to cloud services using the same credentials they use on-premises. It also enables leaked credential detection in Entra ID Identity Protection — Microsoft compares synced password hashes against known breach databases and flags matches as high-risk sign-ins.

Pass-through Authentication was considered but rejected for this environment. It requires the on-premises AD to be reachable for every cloud authentication request — which creates a dependency that does not make sense in a lab environment and adds complexity without meaningful benefit at this scale.

### OU Filtering

Not everything in AD should appear in Entra ID. The sync scope was configured explicitly:

| OU | Synced | Reason |
|---|---|---|
| HR | ✅ Yes | Active staff |
| Finance | ✅ Yes | Active staff |
| IT | ✅ Yes | Active staff |
| Sales | ✅ Yes | Active staff |
| Contractors | ✅ Yes | Active contractors with expiry dates |
| Service_Accounts | ✅ Yes | Service accounts need cloud visibility |
| **Disabled_Users** | ❌ No | Terminated accounts must not appear in Entra ID |
| Domain Controllers | ❌ No | Computer objects — not relevant |
| Computers | ❌ No | Computer objects — not relevant |

The Disabled_Users exclusion is the most important filtering decision. When someone is offboarded in FictionalCorp, their account is disabled and moved to that OU. Excluding it from sync means terminated accounts disappear from Entra ID automatically as the sync propagates — no manual cloud cleanup required.

### Password Writeback

Enabled. This allows password resets initiated through Entra ID Self-Service Password Reset (configured in Lab 08) to write back to the on-premises AD account. Without it, a user who resets their cloud password finds their on-premises password unchanged — which breaks authentication for any on-premises resources they need to access.

### UPN Suffix

FictionalCorp.local is a non-routable internal domain suffix that cannot be verified in Entra ID. The installer flagged this as a warning — expected behaviour for any organisation using a .local domain. The configuration continues without matching UPN suffixes to verified domains. Synced users authenticate to Entra ID using the fictionalcorp.onmicrosoft.com tenant domain rather than the .local suffix.

---

## Obstacles Encountered

Three issues came up during installation. Each one is worth documenting because they are common in real enterprise deployments.

**TLS 1.2 not configured.**  
The installer blocked immediately with an "Incorrect version of TLS" error. Windows Server 2022 does not enable TLS 1.2 for .NET Framework by default — it requires explicit registry configuration. Fixed by setting `SchUseStrongCrypto` in both the 32-bit and 64-bit .NET Framework registry paths and enabling TLS 1.2 in the SCHANNEL protocols. Server restart required after the fix.

**Internet Explorer Enhanced Security Configuration blocking the login page.**  
Azure AD Connect uses an embedded browser for the Azure authentication step. On Windows Server, IE Enhanced Security Configuration is enabled by default and blocked the Microsoft login page. Fixed by disabling IE Enhanced Security Configuration in Server Manager under Local Server settings.

**JavaScript blocked in Internet Explorer.**  
After disabling the enhanced security configuration, the Microsoft login page loaded but threw a JavaScript error. Fixed by enabling Active Scripting in Internet Explorer's security settings for both the Internet and Trusted Sites zones. In a production environment this would be handled by deploying Edge as the default browser on the sync server — IE should not be the embedded authentication browser for any modern deployment.

---

## Verification

After installation completed and the initial sync cycle ran, 13 FictionalCorp users appeared in the Entra ID tenant at fictionalcorp.onmicrosoft.com:

```
Alex Chen          · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
David Osei         · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Fatimah Al-Hassan  · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
James Adeyemi      · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Jessica Park       · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Karen Nwosu        · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Linda Okafor       · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Marcus Williams    · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Michael Obi        · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Priya Sharma       · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
Rachel Thompson    · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
svc.financecore    · On-premises sync: Yes · FictionalCorp.onmicrosoft.com
```

Every account shows **On-premises sync: Yes** — confirming the hybrid connection is live and working.

Live sync was verified by changing Marcus Williams' department attribute in on-premises ADUC, forcing a delta sync, and confirming the change appeared in Entra ID within two minutes:

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

---

## The Hybrid Sync Gap

One behaviour worth understanding before it causes a problem in production: Azure AD Connect syncs on a 30-minute cycle by default. That means when an account is disabled in on-premises AD, the disable state takes up to 30 minutes to propagate to Entra ID.

During that window, the terminated employee's cloud sessions remain active. They can still access email, SharePoint, Teams, and any cloud application assigned in Entra ID — even though their AD account is already disabled.

For a standard resignation this is an acceptable window. For a for-cause termination it is not.

The fix is documented in the updated offboarding procedure from Lab 03: immediately after disabling in AD, manually revoke all Entra ID sessions through the portal. Do not wait for the sync cycle. The session revocation is immediate. The sync propagating the disable state is background.

```
Step 1: Disable in AD                    → immediate (on-prem access cut)
Step 2: Revoke Entra ID sessions         → immediate (cloud access cut)
Step 3: Remove Azure licenses            → immediate
Step 4: AD Connect sync propagates       → background (up to 30 mins)
```

---

## Threat Connections

**Password Hash Synchronization enables cloud-based credential threat detection.**  
Syncing password hashes to Entra ID is not just a convenience feature. It enables Microsoft's leaked credential detection — a service that compares synced hashes against breach databases from the dark web. When a match is found, the account is flagged as high risk in Identity Protection. This is a threat detection capability that a purely on-premises environment cannot replicate. The hash that goes to Microsoft is a derived value, not the actual password — the original credential never leaves the organisation.

**OU filtering as an offboarding control.**  
Excluding Disabled_Users from sync scope is a security control, not just an organisational preference. Without it, a disabled on-premises account would remain visible in Entra ID until the sync propagated the disabled state — and during that window, token-based cloud authentication could still succeed depending on session state. Exclusion at the OU level is a harder boundary than relying on sync timing.

**The sync service account principle.**  
Azure AD Connect creates a dedicated service account in AD to handle synchronisation. That account has specific directory read permissions — it does not need Domain Admin rights for normal operation. In a production deployment, the principle of least privilege applies here too: the sync account should have only the permissions it needs for sync, nothing more. Over-privileged sync accounts are a real attack vector — if the sync server is compromised, the sync account becomes the path into AD.

---

## Recommendations

**Set Edge as the default browser on any Azure AD Connect server before installation.**  
The embedded browser that Azure AD Connect uses for authentication falls back to Internet Explorer when Edge is not the default. IE on Windows Server is heavily restricted by default — Enhanced Security Configuration, blocked JavaScript, blocked modern authentication flows. Every obstacle encountered during this installation traces back to that one missing prerequisite. Set Edge as default first. Save yourself an hour.

**Configure the sync cycle interval deliberately.**  
The default 30-minute sync cycle is appropriate for most environments. But for environments with high offboarding frequency — contractors, seasonal workers, project-based staff — a shorter interval reduces the window between an on-premises disable and cloud propagation. The interval can be set with:
```powershell
Set-ADSyncScheduler -CustomizedSyncCycleInterval 00:15:00
```
Reduce it only if the operational need justifies the additional load on the sync server.

**Monitor the sync service, not just the sync results.**  
Most teams check whether users appeared in Entra ID and consider the job done. The more important thing to monitor is the health of the sync service itself — whether it is running, when it last completed successfully, and whether any errors accumulated over time. A sync service that silently fails means on-premises changes stop propagating to the cloud without anyone noticing until something breaks.

```powershell
Get-ADSyncConnectorRunStatus
Get-ADSyncScheduler | Select-Object LastSyncTime, NextSyncCycleStartTimeInUTC, SyncCycleEnabled
```

These two commands should be part of any routine IAM health check.

---

## What I Would Do Differently in Production

The lab domain uses the `.local` suffix — a common choice in legacy environments but one that creates ongoing friction in any hybrid deployment. A routable UPN suffix (like `fictionalcorp.com`) should be added as an alternative UPN in AD before the first Azure AD Connect configuration. Users then sync with clean, routable UPNs that match a verified Entra ID domain, enabling full seamless SSO without workarounds.

Adding this after Azure AD Connect is configured and hundreds of accounts exist requires a UPN migration — a disruptive operation that often gets deferred indefinitely. The right time to do it is before the sync is configured. In this lab that window has passed, which is a useful lesson about the cost of early decisions made for convenience.

---

## Personal Observation

The obstacles in this lab — TLS, IE Enhanced Security, JavaScript — are not interesting problems. They are friction. But they are exactly the kind of friction that shows up in production deployments and consumes hours of troubleshooting time when engineers are not expecting it. Documenting each one, understanding why it occurred, and knowing how to prevent it next time is the actual value of working through it manually rather than scripting around it.

The moment the sync completed and 13 FictionalCorp users appeared in Entra ID with **On-premises sync: Yes** across the board — that is when the two environments became one. Everything built in Labs 01 through 04 is now a hybrid identity system. The remaining labs build on that.

---

## What Comes Next

**Lab 06 — Conditional Access Policies**  
With FictionalCorp users now present in Entra ID, Conditional Access policies can be applied to them. Lab 06 builds three policies that mirror real enterprise security requirements: blocking legacy authentication, enforcing MFA for privileged roles, and restricting access by location. These policies apply to the same synced users set up in this lab.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*
