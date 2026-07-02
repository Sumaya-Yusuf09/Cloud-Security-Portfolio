# Project 1: Identity & Access Management (IAM) with Microsoft Entra ID

## Overview

This project simulates a small organization's identity and access management setup in **Microsoft Entra ID** (Azure AD). It covers the core building blocks of IAM in a real Microsoft 365 / Azure environment: user and group provisioning, Conditional Access policies, Privileged Identity Management (PIM), and a mock joiner-mover-leaver (JML) offboarding process.

The goal was to get hands-on with the tools that a SOC analyst, IT support specialist, or cybersecurity analyst would actually use day-to-day — not just read about them.

**Environment:** Microsoft Entra ID (tenant: Standardkatalog)
**Tools used:** Microsoft Entra admin center, Conditional Access, Privileged Identity Management (PIM), Conditional Access "What If" tool

---

## Objectives

1. Set up a small test tenant with users and groups
2. Build and enforce a Conditional Access policy requiring MFA
3. Add a second Conditional Access policy restricting sign-in by location
4. Validate policy logic using the "What If" simulation tool
5. Configure PIM for just-in-time, approval-based privileged access
6. Simulate an employee offboarding and show the deprovisioning step

---

## Step 1 — Test Users & Groups

Created 6 test users in Entra ID (Anna Andersson, Aya Marwan, Erik Samuel, Hakeem Saafo, Moa Ali, and my own account) and 2 security groups to represent typical org structure:

- **IT-admins** — used later as the PIM-eligible group for privileged role assignment
- **Finance-users** — used later in the offboarding scenario

Both groups were created as assigned-membership security groups, which is the standard type for role and policy scoping in Entra ID.

**User list before any policies were applied:**

![User list](images/User_Created-Azure.png)

**The two groups, IT-admins and Finance-users:**

![Groups created](images/Groups_Created-Azure.png)

---

## Step 2 — Conditional Access: Require MFA for All Users

Built a Conditional Access policy named **"Require MFA for all users"** that:

- Applies to **all users and groups**
- Targets **all cloud apps/resources**
- Grants access only if **multifactor authentication** is satisfied
- **Excludes my own account** (Sumaya Yusuf) as a break-glass exclusion, so I wouldn't lock myself out of the tenant while testing — a common real-world safeguard when rolling out a new policy

The policy was enabled in **Report-only** mode first during setup to avoid an accidental lockout, then switched **On** once verified.

**Policy assignment and exclusion configuration:**

![MFA policy exclusion](images/New_policy_for_all_users_except_Sumaya_Yusuf.png)

**Grant control set to "Require multifactor authentication":**

![MFA grant control](images/Adding_control_access_to_the_new_policy.png)

**Confirmed policy state: On:**

![MFA policy on](images/MFA_policy_is_on.png)

---

## Step 3 — Conditional Access: Block Sign-in from Outside Sweden/EU

Added a second policy, **"Block sign-in from outside EU"**, to restrict access based on location. Since I don't have a way to genuinely spoof my geolocation, I built and validated this policy logic using the **What If** simulation tool rather than a live sign-in attempt (see Step 4).

Both policies together represent a layered access model: MFA everywhere, plus a hard location-based block for out-of-region sign-ins.

**Both policies listed and enabled:**

![Policy list](images/Policy_List.png)

---

## Step 4 — Validating Policies with the "What If" Tool

Used Entra's **Conditional Access "What If"** tool to simulate sign-in scenarios and confirm the policies actually trigger as expected, without needing a real sign-in attempt from another country.

**Test 1 — Simulated sign-in from Brazil:**
Input: IP `190.25.30.45`, Country: Brazil. Result: both policies evaluated as **applying** — "Require MFA for all users" and "Block sign-in from outside EU" — confirming the location-based block would correctly catch an out-of-region sign-in.

![What-if Brazil test](images/Testing__1__-_What_if__Policy_.png)

