---
layout: blog
title: 800 limit on Resource groups and a way around it
---

As quoted from Microsoft around [resource group limits](https://docs.microsoft.com/en-us/azure/azure-subscription-service-limits#resource-group-limits)
> Maximum limit
Resources per resource group, per resource type	**800**	Some resource types can exceed the 800 limit. See Resources not limited to 800 instances per resource group.


### Resource group deployments
![Deployments](https://clouddna-au.github.io/assets/images/blog/2019-11-09/deployments.jpg)

When you get to your limit, you receive the following error when trying to deploy anything to the resource group:
> Creating the deployment 'Your-Deployment-Name-Goes-Here' would exceed the quota of '800'


## Steps to periodically remove deployments from the resource group
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
Ensures the right version of powershell module is loaded for function runtime.
```powershell
@{
    # For latest supported version, go to 'https://www.powershellgallery.com/packages/Az'. Uncomment the next line and replace the MAJOR_VERSION, e.g., 'Az' = '2.*'
     'Az' = '2.*'
}
```

### 3. Add function configuration settings
resource group

subscription


### 4. Add the code
This code will generate an access token to communicate with management api, retrieve all deployments for a resource group and then delete each deployment.
```powershell
# Input bindings are passed in via param block.
param($Timer)

Write-Host "Getting resource deployments"
$azContext = Get-AzContext

Write-Host "resource group: " $env:resourceGroup
Write-Host "subscriptionId: " $env:subscriptionId

# Get access token
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
    Write-Host "Removed deployment $deploymentname"
}
```

### 3. Deploy function to Azure through Azure CLI, VsCode or manually

### 4. Enable Managed Identity
This can be done through powershell or manually through the portal.

* Enable Managed Identity for Azure Function
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/managedidentity.jpg)

### 5. Enable Service Principal access to managed resources IAM
Navigate to `Resource group` you want to manage with the powershell Azure function and select Access Control (IAM) from menu
* Enter in the name of the function as you have deployed it and click search.
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/searchfunctioniam.jpg)

* Click on function name and assign `contributor` role toi it.
![Octocat](https://clouddna-au.github.io/assets/images/blog/2019-11-09/assigncontributorrolefunction.jpg)

This will ensure the function can perform CRUD operations against the resource group.


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