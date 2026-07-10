# Project B: Storage Misconfiguration and Hardening

## Tools Used

`WIndows PowerShell` `Azure Portal` `Azure Storage Account` `Azure Blob Storage` `Azure Log Analytics`

## Why this project, right now

On July 2026, Accenture confirmed a security incident after a threat actor claimed to have stolen about 35GB of source code, RSA and SSH keys, Azure personal access tokens, and Azure Storage access keys, reportedly taken from an exposed Azure DevOps repository. You can read the coverage here: [Accenture acknowledges security incident following 35GB data theft claim](https://www.helpnetsecurity.com/2026/07/08/accenture-data-breach-2026/).

Here is the part that actually matters if you are not technical at all. Nobody hacked their way in with some clever trick. A storage location that should have stayed private was reachable by anyone who found the address, the same way a filing cabinet left unlocked in an empty office is reachable by anyone who wanders in. That is the whole story. And Accenture, a company whose entire business is helping other companies stay secure, has now had this exact type of incident twice, once in 2017 with four open AWS buckets, and again in 2026. That repetition is the real lesson here. This is not a beginner mistake that only happens to small, careless teams. It happens to companies with enormous security budgets, because the mistake is boring and easy to miss, not because it is hard to understand.

This project recreates that exact mechanism on a small, controlled scale. I deliberately left an Azure storage container open the same way, proved from outside my own account that it was genuinely reachable, then closed it properly and proved that too.

## What I built 

Everything below follows one thread from start to finish. Open the door. Prove it is actually open by walking through it myself, from the outside. Close the door properly. Prove it is actually closed. Then build a smarter way back in for the one person who might legitimately need it, instead of leaving the door sealed forever or propped open for everyone.

I started by creating a resource group and a storage account in North Europe.

```
az group create --name rg-storagehardening --location northeurope
```

```
az storage account create --name projectb20 --resource-group rg-storagehardening --location northeurope --sku Standard_LRS --allow-blob-public-access true
```

That last flag, `--allow-blob-public-access true`, is the switch for the whole account. Every container inside a storage account inherits this setting as a ceiling. If it is off, nothing inside can ever be made public no matter what you do at the container level. I turned it on deliberately here, since the entire point of this project is to walk through what happens when that switch gets left in the wrong position, which is exactly the kind of thing that happens by accident during a rushed setup in a real company.

With that switch flipped on, I created the actual container and set it to public.

```
az storage container create --name testdata --account-name projectb20 --public-access blob
```

![Storage account created](Screenshotss/Step%201.%20Create%20the%20resource%20group%20and%20storage%20account.png)

This single flag is the real misconfiguration. It is the difference between a folder only you can open and a folder anyone with the link can open, and it looks like a harmless, quick setting to someone moving fast.

I uploaded a small test file so there would be something real to actually test against.

```
"this is a test file, not real customer data" | Out-File -FilePath testfile.txt -Encoding ascii
```

```
az storage blob upload --account-name projectb20 --container-name testdata --name testfile.txt --file testfile.txt
```

Then came the part to double check it, rather than just describing a setting. I copied the file's public address and opened it in a private browser window that had never logged into my Azure account at all.

```
az storage blob url --account-name projectb20 --container-name testdata --name testfile.txt -o tsv
```

![File loading with no authentication](Screenshotss/Step%202.%20Open%20the%20file%20in%20a%20cognito%20browser.PNG)

The file loaded instantly. Without asking for any password, or showing any warning, nothing standing between an anonymous browser and the file's contents. This is precisely what the Accenture story looked like from the outside, minus the actual damage, since I control both the door and everything sitting behind it.

Closing it back up meant reversing that same switch at both levels, not just one.

```
az storage account update --name projectb20 --resource-group rg-storagehardening --allow-blob-public-access false
```

```
az storage container set-permission --name testdata --account-name projectb20 --public-access off
```

I could have only changed the container's own setting and left the account level flag turned on. That would have closed this specific container while leaving the master switch itself still open, meaning the very next container anyone creates in this account could just as easily end up exposed again without anyone noticing. Turning the account level flag off as well closes that entire category of mistake permanently, not just this one instance of it. That is the stronger fix, and it is the one I chose.

Encryption at rest is on by default in Azure, but I checked it directly in the portal rather than assuming.

![Encryption confirmed](Screenshotss/Step%203.%20Confirm%20Encryption.png)

Logging took a bit more digging than I expected. The account's general diagnostic settings page only offers a generic transaction count, nothing that actually shows who read, wrote, or deleted anything. That specific detail lives one level deeper, scoped to the blob service itself rather than the account as a whole.

```
az monitor log-analytics workspace create --resource-group rg-storagehardening --workspace-name law-storagehardening --location northeurope
```

![Blob logging configured](Screenshotss/Step%204.%20Enabling%20blob%20read%20write%20delete%20logging%20to%20Log%20Analytics.png)

![Diagnostics status confirmed](Screenshotss/Step%205.%20Show%20logging%20enabled%20on%20Diagnostic%20Settings%20page.png)

Encryption protects the data itself if someone ever reached the underlying disk directly. Logging protects something different, visibility, meaning any future attempt to touch this account, whether it succeeds or gets rejected, actually leaves a trace somewhere reviewable instead of vanishing into nothing.

Then I went back to test the fix the same way I tested the original exposure, same file, same private browser window, same URL from before.

![Access denied after the fix](Screenshotss/Step%206.%20Re-testing%20the%20exposure%20to%20confirm%20the%20fix%20actually%20worked.PNG)

Trusting that a portal setting says "off" is not the same as watching the actual behavior change from the outside. This time the platform itself refused the request outright, with an error that confirms it is actively blocking access, not just displaying a setting that looks correct.

The last real decision in this project was what to do once the door was properly shut. I could have left it sealed completely, no way in for anyone, ever. That is the simplest option, but it is likely that's not how real teams actually operate, since there is almost always a legitimate reason someone needs to reach a specific file eventually, and a permanently sealed system just pushes people toward bad workarounds, like quietly turning public access back on out of frustration. The better option is a narrow, temporary opening built for exactly one purpose, so I generated a Shared Access Signature scoped to read only, on this one file, expiring on a fixed date rather than staying valid indefinitely.

```
az storage blob generate-sas --account-name projectb20 --container-name testdata --name testfile.txt --permissions r --expiry 2026-08-01T00:00Z
```

![SAS token generated](Screenshotss/Step%207.%20Apply%20least%20privilege%20for%20ligitimate%20access.webp)

That command only prints the permission string, not a full working link, so I attached it to the blob's normal address with a question mark in between, the same way any web address carries extra instructions after that symbol.

![SAS URL working](Screenshotss/Step%208.%20Check%20if%20the%20SAS%20URL%20works%20from%20step%208..PNG)

The regular public address still fails exactly like it did after I locked things down. This signed link works, but only for reading, and only until the date I set. Closed by default, open only through something deliberately handed out, scoped, and temporary. And that is the ending of this whole project.

Finally, I deleted everything.

```
az group delete --name rg-storagehardening --yes --no-wait
```

Leaving pieces like an unused Log Analytics workspace sitting around after a project is finished is an easy, quiet way to keep paying for something nobody is using anymore, so I tore down the whole resource group at once rather than picking through it resource by resource.

## What I would test further next time

Nothing actually broke while building this one, so rather than inventing struggles I did not have, here is what I would genuinely add if I built it again. I would go into the Log Analytics workspace and confirm my own test requests actually show up as recorded events, rather than trusting that turning logging on was enough on its own, since configuring a control and confirming it is actually capturing data are two different claims. I would also generate the SAS token with a much shorter expiry, measured in hours instead of weeks, specifically for anything going into a public README, since a long lived signed link sitting in a public repository is technically a live credential for as long as it stays valid. And I would deliberately try turning public access back on afterward, just to watch the account level override actually block it, rather than only ever testing the fix once.

## How to do this yourself

1. Set up a free Azure subscription and open Windows PowerShell.
2. Create a resource group and a storage account, setting `--allow-blob-public-access true` on the account.
3. Create a container inside it with `--public-access blob`.
4. Upload a clearly labeled test file, nothing sensitive.
5. Copy the file's public URL and open it in a private browser window that is not logged into your Azure account. Confirm it loads with no authentication.
6. Disable public access at both the storage account level and the container level.
7. Confirm encryption is active under the storage account's Encryption settings.
8. Enable diagnostic logging scoped to the blob service specifically, sending Storage Read, Storage Write and Storage Delete events to a Log Analytics workspace.
9. Retest the same URL in the same private browser and confirm it now fails.
10. Generate a Shared Access Signature scoped to read only with a set expiry, combine it with the base URL using a question mark, and confirm it works while the regular URL still fails.
11. Delete the resource group once you are finished to avoid any ongoing cost.