**Test 2 — Simulated sign-in for a specific user/app:**
Input: user Hakeem Saafo, device platform Windows, client app "Mobile apps and desktop clients," target app Azure AD Notification. Used to confirm policy scope and targeting logic against a real user/app combination.

![What-if user test](images/Testing_-_What_if__policy_.png)

---

## Step 5 — PIM for the IT-Admins Group

Configured **Privileged Identity Management (PIM)** to enforce just-in-time, approval-gated access to the **Security Reader** role for the IT-Admins group, rather than granting the role permanently.

Role settings configured:
- **Activation maximum duration:** 8 hours
- **Require justification on activation:** enabled
- **Require approval to activate:** enabled, with **IT Admins** set as the approver group
- Assignment type: **Eligible** (not permanently active) — role must be manually activated when needed

**Role setting: 8-hour max duration, justification + approval required:**

![PIM role settings](images/Setting_Assignment_duration___Requiring_approval_upon_acitivations.png)

**Eligible assignments for Security Reader:**

![Security Reader eligible assignments](images/Assigned_Security_Reader_to_IT-admins.png)

![Security Reader eligible assignments 2](images/Assigned_Security_Reader_to_IT_Admins.png)

**Workflow demonstrated end-to-end:**

1. Assigned the **Security Reader** role as an *eligible* assignment to the IT-Admins group
2. A user (Sara Farah) requested activation, providing a justification: *"Investigating Conditional Access and sign-in logs following suspicious sign-in report, need read access to review configuration and audit logs"*

![Activation request](images/Assignment_Activation_Request.png)

3. The request appeared in the **Approve requests** queue and was approved

![Approved request](images/Approve_Assignment_Request.png)

4. Once approved, the role became **active** for a time-boxed window (start/end time logged), after which it automatically expires and access is revoked

![Final assignment list](images/Assignment_List.png)

This reflects a least-privilege model: nobody holds standing admin/reader access — it's requested, justified, approved, and time-limited.

---

## Step 6 — Mock JML: Offboarding "Moa Ali"

Simulated an employee leaving the organization and walked through the deprovisioning steps a real offboarding process would require.

**Scenario:** Moa Ali, a member of the Finance-Users group, leaves the company.

**Before state:** Moa Ali is an active member of the **Finance-users** group (3 members total: Aya Marwan, Hakeem Saafo, Moa Ali)

![Finance users before](images/Finance_User_List.png)

**Step A — Disabled the account** in Entra ID. This immediately blocks sign-in and ends any active sessions across Microsoft services, while preserving the account and its data (no deletion, so mail/data can still be reviewed or handed over during the transition).

![Disabling sign-in](images/Blocking_Moa_Ali_from_signing_in_-_Offboarding.png)

![Account disabled status](images/Account_Disabled_-_Moa_Ali.png)

**Step B — Removed Moa Ali from the Finance-Users group**, revoking the group-based access that came with that role.

**After state:** Finance-users group now shows only 2 members (Aya Marwan, Hakeem Saafo) — Moa Ali's access has been fully deprovisioned.

![Finance users after](images/Moa_Ali_removed_from_Finance_Users_Group.png)

This mirrors a standard leaver checklist: disable sign-in first (immediate containment), then clean up group/role memberships (formal deprovisioning), while keeping the account itself intact for a defined retention period rather than deleting it outright.

---

## Key Takeaways

- **Conditional Access** is where identity and risk-based policy actually meet — MFA enforcement and location-based restrictions are two of the most common controls an org will have in place, and the "What If" tool is genuinely useful for testing policy logic safely before it affects real users.
- **PIM** turns "who has admin rights" into "who can request admin rights, for how long, and with whose approval" — a meaningfully different (and more secure) access model than standing permissions.
- Offboarding isn't a single action — it's a sequence: contain (disable sign-in) → revoke (remove from groups/roles) → retain (don't delete immediately, in case of legal/audit needs).

## Notes

This was built in a personal Entra ID test tenant for learning purposes, not a production environment. Screenshots have some fields redacted (UPNs, object identities) for privacy.
