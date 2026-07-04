# Project 1: Identity and Access Management with Microsoft Entra ID

## The problem

Most small companies hand out access however feels convenient in the moment. Someone joins, they get added to a group. Someone becomes an admin, they keep that access forever, long after they needed it. Someone leaves the company, and their account just sits there active because nobody remembered to turn it off.

That is how breaches happen. Not because of some clever hacker, but because access nobody is watching around anymore.

For this project I set up a small test environment in Microsoft Entra ID and built the kind of access controls a real company would need: forcing MFA, blocking risky sign ins, making admin access temporary instead of permanent, and having an actual process for removing access when someone leaves.

## Why I made these choices

**Why I used Conditional Access instead of just turning on basic MFA for everyone the same way.**
Basic security defaults are all or nothing. Conditional Access lets you build rules based on who the person is, where they are signing in from, and what device they are using. A real company almost never wants one blanket rule for every single person, they want rules that adapt to risk. So I built two separate policies instead of one, to actually show that thinking.

**Why I used PIM (Privileged Identity Management) instead of just adding people to the admin group directly.**
If an admin account gets compromised and that access is permanent, the attacker has permanent access too. PIM means admin rights only exist for a few hours at a time, and only after someone approves the request. It shrinks the window an attacker even has something to exploit. This is the single biggest thing that separates someone who understands identity security from someone who just knows how to add a user to a group.

**Why I tested with the What If tool instead of just trusting the policy would work.**
Writing a policy and assuming it works is how misconfigurations slip through in real environments. The What If tool actually simulates a sign in and tells you which policies would apply. I used it to prove the rule actually does what I intended, not just what I assumed.

## What I actually built, step by step

### 1. Setting up test users and groups

I created a handful of fake employees in Entra ID and put them into two groups, IT Admins and Finance Users. This is the starting point every access control system needs, you cannot restrict access to people and groups that do not exist yet.

![Groups created in Azure](screenshots/Groups_Created-Azure.png)

### 2. Requiring MFA for everyone

The first policy I built forces multi factor authentication for all users. I excluded my own account from this one policy for a very practical reason, so I would not lock myself out of the tenant while testing everything else.

![Adding grant control to the new policy](screenshots/Adding_control_access_to_the_new_policy.png)

![New policy excluding my own account](screenshots/New_policy_for_all_users_except_Sumaya_Yusuf.png)

![Policy showing as on in the list](screenshots/MFA_policy_is_on.png)

### 3. Blocking sign ins from outside the EU

The second policy blocks access attempts coming from outside Sweden and the EU. I cannot fake my own real location to test this properly, so instead of pretending, I used Microsoft's own simulation tool to prove it works, which is explained in the next step.

![Both policies now showing in the list](screenshots/Policy_List.png)

### 4. Proving the policy actually works using What If

Instead of just assuming a policy is configured correctly, I ran a simulated sign in through the What If tool using a fake IP address from Brazil. The tool showed both policies applying to that sign in attempt, which confirms they are working the way they were meant to.

![Testing the sign in with What If](screenshots/Testing_-_What_if__policy_.png)

![Evaluation result showing which policies apply](screenshots/Testing__1__-_What_if__Policy_.png)

### 5. Making admin access temporary with PIM

Instead of giving the IT Admins group permanent Security Reader access, I set it up through PIM so that access has to be requested, approved, and only lasts for a limited time. I set the max activation time and required both a written justification and an approval before anyone can actually use the role.

![Setting activation duration and approval requirement](screenshots/Setting_Assignment_duration___Requiring_approval_upon_acitivations.png)

![Adding the eligible assignment for IT Admins](screenshots/Assigned_Security_Reader_to_IT_Admins.png)

![Assignment now showing as eligible](screenshots/Assignment_List.png)

Here is what it looks like when someone actually needs that access. They submit a request with a reason.

![Requesting activation with a written reason](screenshots/Assignment_Activation_Request.png)

And then someone else has to approve it before it becomes active.

![Approving the activation request](screenshots/Approve_Assignment_Request.png)

### 6. Simulating someone leaving the company

This is the part most portfolios skip completely, but it is one of the most important parts of real identity security. I simulated an employee named Moa Ali leaving the company, and walked through what should happen to her access.

Before she leaves, she is an active member of the Finance Users group.

![Finance users group before offboarding](screenshots/Finance_User_List.png)

When someone leaves, the first thing to do is stop them from being able to sign in at all, before anything else.

![Disabling the account](screenshots/Blocking_Moa_Ali_from_signing_in_-_Offboarding.png)

![Account now shows as disabled](screenshots/Account_Disabled_-_Moa_Ali.png)

Then she gets removed from the group that gave her access to finance systems.

![Finance users group after she is removed](screenshots/Moa_Ali_removed_from_Finance_Users_Group.png)

This two step process matters. Disabling the account first means even if the group removal takes a moment, she already cannot sign in. Companies that skip this step are the ones you read about in breach reports months later, where an ex employee's account was still active and nobody noticed.

## What this actually proves

Anyone can click through a tutorial and end up with a working Conditional Access policy. What I wanted this project to show is the thinking behind it, why time limited admin access matters more than the policy itself, why testing a policy before trusting it matters, and why offboarding is not just deleting an account but a sequence that needs to happen in the right order.

This is the kind of judgment I would bring into a SOC analyst or cloud security role, not just knowing which button to click, but understanding why the button matters.
