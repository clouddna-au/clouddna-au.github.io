---
layout: blog
title: 800 limit on Resource groups
---

As quoted from Microsoft around [resource group limits](https://docs.microsoft.com/en-us/azure/azure-subscription-service-limits#resource-group-limits)
> Maximum limit
Resources per resource group, per resource type	**800**	Some resource types can exceed the 800 limit. See Resources not limited to 800 instances per resource group.


### Resource group deployments
![Deployments](https://clouddna-au.github.io/assets/images/blog/2019-11-09/deployments.jpg)

When you get to your limit, you receive the following error when trying to deploy anything to the resource group:
> Creating the deployment 'Your-Deployment-Name-Goes-Here' would exceed the quota of '800'


## Steps to periodically remove deployments from the resource group
### 1. Enable Managed Identity

Enable Managed Identity for Azure Function
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/managedidentity.jpg)

### 2. Enable Service Principal access to managed resources IAM
Navigate to Resource group and add in Access Control (IAM) the azure function app as contributor to the resource group. Select Function App in Find box and search for Function app name. Add Function app as contributor to Resource Group
### 3. Add/Update the following files:
#### profile.ps1
`Profile.ps1` is needed as it provides the connection to use Azure AZ powershell modules. It authenticates the principal against Azure so any powershell cmdlets can be executed.

```powershell
if ($env:MSI_SECRET -and (Get-Module -ListAvailable Az.Accounts)) {
    Connect-AzAccount -Identity
}
```

#### requirements.psd1

```powershell
@{
    # For latest supported version, go to 'https://www.powershellgallery.com/packages/Az'. Uncomment the next line and replace the MAJOR_VERSION, e.g., 'Az' = '2.*'
     'Az' = '2.*'
}
```
