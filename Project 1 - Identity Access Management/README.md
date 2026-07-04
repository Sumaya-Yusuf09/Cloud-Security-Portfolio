# Project 1: Identity and Access Management in Microsoft Entra ID

## The problem this solves

A lot of companies still let anyone sign in from anywhere with just a password, and when someone leaves the company nobody remembers to actually remove their access in time. That is how accounts get compromised and how ex employees end up keeping access to systems they should not touch anymore. This project is my attempt at fixing both of those problems using Microsoft Entra ID (the identity part of Azure). I built a small test environment with real users and groups, forced everyone to use MFA, blocked sign ins from risky locations, set up time limited admin access that needs approval, and simulated what happens when someone leaves the company and has to be offboarded properly.

I am not claiming this is a full production setup for a real company. It is a lab environment built to show that I understand how identity security actually works in Azure, not just that I read about it.

## What I actually built

**Test users and groups**
I created several test users and two security groups, IT Admins and Finance Users, so I would have a realistic setup to apply policies to instead of just testing on my own account.

![Users created in Entra ID](screenshots/Step_1__User_Created-Azure.png)
![Groups created in Entra ID](screenshots/Step_2__Groups_Created-Azure.png)

**Forcing MFA for everyone**
I built a Conditional Access policy that requires multi factor authentication for all users. I excluded my own account from the policy while building it so I would not accidentally lock myself out while testing.

![New policy excluding my own account while testing](screenshots/Step_3_1_New_policy_for_all_users_except_Sumaya_Yusuf.png)
![Grant control set to require MFA](screenshots/Step_3_2_Adding_control_access_to_the_new_policy.png)
![MFA policy turned on and active](screenshots/Step_4__Check_MFA_policy_is_on.png)

**Blocking sign ins from outside the EU**
On top of MFA, I added a second policy that blocks sign in attempts coming from outside Sweden and the EU. This is meant to stop the most common kind of account takeover, where someone gets a stolen password and tries to log in from a completely different country.

![Both policies now active](screenshots/Step_5__Policy_List_after_adding_another_policy.png)

**Testing the policies actually work**
Conditional Access policies are useless if you do not check they behave the way you expect. Entra ID has a What If tool that lets you simulate a sign in without actually doing one, so I used it to simulate a login from Brazil and confirmed both policies would correctly kick in and block it.

![Setting up the What If simulation](screenshots/Step_6_Testing_-_What_if__policy_.png)
![What If result showing both policies would apply](screenshots/Step_6_Testing__continued_screenshot__-_What_if__Policy_.png)

**Admin access that expires and needs approval**
Standing admin access is one of the biggest risks in any company, because if that account ever gets compromised the attacker basically has the keys to everything, all the time. Instead I used Privileged Identity Management (PIM) to give the IT Admins group eligible access to the Security Reader role, meaning nobody has the role by default. Someone has to actively request it, give a reason, get approved, and the access automatically expires after a few hours.

![Security Reader role assigned to IT Admins as eligible](screenshots/Step_7__Assigned_Security_Reader_to_IT_Admins.png)
![Checking the assignment shows up correctly](screenshots/Step_8__Check_Assignment_List.png)
![Confirming the eligible assignment for IT Admins](screenshots/Step_8___Continued_screenhost__Assigned_Security_Reader_to_IT-Admins.png)
![Role settings requiring approval and justification](screenshots/Step_9__Setting_Assignment_duration___Requiring_approval_upon_acitivations.png)

To prove this actually works end to end, I had one of the test users, Sarah, request activation of the role with a reason attached, then approved that request myself as an admin.

![Sarah requesting activation with a written reason](screenshots/Step_10__Assignment_Activation_Request__as_the_user_Sarah_.png)
![Approving Sarah's request](screenshots/Step_11__Approve_Assignment_Request.png)

**Offboarding someone the right way**
The last part simulates something every company deals with constantly, an employee leaving. I picked one of the Finance Users, Moa Ali, and walked through the full offboarding process the way it should actually happen. First I checked the group to confirm Moa was a member.

![Finance Users group before offboarding](screenshots/Step_12__Check_Finance_User_List.png)

Then I removed Moa from the Finance Users group so all that access tied to the group is gone immediately.

![Moa removed from the Finance Users group](screenshots/Step_13__Offboarding__Remove_Moa_Ali_from_Finance_Users_Group.png)

Removing someone from a group is not enough on its own, so I also disabled the account directly, which blocks all sign in attempts and kills any active sessions across every Microsoft service right away.

![Disabling Moa's account to block sign in](screenshots/Step_14__Blocking_Moa_Ali_from_signing_in__Denied_Access.png)

Finally I checked the account to confirm it actually shows as disabled.

![Confirming the account status is disabled](screenshots/Step_15__Check_If_Moa_Ali_Account_Disabled.png)

## Why I made these choices

I used Conditional Access instead of just turning on basic security defaults because Conditional Access lets you build layered, specific rules like combining MFA with location, and that is closer to how a real company would actually configure this.

I chose PIM with required approval and time limits instead of just giving IT Admins the role permanently, because permanent admin access sitting unused is one of the easiest things for an attacker to exploit if any single admin account gets compromised. Making it eligible and time boxed massively shrinks that window of risk.

For offboarding I did both the group removal and the account disable, not just one of them, because relying on only one of those steps is exactly how companies end up with ex employees who still technically have access somewhere.

## How to try this yourself

You will need an Azure subscription with Entra ID and at least an Entra ID P2 license, since PIM requires P2. Create your own test users and two groups the same way I did, then build the two Conditional Access policies under Security, Conditional Access. Test them with the What If tool before trusting them. Set up PIM under Privileged Identity Management by making a role eligible for a group instead of assigning it directly, and turn on approval and justification requirements in the role settings. For the offboarding part, just remove a test user from a group and disable their account, then double check both changes actually took effect.

## What is next

Project 2 is a serverless security alerting pipeline using Defender for Cloud, Logic Apps, and Key Vault. Project 3 will rebuild this same setup using Terraform, and Project 4 will wrap it into a CI/CD pipeline.

