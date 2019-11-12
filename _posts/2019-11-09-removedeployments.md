---
layout: blog
title: 800 limit on Resource groups and a way around it
---

Have you ever tried to deploy to a Microsoft Azure resource group but received the following error message
> <span style="color:red">Creating the deployment 'Your-Deployment-Name-Goes-Here' would exceed the quota of '800'</span>

thats because there is a limit on the number of deployments that can be made to one resource group. According to Microsoft: [resource group limits](https://docs.microsoft.com/en-us/azure/azure-subscription-service-limits#resource-group-limits)
> Maximum limit
Resources per resource group, per resource type	**800**	Some resource types can exceed the 800 limit. See Resources not limited to 800 instances per resource group.

800 could sound like a lot, but because of certain limitations we were forced to use one resource group for our Dev, Test, Integration environments which both included multiple key vaults, app services, redis caches, storage accounts, AppInsights. So by deploying secrets and code to the environment using CI/CD, we were quickly stacking up the deployment tally.

To remedy this, you can deploy a consumption based Azure function using Powershell to remove deployments on a schedule.

### Resource group deployments
![Deployments](https://clouddna-au.github.io/assets/images/blog/2019-11-09/deployments.jpg)


## Steps to periodically remove deployments from the resource group using a schedule
### 1. Create new timer based powershell function application
* https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-function-powershell

### 2. Add/Update the following files:
#### profile.ps1
`Profile.ps1` is needed as it provides the connection to use Azure AZ powershell modules. It authenticates the principal against Azure so any powershell cmdlets can be executed.

```powershell
if ($env:MSI_SECRET -and (Get-Module -ListAvailable Az.Accounts)) {
    Connect-AzAccount -Identity
}
```

#### requirements.psd1
Ensures the right version of powershell module is loaded for function runtime. v2 is the latest.
```powershell
@{
    # For latest supported version, go to 'https://www.powershellgallery.com/packages/Az'. Uncomment the next line and replace the MAJOR_VERSION, e.g., 'Az' = '2.*'
     'Az' = '2.*'
}
```

#### function.json
For this function I specified a timer based function that would run on a schedule of 8:30pm every night. Any [CRON Expression](https://codehollow.com/2017/02/azure-functions-time-trigger-cron-cheat-sheet/) will work:
```json
{
  "bindings": [
    {
      "name": "Timer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 30 20 * * *"
    }
  ]
}
```

### 3. Add function configuration settings
The example code uses environment variables from the Azure function runtime to read the resource group name and subscription id. This can be set using either the portal through the configuration section
![Configuration](https://clouddna-au.github.io/assets/images/blog/2019-11-09/configurationfunction.jpg)
or using Powershell to set settings for the Application Service.

### 4. Add the code
This code will generate an access token to communicate with management api, retrieve all deployments for a resource group and then delete each deployment for that resource group.

```powershell
# Input bindings are passed in via param block.
param($Timer)

Write-Host "Getting resource deployments"
$azContext = Get-AzContext

Write-Host "resource group: " $env:resourceGroup
Write-Host "subscription id: " $env:subscriptionId

# Get access token to communicate with azure management api
$azureRmProfile = [Microsoft.Azure.Commands.Common.Authentication.Abstractions.AzureRmProfileProvider]::Instance.Profile;
$profileClient = New-Object Microsoft.Azure.Commands.ResourceManager.Common.RMProfileClient($azureRmProfile);
$accessToken = $profileClient.AcquireAccessToken($azContext.Tenant.TenantId).AccessToken;

$commandUri = "https://management.azure.com/subscriptions/$env:subscriptionId/resourceGroups/$env:resourceGroup/providers/Microsoft.Resources/deployments/" + "?top=1&api-version=2019-05-10"

# Get resource group deployments
$response = Invoke-RestMethod -Method Get -Uri $commandUri -Headers @{ Authorization="Bearer $accessToken" }

Write-Host "Number of Deployments: "$response.value.length

# Remove each deployment
foreach ($deployment in $response.value){
    $deploymentname = $deployment.name
    $deleteDeploymentUri = "https://management.azure.com/subscriptions/$env:subscriptionId/resourceGroups/$env:resourceGroup/providers/Microsoft.Resources/deployments/$deploymentname" + "?api-version=2019-05-10"
    $response = Invoke-RestMethod -Method Delete -Uri $deleteDeploymentUri -Headers @{ Authorization="Bearer $accessToken" }
    Write-Host "Removed deployment: $deploymentname"
}
```

### 3. Deploy function to Azure through Azure CLI, VsCode, Azure DevOps or manually

### 4. Enable Managed Identity
[Managed Identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) are a feature that provides Azure services with an automatically managed identity in Azure AD. You can use the identity to authenticate to any service that supports Azure AD authentication, including Key Vault, without any credentials in your code.

This can be done through powershell or manually through the portal.

* Enable System assigned Managed Identity for Function
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/managedidentity.jpg)

### 5. Enable Managed Identity(aka.Service Principal) access to resource group using IAM
Navigate to `Resource group` in the Portal you want to manage with the powershell Azure function and select Access Control (IAM) from menu
* Enter in the name of the function as you have deployed it and click search.
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/searchfunctioniam.jpg)

* Click on function name and assign `contributor` role to it.
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/assigncontributorrolefunction.jpg)

This will ensure the function can perform CRUD operations against the resource group using the Azure management API.


### Example Response from Management REST API
```json
 {
     "value": [
         {
             "id": "/subscriptions/{subscription}/resourceGroups/{resourcegroupname}/providers/Microsoft.Resources/deployments/set-secrets-20191008-054045-bfab",
             "name": "set-secrets-20191008-054045-bfab",
             "type": "Microsoft.Resources/deployments",
             "properties": {
```

Thats it, this Azure timer based function will now run every night at 8:30pm and remove all deployments for my resource group.