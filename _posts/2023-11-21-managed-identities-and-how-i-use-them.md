---
layout: post
title: Managed identities and how I use them
date: 2023-11-21 16:00 +0100
categories:
- Azure
- Identity
tags:
- azure
- identity
pin: true
---
## Managed Identities 

Today I want to highlight Managed Identities and how they can be used in different ways. 
There are two forms of Managed Identities and they are System assigned and User assigned, they both have use cases and can be utilized the same way. 

There can only be one System assigned identity on every resource in Azure that is tied to the resource lifetime. On the other hand a resource could have multiple User assigned identities and the User assigned identity can also be assigned to several resources making it more versatile.

> User assigned managed identities can't be assigned on Azure Arc enabled machines. But every Azure Arc machine gets a System assigned identity that can be used.
{: .prompt-info }

Since Im mostly using Azure and a Automation account I mainly use the System assigned identity and secure the automation account in every way possible. 

## Using the identity

So when we got this far and want to use our Managed identity to automate various tasks with we need to assign it permissions to do so and that is easy right? Not so much if you want to use it against Entra ID or any other service that requires Graph permissions instead of Azure permissions. 

So how do we do it? Since we can't go to the portal to assign permissions to a Managed Identity we have to script it. 

> In order to run the code below you need to either be a Global Administrator or have the role Privileged Role Administrator. It also requires the Microsoft.Graph Powershell module.
{: .prompt-info }

```powershell
$scope = @(
    "AppRoleAssignment.ReadWrite.All" # Required scope to connect to Graph to set permissions to the Managed Identity
)

Connect-MgGraph -Scopes $scope

$MIName = "" # Name of the Managed Identity

$Permissions = @( # Permissions to be set to the Managed Identity
    "User.Read.All"
    "Group.Read.All"
    "GroupMember.Read.All"
)

$GraphAppId = "00000003-0000-0000-c000-000000000000" # Application ID of the Graph API
$MI = Get-MgServicePrincipal -Filter "displayName eq '$MIName'" # Get the Managed Identity
$GraphSP = Get-MgServicePrincipal -Filter "appId eq '$GraphAppId'" # Get the Graph API Service Principal

$AppRole = $GraphSP.Approles | Where-Object { $_.Value -in $Permissions } # Get the AppRole object for the permissions

foreach($Role in $AppRole) {
    $ApproleAssignment = @{
        "PrincipalId" = $MI.Id
        "ResourceId" = $GraphSP.Id
        "AppRoleId" = $Role.Id
    }
    New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $ApproleAssignment.PrincipalId -AppRoleAssignment $ApproleAssignment -Verbose
}
```

Inside the $Permissions table it's easy to specify the permissions required for your desired automation. With these permissions I had a script looking up groups with a specific name, getting the group members and doing a lookup of the users with a set of properties that would be sent to a PowerBI report.

The limits of using a Managed identity is only set by the imagination, there are so much things that can be done and this is just a hint of what you can do. 