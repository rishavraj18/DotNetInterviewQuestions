# Azure

## Azure Imp Urls:

### Azure Portal:

https://portal.azure.com/

### To download latest PowerShell version refer below github repo:

https://github.com/PowerShell/PowerShell/releases

to be continued...

## PowerShell Commands:

### Get powershell version

```powershell
$PSVersionTable.PSVersion 
```

### PowerShell module for interacting with Azure list:

 Az offers shorter commands, improved stability, and cross-platform support.

```powershell
Get-Module -Name Az -ListAvailable
```

### Install PowerShell module for interacting with Azure list:

* Installs a PowerShell module from the PowerShell Gallery (the official public module repository).
* PowerShell downloads the Az module and its dependencies
* Installs them for your user
* You can start using Azure commands immediately

```powershell
Install-Module -Name Az -AllowClobber -Scope CurrentUser
```

* AllowClobber : Allows the installation even if command names conflict with existing modules
* Scope CurrentUser : Installs the module only for the current user, not system-wide. No admin rights required. 
* Alternative -Scope AllUsers. Requires Administrator privileges and installs for all users.

https://learn.microsoft.com/bs-latn-ba/powershell/azure/install-az-ps?view=azps-0.10.0

| Parameter            | Meaning                        |
| -------------------- | ------------------------------ |
| `Install-Module`     | Installs a PowerShell module   |
| `Az`                 | Azure PowerShell SDK           |
| `-AllowClobber`      | Overwrites conflicting cmdlets |
| `-Scope CurrentUser` | Install only for current user  |

#### With Az, you can:

* Log in to Azure (Connect-AzAccount)
* Create/manage VMs, App Services, Storage, SQL, etc.
* Manage Azure resources via scripts and automation
* Az replaces the older AzureRM module.


## Step By Step setup Azure for development:

* create Azure account
* install PowerShell
* install the Az module

to be continued...

## Create Web App:

* Subscription is a billing model. 
* Resource group is for organising structure(like folder on desktop).
* Name - unique
* Publish - Code / container
* Runtime stack - runtime version (e.g .NET 8/9/10 -> cross platform OS, ASP.Net V4.8 -> windows OS only) 
* Region - More than 60 regions (Canada central, East US etc.)
* Pricing plans - 

to be continued...