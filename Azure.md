# Azure

## Azure Imp Urls:

### Azure Portal:

https://portal.azure.com/

### To download latest PowerShell version refer below github repo:

https://github.com/PowerShell/PowerShell/releases

### Learning paths on MS Learn:

https://docs.microsoft.com/en-us/learn/certifications/exams/az-204#two-ways-to-prepare


### Azure Code Samples:

https://azure.microsoft.com/en-us/resources/samples/?sort=0


### Official Azure Documentation:

https://docs.microsoft.com/en-us/azure/



### Official Microsoft Developer YouTube Channel

https://www.youtube.com/channel/UCsMica-v34Irf9KVTh6xx-g



### Azure REST API Browser

https://docs.microsoft.com/en-us/rest/api/?view=Azure



### Microsoft Labs and Workshops - Practice is the key to success

* Azure Citadel - Labs and Workshops

https://azurecitadel.com/


### Github AZ-204 from Microsoft Training

https://microsoftlearning.github.io/AZ-204-DevelopingSolutionsforMicrosoftAzure/

### Microsoft has a Github page that contains labs for AZ-204:

https://github.com/MicrosoftLearning/AZ-204-DevelopingSolutionsforMicrosoftAzure


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

## WebApp (PAAS):

* Azure App service
* Platform as a service

## Running a WebApp in a VM:

* It's is entirely possible to run a web application in a virtual machine
* Install IIS on that VM
* Deploy your web app to that environment
* Responsibility : OS needs to be updated, dlls/apps installed on that machine, Open ports, virtual network settings etc.

## The new App Service Paradigm:

* Azure runs the server
* You don't have control over the underlyning OS or choose the hardware
* You pay for performance
* You upload your code and data
* Azure runs the app
* Easy to scale, lots of developer bells and whistles, less worries
* Similar price


## Create Web App:

![CreateWebApp1](/img/CreateWebApp1.JPG "CreateWebApp1")

* Subscription is a billing model (Azure subscription 1)
* Resource group is for organising structure like folder on desktop (newwebapp)
* Name - unique (rileo-h6a3cpfcfhhjdtbb.canadacentral-01.azurewebsites.net)
* Publish - Code / container
* Runtime stack - runtime version (e.g .NET 8/9/10 -> cross platform OS, ASP.Net V4.8 -> windows OS only) 
* Region - More than 60 regions (Canada central, East US, South India etc.)
* Windows Plan (Canada Central) - created ASPplandemo
* Pricing plans 

![AppServicePricing](/img/AppServicePricing.JPG "AppServicePricing")

![CreateWebApp2](/img/CreateWebApp2.JPG "CreateWebApp2")

![CreateWebApp3](/img/CreateWebApp3.JPG "CreateWebApp3")

![CreateWebApp4](/img/CreateWebApp4.JPG "CreateWebApp4")

![VirtualNetworkSetup](/img/VirtualNetworkSetup.JPG "VirtualNetworkSetup")

![CreateWebApp5](/img/CreateWebApp5.JPG "CreateWebApp5")

![CreateWebApp6](/img/CreateWebApp5.JPG "CreateWebApp5")

### Application Insights

* Azure Monitor application insights is an Application Performance Management (APM) service for developers and DevOps professionals. It will detect performance anomalies, and includes powerful analytics tools to help you diagnose issues and to understand what users actually do with your app. Your bill is based on amount of data used by Application Insights and your data retention settings.
* Enable Application Insights -> Feeds data in Azure monitor
* Telemetry framework -> To push certain telemetry to application insights from custom code


### Deploy an ASP.NET Core and Azure SQL Database app to Azure App Service:

https://learn.microsoft.com/en-gb/azure/app-service/tutorial-dotnetcore-sqldb-app?tabs=copilot&pivots=azure-portal


### Deploy to Azure App Service by using GitHub Actions:

https://learn.microsoft.com/en-us/azure/app-service/deploy-github-actions?tabs=openid%2Caspnetcore

![Deployment1](/img/Deployment1.JPG "Deployment1")

![Deployment2](/img/Deployment2.JPG "Deployment2")

![Deployment3](/img/Deployment3.JPG "Deployment3")

![Deployment4](/img/Deployment4.JPG "Deployment4")

![SiteSite](/img/Site.JPG "Site")

## App Service Plan:

![AppServicePlan](/img/AppServicePlan.JPG "AppServicePlan")

## Manual Scaling:

When scaling demand changes, you can manually scale your resource to a specific instance count, or via a custom Autoscale rule based policy that scales based on metric(s) thresholds, or schedule instance count which scales during designated time windows. You can also use Automatic Scaling features which enables platform managed scale in and scale out for your apps based on incoming HTTP traffic.

![ManualScaling](/img/ManualScaling.JPG "ManualScaling")

![ManualScaling2](/img/ManualScaling2.JPG "ManualScaling2")

![ManualScaling3](/img/ManualScaling3.JPG "ManualScaling3")


## Building and Publishing in Azure App service:

![Publish1](/img/Publish1.JPG "Publish1")

![Publish2](/img/Publish2.JPG "Publish2")

![Publish3](/img/Publish3.JPG "Publish3")




