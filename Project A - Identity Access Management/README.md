# Project 1: Identity and Access Management in Microsoft Entra ID

## The problem this solves

A lot of companies still let anyone sign in from anywhere with just a password, and when someone leaves the company nobody remembers to actually remove their access in time. That is how accounts get compromised and how ex employees end up keeping access to systems they should not touch anymore. This project is my attempt at fixing both of those problems using Microsoft Entra ID (the identity part of Azure). I built a small test environment with real users and groups, forced everyone to use MFA, blocked sign ins from risky locations, set up time limited admin access that needs approval, and simulated what happens when someone leaves the company and has to be offboarded properly.

I am not claiming this is a full production setup for a real company. It is a lab environment built to show that I understand how identity security actually works in Azure, not just that I read about it.

## What I actually built

**Test users and groups**
I created several test users and two security groups, IT Admins and Finance Users, so I would have a realistic setup to apply policies to instead of just testing on my own account.

![Users created in Entra ID](Step%201.%20User_Created-Azure.png)
![Groups created in Entra ID](Step%202.%20Groups_Created-Azure.png)

**Forcing MFA for everyone**
I built a Conditional Access policy that requires multi factor authentication for all users. I excluded my own account from the policy while building it so I would not accidentally lock myself out while testing.

![New policy excluding my own account while testing](Step%203.1%20New%20policy%20for%20all%20users%20except%20Sumaya%20Yusuf.png)
![Grant control set to require MFA](Step%203.2%20Adding%20control%20access%20to%20the%20new%20policy.png)
![MFA policy turned on and active](Step%204.%20Check%20MFA%20policy%20is%20on.png)

**Blocking sign ins from outside the EU**
On top of MFA, I added a second policy that blocks sign in attempts coming from outside Sweden and the EU. This is meant to stop the most common kind of account takeover, where someone gets a stolen password and tries to log in from a completely different country.

![Both policies now active](Step%205.%20Policy%20List%20after%20adding%20another%20policy.png)

**Testing the policies actually work**
Conditional Access policies are useless if you do not check they behave the way you expect. Entra ID has a What If tool that lets you simulate a sign in without actually doing one, so I used it to simulate a login from Brazil and confirmed both policies would correctly kick in and block it.

![Setting up the What If simulation](Step%206%20Testing%20-%20What%20if%20%28policy%29.png)
![What If result showing both policies would apply](Step%206%20Testing%20%28continued%20screenshot%29%20-%20What%20if%20%28Policy%29.png)

**Admin access that expires and needs approval**
Standing admin access is one of the biggest risks in any company, because if that account ever gets compromised the attacker basically has the keys to everything, all the time. Instead I used Privileged Identity Management (PIM) to give the IT Admins group eligible access to the Security Reader role, meaning nobody has the role by default. Someone has to actively request it, give a reason, get approved, and the access automatically expires after a few hours.

![Security Reader role assigned to IT Admins as eligible](Step%207.%20Assigned%20Security%20Reader%20to%20IT%20Admins.png)
![Checking the assignment shows up correctly](Step%208.%20Check%20Assignment%20List.png)
![Confirming the eligible assignment for IT Admins](Step%208.%20%28Continued%20screenhost%29%20Assigned%20Security%20Reader%20to%20IT-Admins.png)
![Role settings requiring approval and justification](Step%209.%20Setting%20Assignment%20duration%20%26%20Requiring%20approval%20upon%20acitivations.png)

To prove this actually works end to end, I had one of the test users, Sarah, request activation of the role with a reason attached, then approved that request myself as an admin.

![Sarah requesting activation with a written reason](Step%2010.%20Assignment%20Activation%20Request%20%28as%20the%20user%20Sarah%29.png)
![Approving Sarah's request](Step%2011.%20Approve%20Assignment%20Request.png)

**Offboarding someone the right way**
The last part simulates something every company deals with constantly, an employee leaving. I picked one of the Finance Users, Moa Ali, and walked through the full offboarding process the way it should actually happen. First I checked the group to confirm Moa was a member.

![Finance Users group before offboarding](Step%2012.%20Check%20Finance%20User%20List.png)

Then I removed Moa from the Finance Users group so all that access tied to the group is gone immediately.

![Moa removed from the Finance Users group](Step%2013.%20Offboarding.%20Remove%20Moa%20Ali%20from%20Finance%20Users%20Group.png)

Removing someone from a group is not enough on its own, so I also disabled the account directly, which blocks all sign in attempts and kills any active sessions across every Microsoft service right away.

![Disabling Moa's account to block sign in](Step%2014.%20Blocking%20Moa%20Ali%20from%20signing%20in.%20Denied%20Access.png)

Finally I checked the account to confirm it actually shows as disabled.

![Confirming the account status is disabled](Step%2015.%20Check%20If%20Moa%20Ali%20Account%20Disabled.png)

In a full offboarding you would also pull any licenses since those cost money whether someone is using them or not, remove any PIM role assignments or app permissions Moa still had, and reset the password on top of disabling the account so that even if the account got turned back on by mistake later, the old password would be useless.

## Why I made these choices

I used Conditional Access instead of just turning on basic security defaults because Conditional Access lets you build layered, specific rules like combining MFA with location, and that is closer to how a real company would actually configure this.

I chose PIM with required approval and time limits instead of just giving IT Admins the role permanently, because permanent admin access sitting unused is one of the easiest things for an attacker to exploit if any single admin account gets compromised. Making it eligible and time boxed massively shrinks that window of risk.

For offboarding I did both the group removal and the account disable, not just one of them, because relying on only one of those steps is exactly how companies end up with ex employees who still technically have access somewhere. In hindsight the tighter order would have been to disable the account first and clean up the group after, since disabling closes the door completely and instantly, so even if a group or permission gets missed the person still cannot get in. Doing it the other way round still gets to the same end result, it just leaves a tiny window open while the cleanup happens. Then I also checked for any licenses, since those cost money whether someone is using them or not, removed any PIM role assignments or app permissions Sara still had, and reset the password on top of disabling the account, so that even if the account got switched back on by mistake later, the old password would be useless.

## How to try this yourself

You will need an Azure subscription (used the free tier) with Entra ID and at least an Entra ID P2 license (Used free trail), since PIM requires P2. Create your own test users and two groups the same way I did, then build the two Conditional Access policies under Security, Conditional Access. Test them with the What If tool before trusting them. Set up PIM under Privileged Identity Management by making a role eligible for a group instead of assigning it directly, and turn on approval and justification requirements in the role settings. For the offboarding part, just remove a test user from a group and disable their account, change password, pull any licenses and assignments then double check that the changes actually took effect.

## Tools used
`Microsoft Entra ID` `Conditional Access` `Privileged Identity Management` `Microsoft Entra groups`
