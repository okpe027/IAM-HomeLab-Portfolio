# Lab 06 — Conditional Access: The Point Where Identity Becomes Security

**Phase:** 2 of 5 &nbsp;|&nbsp; **Lab:** 06 of 20 &nbsp;|&nbsp; **Completed:** April 2026  
**Environment:** Microsoft Entra ID P2 · fictionalcorp.onmicrosoft.com  
**Builds on:** Lab 05 — FictionalCorp users synced from on-premises AD to Entra ID

---

## Before This Lab, the Door Was Open

After Lab 05, FictionalCorp had a fully functioning hybrid identity environment. Thirteen users synced from on-premises Active Directory to Microsoft Entra ID. Azure AD Connect running. Password hashes synced. Everything confirmed working.

And yet — anyone with a valid username and password could walk straight in. From anywhere. On any device. Through any application. Including applications that were built before multi-factor authentication existed and have no ability to support it.

That is not a hybrid identity environment. That is a hybrid identity environment with the front door unlocked.

Conditional Access is the lock.

---

## What Conditional Access Actually Does

Most people describe Conditional Access as a security feature. That is true but it undersells it. Conditional Access is a decision engine. Every single sign-in attempt that hits the FictionalCorp tenant gets evaluated against a set of questions before anything happens:

- Who is this?
- What are they trying to access?
- Where are they signing in from?
- What device are they on?
- What does their sign-in risk score look like?

Based on the answers, one of three things happens: access is granted, access is blocked, or access is granted only after the user satisfies an additional requirement. The requirement might be MFA. It might be a compliant device. It might be accepting terms of use.

This is Zero Trust in practice. Not as a marketing concept — as an actual technical implementation. Every request evaluated on its own merits. Nothing assumed safe because of where it came from.

---

## Setting the Stage — What Had to Happen First

Before building a single policy, two things needed to be addressed.

**Security Defaults had to go.**

When a new Entra ID tenant is created, Microsoft enables something called Security Defaults — a fixed set of baseline security controls applied automatically. It requires MFA for admin accounts, blocks legacy authentication, and a few other things. It is a good starting point for organisations that have not yet built their own security posture.

The problem is that Security Defaults and Conditional Access cannot coexist. They are two different philosophies about the same problem. Security Defaults says: apply these fixed controls everywhere. Conditional Access says: let us define exactly what we want, for exactly who, under exactly which circumstances. Choosing Conditional Access means taking full ownership of the policy stack. Security Defaults was disabled with the reason "My organization is using Conditional Access" — which is exactly true.

**A break-glass account had to be created.**

This is something that does not get talked about enough in most IAM training, and it is one of those things you only truly appreciate after you have nearly locked yourself out of a tenant.

A break-glass account is an emergency administrator account that is deliberately excluded from every single Conditional Access policy. The name comes from the physical security concept — break glass in an emergency. It exists for one scenario: a misconfigured CA policy locks out all other admin accounts and you need a way back in.

```
Account:   breakglass@fictionalcorp.onmicrosoft.com
Purpose:   Emergency access only
Used for:  Nothing in normal operations. Ever.
Excluded:  Every Conditional Access policy in the tenant
```

In a production environment this account's credentials are stored offline — sometimes literally printed and locked in a physical safe. Any sign-in from it triggers an immediate security alert because it should never be used unless something has gone seriously wrong. In the FictionalCorp lab it lives in the GRP-CA-Excluded group alongside the test accounts.

Creating it before building any policies is the right order. Creating it after you have already locked yourself out is too late.

---

## Policy 1 — Block Legacy Authentication

**CA-001 · State: On · Created: April 28, 2026**

---

This was the first policy built and the most impactful one in terms of pure attack surface reduction.

Legacy authentication protocols — Exchange ActiveSync, SMTP, POP3, IMAP, older Office clients — were designed in an era before multi-factor authentication existed. They have no ability to support it. That means any MFA policy applied to a user account is completely irrelevant if an attacker can authenticate through a legacy protocol. They just bypass MFA entirely by using a different door.

Microsoft has published data showing that more than 99% of password spray attacks use legacy authentication. That number is worth sitting with for a moment. Not some attacks. Not most attacks. Essentially all of them. The reason is obvious once you understand it — why would an attacker attempt to bypass MFA when they can use a protocol that was built before MFA existed?

Blocking legacy authentication does not make it harder to attack FictionalCorp through these protocols. It makes it impossible.

**How the policy was configured:**

The policy applies to all users. The GRP-CA-Excluded group is exempt — this ensures test accounts and the break-glass account are never blocked. The target is all cloud apps. The condition is specific: Exchange ActiveSync clients and Other clients are checked. Browser and mobile app authentication are left alone — those support modern authentication and are not the problem being solved here.

The grant control is Block access. No conditions. No exceptions beyond the excluded group. If the authentication attempt uses a legacy protocol, it does not get in.

---

## Policy 2 — Require MFA for Administrators

**CA-002 · State: On · Created: April 29, 2026**

---

Admin accounts are the most valuable target in any identity environment. Not because admins are careless — most are not. But because the asymmetry of impact is enormous.

A standard user account compromised through a phishing attack gives an attacker access to one person's email, files, and applications. An admin account compromised the same way gives them the ability to create new accounts, assign roles to those accounts, disable security controls, access every piece of data in the environment, and cover their tracks. The credential theft method is identical. The consequences are incomparably different.

