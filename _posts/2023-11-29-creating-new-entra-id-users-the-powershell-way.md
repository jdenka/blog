---
layout: post
title: Creating new Entra ID users the PowerShell way
categories:
- Entra ID
- PowerShell
- Identity
tags:
- entra id
- powershell
- identity
pin: true
date: 2023-11-29 13:06 +0100
---
In the dynamic and evolving realm of Entra ID administration, the creation of new user accounts is not just a routine task. Leveraging the capabilities of the Microsoft Graph PowerShell SDK adds a layer of sophistication to this process, offering a robust interface for seamless and automated user management. In this article, we'll embark on a journey to create new users using the Microsoft Graph PowerShell SDK, injecting a touch of flair into the process.

## Adopting the Microsoft Graph PowerShell SDK
Before delving into the creation part of new users, it's essential to understand the importance of the Microsoft Graph PowerShell SDK. The SDK acts as a gateway between your PowerShell environment and Microsoft 365 services, providing a platform for automation. Whether you are dealing with user provisioning, collaboration settings or security configurations the Microsoft Graph PowerShell SDK simplifies complex tasks into manageable PowerShell cmdlets.

## Prerequisites
Before you begin, ensure that you have the Microsoft Graph PowerShell SDK installed. You can install it using the following command:

```powershell
# PowerShell 7.4 and above
Install-PSResource -Name Microsoft.Graph

# For older versions that don't use PSResourceGet
Install-Module -Name Microsoft.Graph
```
> To use managed identities with Microsoft Graph PowerShell SDK please ensure you have version 2.0 or above.
{: .prompt-info }

You will also need a user account or a workload identity with permissions to create users.

## Connecting to Microsoft Graph
The first step is to establish a connection to Microsoft Graph. Depending on your scenario, you can choose the appropriate authentication method, such as Managed Identities, certificate-based authentication, service principal, or interactive. In this example I'm using interactive login:

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All"
```

For more examples on how to connect to Microsoft Graph please take a look at this [Microsoft Learn article](https://learn.microsoft.com/en-us/powershell/microsoftgraph/authentication-commands?view=graph-powershell-1.0).

Always apply the minimum necessary privileges. If you're unsure about the capabilities of a particular scope in Microsoft Graph, refer to the [following resource](https://graphpermissions.merill.net/).

## Creating a new user
After establishing a connection to Microsoft Graph, we could simply use the New-MgUser but what would be the fun in that? 
So instead of navigating through New-MgUser I created my own function to create new users with ease. 

Allow me to present New-EntraUser. While it may not be flawless, it is acceptable for its purpose. The function requires two mandatory parameters, givenname and surname, along with the option to specify the domain for inclusion in the user's username. Although the domain is not obligatory, a validateset is employed, enabling the specification of custom domains if desired.

For example:
```powershell
New-EntraUser -GivenName Dennis -SurName Johansson -Domain cloudidentity.se
```
If the domain is specified, executing the command above will generate the user dennis.johansson@cloudidentity.se. In the absence of a specified domain, the script will determine and utilize the default domain associated with the connected tenant.

The function can be found [here](https://github.com/jdenka/New-EntraUser). If you have any suggestions or want to improve it please create a issue or a pull request on the repository. The link will also always contain the newest version. 

The code for those who want to read it here: 
Update: 2023-11-30 got a question on LinkedIn what happens if a user already exists. Was not thinking about that when I created this but that feature has now been added. Thank you [Tony](https://github.com/tonylanglet) for that question!

```powershell
function New-EntraUser {
    param (
        [Parameter(Mandatory = $true)]
        [string]$GivenName,
        [Parameter(Mandatory = $true)]
        [string]$SurName,
        [Parameter(Mandatory = $false)]
        [ValidateSet("cloudidentity.se", "replacethiswithyourdomains.abc", "yourcompanydomain.xyz", "thiswasjustfortesting.net")]
        $Domain

    )
    # Thanks to https://www.github.com/Ehmiiz for a awesome password module
    if (-not (Get-Module BinaryPasswordGenerator -ListAvailable)) {
        Write-Error "Module 'binarypasswordgenertor' is needed for this function to work" -ErrorAction Stop
    }

    # Thanks to https://www.github.com/mamapi for this function
    function Remove-DiacriticChars {
        param ([String]$srcString = [String]::Empty)
        $normalized = $srcString.Normalize( [Text.NormalizationForm]::FormD )
        $sb = new-object Text.StringBuilder
        $normalized.ToCharArray() | Foreach-Object { 
            if ( [Globalization.CharUnicodeInfo]::GetUnicodeCategory($_) -ne [Globalization.UnicodeCategory]::NonSpacingMark) {
                [void]$sb.Append($_)
            }
        }
        $sb.ToString()
    }
    # If $Domain is null or empty it will check the environments default domain and use it.
    if ([string]::IsNullOrEmpty($Domain)) {
        $pd = Get-MgDomain | Where-Object { $_.IsDefault -eq 'True' } | Select-Object id
        $domain = $pd.Id
    }

    $MailName = Remove-DiacriticChars -srcString "$($GivenName).$($SurName)"

    $Password = New-Password -Length 16
    $PasswordProfile = @{
        Password                      = "$($password)"
        ForceChangePasswordNextSignIn = $true
    }
    # Define user parameters
    $userParams = @{
        GivenName         = "$($GivenName)"
        Surname           = "$($SurName)"
        DisplayName       = "$($GivenName) $($SurName)"
        PasswordProfile   = $PasswordProfile
        AccountEnabled    = $true
        MailNickname      = $MailName
        UserPrincipalName = "$($MailName)@$($Domain)"
    }
    $counter = 0
    $checkupn = get-mguser -search "userprincipalname:$($userparams.UserPrincipalName)" -ConsistencyLevel eventual
    if ($null -eq $checkupn) {
        # Create a new user
        try {
            New-MgUser @userParams -ErrorAction Stop
            Write-Output "--------- `nCreated user with username: `n$($userParams.UserPrincipalName) `nWith password: `n$Password `n---------"
        }
        catch {
            Write-Error "Failed to create user" 
        }
    }
    else {
        try {
            while ($null -ne $checkupn) {
                $counter++
                $countername = $MailName+$counter
                $userParams = @{
                    GivenName         = "$($GivenName)"
                    Surname           = "$($SurName)"
                    DisplayName       = "$($GivenName) $($SurName)"
                    PasswordProfile   = $PasswordProfile
                    AccountEnabled    = $true
                    MailNickname      = $countername
                    UserPrincipalName = "$($countername)@$($Domain)"
                }
                $checkupn = get-mguser -search "userprincipalname:$($userparams.UserPrincipalName)" -ConsistencyLevel eventual
            }
            New-MgUser @userParams -ErrorAction Stop
            Write-Output "--------- `nCreated user with username: `n$($userParams.UserPrincipalName) `nWith password: `n$Password `n---------"
        }
        catch {
            Write-Error "Failed to create user"
        }
    }
}
```