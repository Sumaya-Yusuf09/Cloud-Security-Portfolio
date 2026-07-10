# Project A - Identity and Access Management in Microsoft Entra ID

## Tools Used

`Microsoft Entra ID` `Conditional Access` `Privileged Identity Management` `Azure Portal`

## Why this project

On May 27, 2026, Charter Communications confirmed a breach that started with one phone call. Someone posing as IT support talked an employee into handing over their Microsoft Entra login. No hacking, just a call and a password. You can read the coverage here: [Charter confirms data breach after ShinyHunters extortion threat](https://www.bleepingcomputer.com/news/security/charter-confirms-data-breach-after-shinyhunters-extortion-threat/).

That one login was enough to reach Charter's Salesforce systems. Attackers claim they pulled over 40 million customer records, names, addresses, phone numbers, support tickets.

Charter reportedly knew in early April and did not confirm it publicly until late May, which turns a security problem into a trust problem too.

The lesson here, one login should never be the only thing standing between an attacker and everything. If someone talks their way into an identity, there needs to be a second check in place, and a limit on what that identity can actually reach.

This project builds both of those things. Multifactor Authentication (MFA) so a stolen password alone is not enough. Time limited admin access through Privileged Identity Management (PIM) so a compromised account never comes with standing privileges attached. And a real offboarding, since access has to actually get removed too, not just granted carefully.

## What I built

I started by creating test users and two groups, IT Admins and Finance Users, so I had a realistic setup rather than testing against my own single account.

![Storage account created](screenshots/Step%201.%20Create%20the%20resource%20group%20and%20storage%20account.png)

![Users created in Entra ID](screenshots/Step%201.%20User_Created-Azure.png)
![Groups created in Entra ID](Step%202.%20Groups_Created-Azure.png)

The first real control was Conditional Access requiring MFA for everyone. I excluded my own account while building it, so I would not accidentally lock myself out mid setup, a small precaution that matters more than it sounds like it should.

![New policy excluding my own account while testing](Step%203.1%20New%20policy%20for%20all%20users%20except%20Sumaya%20Yusuf.png)
![Grant control set to require MFA](Step%203.2%20Adding%20control%20access%20to%20the%20new%20policy.png)
![MFA policy turned on and active](Step%204.%20Check%20MFA%20policy%20is%20on.png)

I could have relied on Azure's basic security defaults instead of building a custom policy. Security defaults turn on MFA broadly but give almost no control over how or when. Conditional Access lets you combine conditions, MFA plus location, MFA plus device state, and that flexibility is closer to how a real company actually needs to configure this. I chose Conditional Access for that reason.

On top of MFA, I added a second policy blocking sign in from outside the EU.

![Both policies now active](Step%205.%20Policy%20List%20after%20adding%20another%20policy.png)

A policy is only worth something if it actually behaves the way you expect. I used Entra's What If tool to simulate a sign in from Brazil and confirmed both policies would correctly trigger and block it, without needing to actually risk a real login attempt from outside the EU to prove it.

![Setting up the What If simulation](Step%206%20Testing%20-%20What%20if%20%28policy%29.png)
![What If result showing both policies would apply](Step%206%20Testing%20%28continued%20screenshot%29%20-%20What%20if%20%28Policy%29.png)

This is the part of the project that speaks most directly to the Charter incident. Standing admin access, a login that always has elevated privileges with no expiry, is exactly what turns one stolen credential into full access. I used Privileged Identity Management so the IT Admins group is only eligible for the Security Reader role, not permanently assigned it. Getting the role active requires a written reason, and it automatically expires after a few hours.

![Security Reader role assigned to IT Admins as eligible](Step%207.%20Assigned%20Security%20Reader%20to%20IT%20Admins.png)
![Checking the assignment shows up correctly](Step%208.%20Check%20Assignment%20List.png)
![Confirming the eligible assignment for IT Admins](Step%208.%20%28Continued%20screenhost%29%20Assigned%20Security%20Reader%20to%20IT-Admins.png)
![Role settings requiring approval and justification](Step%209.%20Setting%20Assignment%20duration%20%26%20Requiring%20approval%20upon%20acitivations.png)

I could have just assigned the role permanently and trusted that nobody would misuse it. That is the weaker option, since a permanent assignment sitting unused is precisely what an attacker benefits from if that account is ever the one compromised. Making it eligible and time boxed instead shrinks that window down to hours instead of forever.

To prove this actually works end to end rather than just existing as a setting, I had a test user, Sarah, request activation with a written reason, then approved that request myself as an admin.

![Sarah requesting activation with a written reason](Step%2010.%20Assignment%20Activation%20Request%20%28as%20the%20user%20Sarah%29.png)
![Approving Sarah's request](Step%2011.%20Approve%20Assignment%20Request.png)

The last part of this project is offboarding, since Charter's incident is also a reminder that access has to actually get removed, not just granted carefully. I picked a test Finance User, Moa Ali, and walked through what leaving a company should actually look like. First I confirmed Moa was a member of the Finance Users group.

![Finance Users group before offboarding](Step%2012.%20Check%20Finance%20User%20List.png)

Then I removed Moa from the group, so anything tied to that group's access is gone immediately.

![Moa removed from the Finance Users group](Step%2013.%20Offboarding.%20Remove%20Moa%20Ali%20from%20Finance%20Users%20Group.png)

Removing someone from a group is not the whole job. I also disabled the account directly, which blocks every sign in attempt and ends any active session across Microsoft services right away.

![Disabling Moa's account to block sign in](Step%2014.%20Blocking%20Moa%20Ali%20from%20signing%20in.%20Denied%20Access.png)

I could have done just one of these two steps and called it finished. Removing from the group alone leaves the account itself still active, still able to sign in and potentially reach anything not tied to that specific group. Disabling the account alone, without cleaning up group membership, leaves stale permissions sitting around if the account is ever re-enabled by mistake later. Doing both closes the gap that either one on its own would have left open.

![Confirming the account status is disabled](Step%2015.%20Check%20If%20Moa%20Ali%20Account%20Disabled.png)

## What I would test further next time

Nothing broke while building this one, so instead of inventing struggles, here is what I would genuinely add if I rebuilt it. I would simulate the actual Charter scenario more directly, a successful password compromise followed by an MFA prompt, to confirm the policy stops the sign in at that exact second step rather than just trusting the configuration looks right. I would also test what happens if PIM approval is requested but never approved, to confirm access genuinely stays blocked rather than defaulting open after some timeout. And I would add a Conditional Access policy specifically requiring a compliant or known device for admin role activation, since Charter's attackers succeeded through social engineering on a device Charter never controlled in the first place.

## How to do this yourself

1. Create a handful of test users and two groups in Microsoft Entra ID, so you are not testing against your own single account.
2. Build a Conditional Access policy requiring MFA for all users, excluding your own account while you set it up.
3. Add a second Conditional Access policy blocking sign in from outside your target region.
4. Use the What If tool to simulate a sign in from outside that region and confirm both policies trigger correctly.
5. Set up Privileged Identity Management so an admin role is eligible rather than permanently assigned, with approval and justification required to activate it.
6. Have a second test account request activation with a written reason, then approve it yourself to confirm the full flow works.
7. Pick a test user, confirm their group membership, then remove them from the group as part of a mock offboarding.
8. Disable that same account directly, and confirm its status actually shows as disabled afterward.
