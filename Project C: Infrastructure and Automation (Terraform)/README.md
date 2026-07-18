# Project 1: Infrastructure and Automation with Terraform

## The problem this solves

When a company builds its cloud environment by clicking through the Azure portal by hand, small mistakes creep in every time someone repeats the process. A setting gets forgotten. A resource gets named slightly differently. After a while, nobody can say with confidence that the staging environment actually matches production, and that gap is where outages and security gaps usually come from.

Terraform solves this by turning the environment into a file instead of a memory. The same file builds dev, staging, and production the exact same way, every time.

## What this builds

A resource group containing a virtual network with two subnets, a network security group that blocks everything by default except one narrow exception, a Windows VM sitting in the private subnet with no public IP, and a storage account. Terraform's own record of all of this, called state, is stored remotely in Azure Blob Storage rather than on a laptop.

---

## Step 1. Create the remote state storage account

```
az group create --name rg-tfstate --location northeurope

az storage account create --name sttfstate140726 --resource-group rg-tfstate --location northeurope --sku Standard_LRS

az storage container create --name tfstate --account-name sttfstate140726
```

**What this does.** Terraform needs somewhere to keep a record of everything it has built, called state, so it knows what already exists the next time it runs. This storage account is that record's home. It's the one piece built by hand, since Terraform can't manage the very thing it uses to manage everything else.

**Why it matters to a business.** If two people on a team ran Terraform from their own laptops with no shared state, they could each think they were the only one making changes, and end up overwriting each other's work without knowing it. Storing state remotely, in one shared place, is what makes Terraform safe to use in a team instead of just by one person alone.

![Resource group with storage account and tfstate container](screenshots/Step%201.%20Create%20resource%20group%2C%20storage%20account%20and%20storage%20container.png)

---

## Step 2. Set up the project structure

Created `main.tf`, `variables.tf`, `terraform.tfvars`, `outputs.tf`, `backend.tf` in a new folder, along with a `.gitignore` file listing:

```
.terraform/
*.tfstate
*.tfstate.backup
```

**What this does.** Splits the project into separate files instead of one long file: what gets built, what values can change, and what Terraform outputs afterward.

**Why it matters to a business.** Anyone opening this project later, whether that's a new hire, a reviewer, or a hiring manager, can find what they're looking for quickly instead of scrolling through one giant file. The `.gitignore` file also makes sure nothing sensitive, like the state file itself, ever accidentally gets uploaded to GitHub.

---

## Step 3. Configure the backend and initialize

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstate140726"
    container_name       = "tfstate"
    key                  = "project.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}
```

```
terraform init
```

**What this does.** Points Terraform at the remote storage account from Step 1, so every command from here on writes its progress there instead of onto a local machine.

**Why it matters to a business.** This is what would let a CI/CD pipeline, or a second engineer, safely run the same project without needing anything from the original laptop it was built on.

![Terraform successfully initialized](screenshots/Step%203.%20Terraform%20has%20been%20successfully%20initialized.png)

---

## Step 4. Resource group, network, and subnets

```hcl
resource "azurerm_resource_group" "main" {
  name     = "${var.project_name}-rg"
  location = var.location
}

