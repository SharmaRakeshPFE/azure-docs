---
title: Set up the Azure Monitor agent on Windows client devices
description: This article describes the instructions to install the agent on Windows 10, 11 client OS devices, configure data collection, manage and troubleshoot the agent.
ms.topic: conceptual
author: shseth
ms.author: shseth
ms.date: 10/18/2022
ms.custom: references_region
ms.reviewer: shseth

---

# Azure Monitor agent on Windows client devices
This article provides instructions and guidance for using the client installer for Azure Monitor Agent. It also explains how to leverage Data Collection Rules on Windows client devices.

Using the new client installer described here, you can now collect telemetry data from your Windows client devices in addition to servers and virtual machines.
Both the [extension](./azure-monitor-agent-manage.md#virtual-machine-extension-details) and this installer use Data Collection rules to configure the **same underlying agent**.

### Comparison with virtual machine extension
Here is a comparison between client installer and VM extension for Azure Monitor agent:

| Functional component | For VMs/servers via extension | For clients via installer|
|:---|:---|:---|
| Agent installation method | Via VM extension | Via client installer |
| Agent installed | Azure Monitor Agent | Same |
| Authentication | Using Managed Identity | Using AAD device token |
| Central configuration | Via Data collection rules | Same |
| Associating config rules to agents | DCRs associates directly to individual VM resources | DCRs associate to Monitored Object (MO), which maps to all devices within the AAD tenant |
| Data upload to Log Analytics	| Via Log Analytics endpoints | Same |
| Feature support | All features documented [here](./azure-monitor-agent-overview.md) | Features dependent on AMA agent extension that don't require additional extensions. This includes support for Sentinel Windows Event filtering |
| [Networking options](./azure-monitor-agent-overview.md#networking) | Proxy support, Private link support | Proxy support only |



## Supported device types

| Device type | Supported? | Installation method | Additional information |
|:---|:---|:---|:---|
| Windows 10, 11 desktops, workstations | Yes | Client installer | Installs the agent using a Windows MSI installer |
| Windows 10, 11 laptops | Yes |  Client installer | Installs the agent using a Windows MSI installer. The installs works on laptops but the agent is **not optimized yet** for battery, network consumption |
| Virtual machines, scale sets | No | [Virtual machine extension](./azure-monitor-agent-manage.md#virtual-machine-extension-details) | Installs the agent using Azure extension framework |
| On-premises servers | No | [Virtual machine extension](./azure-monitor-agent-manage.md#virtual-machine-extension-details) (with Azure Arc agent) | Installs the agent using Azure extension framework, provided for on-premises by installing Arc agent |


## Prerequisites
1. The machine must be running Windows client OS version 10 RS4 or higher.
2. To download the installer, the machine should have [C++ Redistributable version 2015)](/cpp/windows/latest-supported-vc-redist?view=msvc-170&preserve-view=true) or higher
3. The machine must be domain joined to an Azure AD tenant (AADj or Hybrid AADj machines), which enables the agent to fetch Azure AD device tokens used to authenticate and fetch data collection rules from Azure.
4. You may need tenant admin permissions on the Azure AD tenant.
5. The device must have access to the following HTTPS endpoints:
	-	global.handler.control.monitor.azure.com
	-	`<virtual-machine-region-name>`.handler.control.monitor.azure.com (example: westus.handler.control.azure.com)
	-	`<log-analytics-workspace-id>`.ods.opinsights.azure.com (example: 12345a01-b1cd-1234-e1f2-1234567g8h99.ods.opinsights.azure.com)
    (If using private links on the agent, you must also add the [data collection endpoints](../essentials/data-collection-endpoint-overview.md#components-of-a-data-collection-endpoint))
6. A data collection rule you want to associate with the devices. If it doesn't exist already, [create a data collection rule](./data-collection-rule-azure-monitor-agent.md#create-a-data-collection-rule). **Do not associate the rule to any resources yet**.

## Install the agent
1. Download the Windows MSI installer for the agent using [this link](https://go.microsoft.com/fwlink/?linkid=2192409). You can also download it from **Monitor** > **Data Collection Rules** > **Create** experience on Azure portal (shown below):
	[![Diagram shows download agent link on Azure portal.](media/azure-monitor-agent-windows-client/azure-monitor-agent-client-installer-portal.png)](media/azure-monitor-agent-windows-client/azure-monitor-agent-client-installer-portal-focus.png#lightbox)
2. Open an elevated admin command prompt window and update path to the location where you downloaded the installer.
3. To install with **default settings**, run the following command:
	```cli
	msiexec /i AzureMonitorAgentClientSetup.msi /qn
	```
4. To install with custom file paths or [network proxy settings](./azure-monitor-agent-overview.md#proxy-configuration), use the command below with the values from the following table:
	```cli
	msiexec /i AzureMonitorAgentClientSetup.msi /qn DATASTOREDIR="C:\example\folder"
	```

	| Parameter | Description |
	|:---|:---|
	| INSTALLDIR | Directory path where the agent binaries are installed |
	| DATASTOREDIR | Directory path where the agent stores its operational logs and data |
	| PROXYUSE | Must be set to "true" to use proxy |
	| PROXYADDRESS | Set to Proxy Address. PROXYUSE must be set to "true" to be correctly applied |
	| PROXYUSEAUTH | Set to "true" if proxy requires authentication |
	| PROXYUSERNAME | Set to Proxy username. PROXYUSE and PROXYUSEAUTH must be set to "true" |
	| PROXYPASSWORD | Set to Proxy password. PROXYUSE and PROXYUSEAUTH must be set to "true" |

5. Verify successful installation:
	- Open **Control Panel** -> **Programs and Features** OR **Settings** -> **Apps** -> **Apps & Features** and ensure you see ‘Azure Monitor Agent’ listed
	- Open **Services** and confirm ‘Azure Monitor Agent’ is listed and shows as **Running**.
6. Proceed to create the monitored object that you'll associate data collection rules to, for the agent to actually start operating.

> [!NOTE]
>  The agent installed with the client installer currently doesn't support updating configuration once it is installed. Uninstall and reinstall AMA to update its configuration.


## Create and associate a 'Monitored Object'
You need to create a 'Monitored Object' (MO) that creates a representation for the Azure AD tenant within Azure Resource Manager (ARM). This ARM entity is what Data Collection Rules are then associated with.
Currently this association is only **limited** to the Azure AD tenant scope, which means configuration applied to the tenant will be applied to all devices that are part of the tenant and running the agent.
The image below demonstrates how this works:

![Diagram shows monitored object purpose and association.](media/azure-monitor-agent-windows-client/azure-monitor-agent-monitored-object.png)

Then, proceed with the instructions below to create and associate them to a Monitored Object, using REST APIs or PowerShell commands.

### Permissions required
Since MO is a tenant level resource, the scope of the permission would be higher than a subscription scope. Therefore, an Azure tenant admin may be needed to perform this step. [Follow these steps to elevate Azure AD Tenant Admin as Azure Tenant Admin](../../role-based-access-control/elevate-access-global-admin.md). It will give the Azure AD admin 'owner' permissions at the root scope. This is needed for all methods described below in this section.

### Using REST APIs

#### 1. Assign ‘Monitored Object Contributor’ role to the operator

This step grants the ability to create and link a monitored object to a user.

**Request URI**
```HTTP
PUT https://management.azure.com/providers/microsoft.insights/providers/microsoft.authorization/roleassignments/{roleAssignmentGUID}?api-version=2021-04-01-preview
```
**URI Parameters**

| Name | In | Type | Description |
|:---|:---|:---|:---|:---|
| `roleAssignmentGUID` | path | string | Provide any valid guid (you can generate one using https://guidgenerator.com/) |

**Headers**
- Authorization: ARM Bearer Token (using ‘Get-AzAccessToken’ or other method)
- Content-Type: Application/json

**Request Body**
```JSON
{
	"properties":
	{
		"roleDefinitionId":"/providers/Microsoft.Authorization/roleDefinitions/56be40e24db14ccf93c37e44c597135b",
		"principalId":"aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
	}
}
```

**Body parameters**

| Name | Description |
|:---|:---|
| roleDefinitionId | Fixed value: Role definition ID of the 'Monitored Objects Contributor' role: `/providers/Microsoft.Authorization/roleDefinitions/56be40e24db14ccf93c37e44c597135b` |
| principalId | Provide the `Object Id` of the identity of the user to which the role needs to be assigned. It may be the user who elevated at the beginning of step 1, or another user who will perform later steps. |

After this step is complete, **reauthenticate** your session and **reacquire** your ARM bearer token.

#### 2. Create Monitored Object
This step creates the Monitored Object for the Azure AD Tenant scope. It will be used to represent client devices that are signed with that Azure AD Tenant identity.

**Permissions required**: Anyone who has 'Monitored Object Contributor' at an appropriate scope can perform this operation, as assigned in step 1.

**Request URI**
```HTTP
PUT https://management.azure.com/providers/Microsoft.Insights/monitoredObjects/{AADTenantId}?api-version=2021-09-01-preview
```
**URI Parameters**

| Name | In | Type | Description |
|:---|:---|:---|:---|:---|
| `AADTenantId` | path | string | ID of the Azure AD tenant that the device(s) belong to. The MO will be created with the same ID |

**Headers**
- Authorization: ARM Bearer Token
- Content-Type: Application/json

**Request Body**
```JSON
{
    "properties":
	{
        "location":"eastus"
    }
}
```
**Body parameters**

| Name | Description |
|:---|:---|
| `location` | The Azure region where the MO object would be stored. It should be the **same region** where you created the Data Collection Rule. This is the location of the region from where agent communications would happen. |


#### 3. Associate DCR to Monitored Object
Now we associate the Data Collection Rules (DCR) to the Monitored Object by creating Data Collection Rule Associations. 

**Permissions required**: Anyone who has ‘Monitored Object Contributor’ at an appropriate scope can perform this operation, as assigned in step 1.

**Request URI**
```HTTP
PUT https://management.azure.com/{MOResourceId}/providers/microsoft.insights/datacollectionruleassociations/assoc?api-version=2021-04-01
```
**Sample Request URI**
```HTTP
PUT https://management.azure.com/providers/Microsoft.Insights/monitoredObjects/{AADTenantId}/providers/microsoft.insights/datacollectionruleassociations/assoc?api-version=2021-04-01
```

**URI Parameters**

| Name | In | Type | Description |
|:---|:---|:---|:---|:---|
| ``MOResourceId` | path | string | Full resource ID of the MO created in step 2. Example: 'providers/Microsoft.Insights/monitoredObjects/{AADTenantId}' |

**Headers**
- Authorization: ARM Bearer Token
- Content-Type: Application/json

**Request Body**
```JSON
{
	"properties":
	{
		"dataCollectionRuleId": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Insights/dataCollectionRules/{DCRName}"
	}
}
```
**Body parameters**

| Name | Description |
|:---|:---|
| `dataCollectionRuleID` | The resource ID of an existing Data Collection Rule that you created in the **same region** as the Monitored Object. |


### Using PowerShell
```PowerShell
$TenantID = "xxxxxxxxx-xxxx-xxx"  #Your Tenant ID
$SubscriptionID = "xxxxxx-xxxx-xxxxx" #Your Subscription ID
$ResourceGroup = "rg-yourResourseGroup" #Your resroucegroup
$DCRName = "CollectWindowsOSlogs" #Your Data collection rule name

Connect-AzAccount -Tenant $TenantID

#Select the subscription
Select-AzSubscription -SubscriptionId $SubscriptionID

#Grant Access to User at root scope "/"
$user = Get-AzADUser -UserPrincipalName (Get-AzContext).Account

New-AzRoleAssignment -Scope '/' -RoleDefinitionName 'Owner' -ObjectId $user.Id

#Create Auth Token
$auth = Get-AzAccessToken

$AuthenticationHeader = @{
    "Content-Type" = "application/json"
    "Authorization" = "Bearer " + $auth.Token
    }


#1. Assign ‘Monitored Object Contributor’ Role to the operator
$newguid = (New-Guid).Guid
$UserObjectID = $user.Id

$body = @"
{
            "properties": {
                "roleDefinitionId":"/providers/Microsoft.Authorization/roleDefinitions/56be40e24db14ccf93c37e44c597135b",
                "principalId": `"$UserObjectID`"
        }
}
"@

$request = "https://management.azure.com/providers/microsoft.insights/providers/microsoft.authorization/roleassignments/$newguid`?api-version=2021-04-01-preview"


Invoke-RestMethod -Uri $request -Headers $AuthenticationHeader -Method PUT -Body $body


##########################

#2. Create Monitored Object

# "location" property value under the "body" section should be the Azure region where the MO object would be stored. It should be the "same region" where you created the Data Collection Rule. This is the location of the region from where agent communications would happen.

$request = "https://management.azure.com/providers/Microsoft.Insights/monitoredObjects/$TenantID`?api-version=2021-09-01-preview"
$body = @'
{
    "properties":{
        "location":"eastus"
    }
}
'@

$Respond = Invoke-RestMethod -Uri $request -Headers $AuthenticationHeader -Method PUT -Body $body -Verbose
$RespondID = $Respond.id

##########################

#3. Associate DCR to Monitored Object

$request = "https://management.azure.com$RespondId/providers/microsoft.insights/datacollectionruleassociations/assoc?api-version=2021-04-01"
$body = @"
        {
            "properties": {
                "dataCollectionRuleId": "/subscriptions/$SubscriptionID/resourceGroups/$ResourceGroup/providers/Microsoft.Insights/dataCollectionRules/$DCRName"
            }
        }

"@

Invoke-RestMethod -Uri $request -Headers $AuthenticationHeader -Method PUT -Body $body
```



## Verify successful setup
Check the ‘Heartbeat’ table (and other tables you configured in the rules) in the Log Analytics workspace that you specified as a destination in the data collection rule(s).
The `SourceComputerId`, `Computer`, `ComputerIP` columns should all reflect the client device information respectively, and the `Category` column should say 'Azure Monitor Agent'. See example below:

[![Diagram shows agent heartbeat logs on Azure portal.](media/azure-monitor-agent-windows-client/azure-monitor-agent-heartbeat-logs.png)](media/azure-monitor-agent-windows-client/azure-monitor-agent-heartbeat-logs.png)


## Manage the agent

### Check the agent version
You can use any of the following options to check the installed version of the agent:
- Open **Control Panel** > **Programs and Features** > **Azure Monitor Agent** and see the 'Version' listed
- Open **Settings** > **Apps** > **Apps and Features** > **Azure Monitor Agent** and see the 'Version' listed

### Uninstall the agent
You can use any of the following options to check the installed version of the agent:
- Open **Control Panel** > **Programs and Features** > **Azure Monitor Agent** and click 'Uninstall'
- Open **Settings** > **Apps** > **Apps and Features** > **Azure Monitor Agent** and click 'Uninstall'

If you face issues during 'Uninstall', refer to [troubleshooting guidance](#troubleshoot) below

### Update the agent
In order to update the version, install the new version you wish to update to.


## Troubleshoot
### View agent diagnostic logs
1. Rerun the installation with logging turned on and specify the log file name:
	`Msiexec /I AzureMonitorAgentClientSetup.msi /L*V <log file name>`
2. Runtime logs are collected automatically either at the default location `C:\Resources\Azure Monitor Agent\` or at the file path mentioned during installation.
	- If you can't locate the path, the exact location can be found on the registry as `AMADataRootDirPath` on `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\AzureMonitorAgent`.
3. The 'ServiceLogs' folder contains log from AMA Windows Service, which launches and manages AMA processes
4. 'AzureMonitorAgent.MonitoringDataStore' contains data/logs from AMA processes.

### Common issues

#### Missing DLL
- Error message: "There's a problem with this Windows Installer package. A DLL required for this installer to complete could not be run. …"
- Ensure you have installed [C++ Redistributable (>2015)](/cpp/windows/latest-supported-vc-redist?view=msvc-170&preserve-view=true) before installing AMA:

#### Silent install from command prompt fails
Make sure to start the installer on administrator command prompt. Silent install can only be initiated from the administrator command prompt.

#### Uninstallation fails due to the uninstaller being unable to stop the service
- If There's an option to try again, do try it again
- If retry from uninstaller doesn't work, cancel the uninstall and stop Azure Monitor Agent service from Services (Desktop Application)
- Retry uninstall

#### Force uninstall manually when uninstaller doesn't work
- Stop Azure Monitor Agent service. Then try uninstalling again. If it fails, then proceed with the following steps
- Delete AMA service with "sc delete AzureMonitorAgent" from admin cmd
- Download [this tool](https://support.microsoft.com/topic/fix-problems-that-block-programs-from-being-installed-or-removed-cca7d1b6-65a9-3d98-426b-e9f927e1eb4d) and uninstall AMA
- Delete AMA binaries. They're stored in `Program Files\Azure Monitor Agent` by default
- Delete AMA data/logs. They're stored in `C:\Resources\Azure Monitor Agent` by default
- Open Registry. Check `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Azure Monitor Agent`. If it exists, delete the key.


## Questions and feedback
Take this [quick survey](https://forms.microsoft.com/r/CBhWuT1rmM) or share your feedback/questions regarding the client installer.