Requiring MFA for administrators means a stolen password is not enough. The attacker needs the password and physical access to the admin's second factor simultaneously. That combination is significantly harder to achieve than a phishing email.

**The roles covered:**

Rather than applying this policy to individual accounts, it targets directory roles directly. Any account assigned one of these roles is automatically covered regardless of when the account was created or who created it:

- Global Administrator
- Security Administrator
- Exchange Administrator
- SharePoint Administrator
- Privileged Role Administrator
- Conditional Access Administrator

The break-glass account is explicitly excluded. An emergency access account that itself requires MFA is not a reliable emergency access account — if MFA is broken or the authenticator device is unavailable, the break-glass account needs to work unconditionally.

**Testing it:**

A test admin account was created, MFA was registered on it, and a sign-in was attempted. The MFA prompt appeared before access was granted. The policy is working as intended.

---

## Policy 3 — Block Risky Locations

**CA-003 · State: Report-only · Created: April 29, 2026**

---

FictionalCorp is a Canadian company. Its users work in Canada. Its systems are accessed from Canada. If a sign-in attempt arrives from a country that FictionalCorp has no presence in and no business relationship with, that is worth paying attention to.

This policy uses a Named Location — a defined list of trusted countries — and blocks any authentication attempt that originates outside it. Canada and the United States are trusted. Everything else is not.

**Why Report-only and not On?**

This is a judgment call that matters.

Enforcing a location-based block without first understanding your organisation's actual sign-in geography is how you create a service outage. A user travelling internationally for work. A team member working remotely from a different country. A vendor accessing a shared system from overseas. All of these are legitimate sign-in scenarios that a location block would terminate without warning.

Report-only mode evaluates every sign-in against the policy and logs what would have happened — without actually blocking anyone. Two weeks of that data reveals whether any legitimate business activity originates from outside the trusted locations. If it does, the policy needs to be adjusted before it goes live. If it does not, turning it on becomes a straightforward decision with no risk of disruption.

Leaving CA-003 in Report-only is not a delay. It is the responsible way to deploy a policy that will affect every user in the tenant.

---

## What the Tenant Looks Like Now

Three policies. Two enforced. One monitoring.

| Policy | State | What changed |
|---|---|---|
| CA-001 — Block Legacy Authentication | On | Legacy protocol attacks are now impossible |
| CA-002 — Require MFA for Administrators | On | Admin accounts now require a second factor |
| CA-003 — Block Risky Locations | Report-only | Monitoring sign-in geography before enforcing |

Every FictionalCorp user who synced from on-premises AD in Lab 05 is now subject to these policies. The hybrid identity environment built across Labs 01 through 05 now has a security layer on top of it.

---

## Two Things That Did Not Go Smoothly

**The Security Defaults error.**

The first attempt to switch CA-001 from Report-only to On produced an error: "Security defaults must be disabled to enable Conditional Access policy." This was expected — disabling Security Defaults is a prerequisite that needs to happen before any CA policy can be enforced. It was disabled in Entra ID Properties before proceeding.

**The Insights and Reporting dashboard.**

The Conditional Access Insights and Reporting dashboard returned an access error. It requires a Log Analytics workspace connected to the tenant to populate — without one the dashboard has no data source to pull from. This is not a blocker. Sign-in logs are fully accessible through Entra ID Monitoring and show the same underlying data. Log Analytics will be configured when Microsoft Sentinel is deployed in Lab 15.

---

## Three Things Worth Carrying Into Any Production Environment

**Test in Report-only before enforcing anything.**  
A CA policy that blocks legitimate users is indistinguishable from a service outage from the user's perspective. Report-only mode costs nothing and prevents that scenario entirely. Make it standard practice for every policy deployment regardless of how confident you are in the configuration.

**Create the break-glass account before the first policy, not after.**  
The value of a break-glass account is entirely dependent on it existing before something goes wrong. Creating it afterwards — or worse, deciding not to bother — is the kind of decision that gets reversed at the worst possible moment.

**Build policies in layers and confirm each one before starting the next.**  
CA-001 was confirmed working before CA-002 was started. CA-002 was confirmed before CA-003. This approach makes troubleshooting straightforward — if something breaks after a new policy is added, you know exactly which policy caused it. Building all three simultaneously and then trying to isolate a problem is significantly harder.

---

## What Surprised Me

I expected the hardest part of this lab to be the technical configuration. The portal is straightforward once you understand the structure — users, cloud apps, conditions, grant. That part is mechanical.

What actually required more thought was the sequencing decisions. Disabling Security Defaults before creating policies. Creating the break-glass account before building anything that could cause a lockout. Leaving CA-003 in Report-only rather than enforcing it immediately. None of those are things the interface prompts you to do. They are judgment calls that come from understanding why the controls exist, not just how to configure them.

That distinction — between knowing how to configure something and understanding why it is configured that way — is what this lab was really about.

---

## What Comes Next

**Lab 07 — Hybrid Offboarding with Cloud Session Revocation**  
The 30-minute hybrid sync gap documented in Lab 05 means a disabled on-premises account can still have active Entra ID sessions for up to 30 minutes after offboarding. Lab 07 closes that gap by adding immediate cloud session revocation to the offboarding procedure — completing the hybrid offboarding workflow that Lab 03 started.

---

*Opeyemi Ogundimu — IAM Home Lab Portfolio*  
*github.com/okpe027/My_First_Project*