resource "azurerm_virtual_network" "main" {
  name                = "${var.project_name}-vnet"
  address_space       = var.vnet_address_space
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "general" {
  name                 = "${var.project_name}-general-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = var.public_subnet_prefix
}

resource "azurerm_subnet" "private" {
  name                 = "${var.project_name}-private-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = var.private_subnet_prefix
}
```

```
terraform plan
terraform apply
```

**What this does.** Builds the network and splits it into two separate sections, a general purpose subnet and a private one.

**Why it matters to a business.** Think of the two subnets like two separate rooms in an office. The general subnet is the open floor where everyday traffic passes through. The private subnet is the locked back room where the sensitive machine lives. Splitting them means a strict lock can be put on just the back room, without also locking the front door that everyone needs to use.

![Terraform plan output, part 1](screenshots/step4-plan-1.png)
![Terraform plan output, part 2](screenshots/step4-plan-2.png)
![VNet and both subnets in the Azure portal](screenshots/step4-vnet-subnets.png)

---

## Step 5. Network security group

```hcl
resource "azurerm_network_security_group" "private_nsg" {
  name                = "${var.project_name}-private-nsg"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowRDPFromGeneralSubnet"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = "10.0.1.0/24"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "private_assoc" {
  subnet_id                 = azurerm_subnet.private.id
  network_security_group_id = azurerm_network_security_group.private_nsg.id
}
```

**What this does.** Blocks every kind of inbound traffic to the private subnet by default, then opens exactly one narrow exception, allowing remote desktop access only from the general subnet.

**Why it matters to a business.** This is the same logic as a company handing out door keys. Starting from "nobody has a key unless we specifically gave them one" is far safer than starting from "everyone has a key until we take theirs away." If a rule gets forgotten, the safe version fails closed, and the risky version fails open. Most real-world breaches happen because something was left open that someone forgot about.

![NSG rule list showing deny-all and the one narrow allow rule](screenshots/step5-nsg-rules.png)

---

## Step 6. Virtual machine, with no public IP

```hcl
resource "azurerm_network_interface" "vm_nic" {
  name                = "${var.project_name}-vm-nic"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.private.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_windows_virtual_machine" "vm" {
  name                = "${var.project_name}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = var.vm_size
  admin_username      = var.admin_username
  admin_password      = var.admin_password

  network_interface_ids = [azurerm_network_interface.vm_nic.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-Datacenter"
    version   = "latest"
  }

  lifecycle {
    ignore_changes = [identity, vm_agent_platform_updates_enabled]
  }
}
```

```hcl
variable "vm_size" {
  description = "Size of the virtual machine"
  type        = string
  default     = "Standard_B2ats_v2"
}

variable "admin_username" {
  description = "Admin username for the VM"
  type        = string
  default     = "azureadmin"
}

variable "admin_password" {
  description = "Admin password for the VM"
  type        = string
  sensitive   = true
}
```

**What this does.** Builds a Windows VM inside the private subnet, with no public IP address anywhere in its configuration.

**Why it matters to a business.** A machine with no public IP has no direct path in from the open internet at all. The only way to reach it is from inside the network itself. This is the single decision that does the most security work in the whole project, since it removes an entire category of attack before anything else is even configured.

**Why the password is never written down.** The `admin_password` variable has no default value and is marked `sensitive`. Because there's no default, Terraform stops and asks for the password directly in the terminal every time the project is built, instead of reading it from a file. That means the password never sits anywhere in the code, so there is nothing for GitHub to accidentally leak. This mirrors a basic rule in any company handling credentials: sensitive values should never live in plain text, they should be entered at the moment they're needed and nowhere else.

### Problem 1: the VM size and region I originally picked didn't work

The first VM size and region I chose both failed when I ran `apply`. I checked what sizes and regions were actually valid for the subscription, corrected the VM size to `Standard_B2ats_v2` and the region to Sweden Central, and reran it. This is a normal part of working with cloud providers: not every size or region is available everywhere, and the fix is simply checking availability and adjusting the configuration rather than assuming the first choice will always work.

### Problem 2: an apply that looked broken but wasn't

While the VM was being created, `terraform apply` returned an error partway through. Instead of panicking, I checked the Azure portal directly and found the VM had actually finished building successfully. The error was a timing issue with a resource created right after the VM, a known rough edge in how Azure occasionally handles a large deployment with many resources at once.

Rather than deleting everything and starting over, I compared what Terraform believed existed, using `terraform state list`, against what was genuinely sitting in Azure. Two resources, the VM itself and the general subnet, existed in Azure but were missing from Terraform's own records. I brought Terraform's records back in line with reality using `terraform import`, pointing it at the exact resources already running in Azure. Once that was done, `terraform plan` came back clean, confirming Terraform and Azure agreed again.

The lesson here is one that applies well beyond Terraform: an error message on screen doesn't always mean something actually failed. Checking the real state of a system before reacting, rather than assuming the error message is the whole story, is a habit that matters just as much in security work as it does in infrastructure.

### Problem 3: the identity that kept reappearing

Azure automatically attaches a system identity to the VM to support a built-in compliance feature. Terraform didn't know why that identity was there, saw it as something it hadn't created, and removed it every time it ran. Azure then put the identity straight back. This created a loop where every single apply undid something Azure needed.

The fix was telling Terraform to leave that one specific setting alone, since it was legitimately being managed by something else:

```hcl
lifecycle {
  ignore_changes = [identity, vm_agent_platform_updates_enabled]
}
```

The lesson: not every setting on a cloud resource belongs to the tool that built it. Recognising when something is being managed elsewhere, and telling your automation to step back from it, is part of running infrastructure safely rather than fighting the platform it runs on.

![VM networking tab showing no public IP](screenshots/step6-vm-no-public-ip.png)

---

## Step 7. Proving the isolation is real

I tried opening the VM's private IP address directly in a browser, from my own machine, outside the network.

**What this does.** Confirms the isolation actually works in practice, not just on paper.

**Why it matters to a business.** A setting that looks correct in the configuration file isn't proof that it actually behaves that way once deployed. Testing from a genuinely outside position, the same way an attacker would have to, is the only real way to confirm a system is protected. This connection failed, which is exactly the result that was expected.

![Failed connection attempt from outside the VNet](screenshots/step7-isolation-proof.png)

---

## Step 8. Storage account, parameterization, and outputs

```hcl
resource "azurerm_storage_account" "logs" {
  name                     = "${var.project_name}logs140726"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

output "vm_private_ip" {
  value = azurerm_network_interface.vm_nic.private_ip_address
}
```

Every value that could change between environments, the region, the VM size, the naming prefix, lives in `variables.tf` and `terraform.tfvars`, and is never hardcoded directly into the main configuration.

**What this does.** Adds a storage account for logging, and confirms that all the configurable details of the project sit in one place, separate from the actual logic of what gets built.

**Why it matters to a business.** This is what makes the project reusable rather than a one-time build. Standing up the exact same environment again for staging should only ever mean changing a handful of values in one file, never rewriting the underlying logic. That's the difference between infrastructure that scales across a company and infrastructure that only ever worked once.

![Terraform apply output showing the VM private IP](screenshots/step8-apply-output.png)
![Storage account created for logging](screenshots/step8-storage-account.png)

---

## Step 9. Proving it's stable, then tearing it down

```
terraform plan
```

Reported no changes, since nothing had been edited since the last apply.

![Terraform plan showing no changes](screenshots/step9-plan-no-changes.png)

```
terraform destroy
```

![Destroy prompt requesting the admin password](screenshots/step9-destroy-prompt.png)
![Destroy plan showing 9 resources to remove](screenshots/step9-destroy-plan.png)
![Destroy complete, 9 resources destroyed](screenshots/step9-destroy-complete.png)

**What this does.** Confirms the configuration and the real environment genuinely match, then removes every resource that was built.

**Why it matters to a business.** A clean "no changes" result is proof that what's written in the code and what's actually running in Azure are in complete agreement, which is the entire reason to use Terraform instead of building things by hand. Tearing everything down afterward confirms that the whole environment, network, security rules, VM, and storage, was genuinely defined in code, with nothing left over that was quietly built by hand and forgotten about.

---

## What I would add next

Looking at where this project stands right now, the next thing I would build is a small pipeline in GitHub Actions that automatically checks and previews any change to this infrastructure before it goes live, requiring someone to manually approve it before it actually applies. I would also add lifecycle rules to the storage account, so older logs automatically move to cheaper storage or get deleted after a set period instead of sitting there indefinitely. Finally, I would extend this same project into a second environment for staging, changing nothing but a handful of values in one file, to prove out loud that the reusability this project was built around actually holds up in practice.
