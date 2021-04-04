---
title: "Manage Azure AD group relationships via the Graph API & PowerShell"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - group-relationships
  - group-members
  - group-owners
  - azuread
excerpt: "For Azure AD groups, owners or members of the group are defined as group 'relationships' this is a series of PowerShell functions to manage these..."
---
Managing Azure AD group members is a dependency for the Conditional Access policies, as for (almost) every policy we'll be adding members to groups. Such as the nested groups that will be added to the inclusion/exclusion groups created within the pipeline.

This creates a complete solution that can be deployed in an Azure Pipeline.

### Managing Azure AD group relationships
- [Get Azure AD group relationships](#get-azure-ad-group-relationships)
- [Create Azure AD group relationships](#create-azure-ad-group-relationships)
- [Remove Azure AD group relationships](#remove-azure-ad-group-relationships)

## Get Azure AD group relationships
The first function is [Get-WTAzureADGroupRelationship][function-get], which you can access from my GitHub.

This gets the Azure AD group owners, memberOf or members, for the specific group IDs specified.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API and ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Relationship\Get-WTAzureADGroupRelationship.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"
$GroupIDs = @("gkg23497-43gf-983s-5fg36-dsf234kafs24","hsw23497-hg5d-t59b-fd35k-dsf234kafs24")
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$Relationship = "members"

# Create hashtable
$Parameters = @{
  ClientID     = $ClientID
  ClientSecret = $ClientSecret
  TenantDomain = $TenantDomain
  GroupIDs     = $GroupIDs
  Relationship = $Relationship
}

# Get the members for the specific group, splat the parameters (including the service principal to obtain an access token)
Get-WTAzureADGroupRelationship @Parameters

# Or pipe specific group IDs to get the members, including an access token previously obtained
$GroupIDs | Get-WTAzureADGroupRelationship -AccessToken $AccessToken -Relationship $Relationship

# Or specify each parameter individually, including an access token previously obtained
Get-WTAzureADGroupRelationship -AccessToken $AccessToken -GroupIDs $GroupIDs -Relationship $Relationship
```

</details>

### What does this do? <!-- omit in toc -->
- This sets specific variables, including the activity, the tags to be evaluated in the relationships, and the Graph Uri
  - A group relationship could consist of owners, memberOf or members which is validated
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- The private function is then called, with the query altered as appropriate depending on the parameters

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Get-WTAzureADGroupRelationship {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Azure AD group Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Azure AD group Graph permissions"
        )]
        [string]$ClientSecret,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The initial domain (onmicrosoft.com) of the tenant"
        )]
        [string]$TenantDomain,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The access token, obtained from executing Get-WTGraphAccessToken"
        )]
        [string]$AccessToken,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude tag processing of groups"
        )]
        [switch]$ExcludeTagEvaluation,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD group to get the members of, this must contain valid id(s)"
        )]
        [Alias("id", "GroupID", "GroupIDs")]
        [string[]]$IDs,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The group relationship to return, such as group members, owners or groups this group is a member of"
        )]
        [ValidateSet("members", "owners", "memberOf")]
        [string]$Relationship
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Private\Invoke-WTGraphGet.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Activity = "Getting Azure AD group $Relationship"
            $Uri = "groups"
            $Tags = @("SVC", "REF", "ENV")

        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            # If there is no access token, obtain one
            if (!$AccessToken) {
                $AccessToken = Get-WTGraphAccessToken `
                    -ClientID $ClientID `
                    -ClientSecret $ClientSecret `
                    -TenantDomain $TenantDomain
            }
            if ($AccessToken) {

                # Build Parameters
                $Parameters = @{
                    AccessToken = $AccessToken
                    Activity    = $Activity
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }
                if (!$ExcludeTagEvaluation) {
                    $Parameters.Add("Tags", $Tags)
                }

                # Get Azure AD group relationship
                $QueryResponse = foreach ($Id in $IDs) {
                    Invoke-WTGraphGet @Parameters -Uri "$Uri/$Id/$Relationship"
                }

                # Return response if one is returned
                if ($QueryResponse) {
                    $QueryResponse
                }
                else {
                    $WarningMessage = "No group $Relationship exist in Azure AD for any of the group IDs specified"
                    Write-Warning $WarningMessage
                }
            }
            else {
                $ErrorMessage = "No access token specified, obtain an access token object from Get-WTGraphAccessToken"
                Write-Error $ErrorMessage
                throw $ErrorMessage
            }
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    End {
        try {
            
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
}
```

</details>

## Create Azure AD group relationships
The next function is [New-WTAzureADGroupRelationship][function-new], which you can access from my GitHub.

This creates new Azure AD group relationships, which can be either owners or members, this is used within the pipeline to add members to the Conditional Access inclusion/exclusion groups created in the pipeline.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Relationship\New-WTAzureADGroupRelationship.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"
$GroupID = "gb5d3497-78jb-983s-hb5s6-gbv334kafs24"
$RelationshipIDs = @("gkg23497-43gf-983s-5fg36-dsf234kafs24","hsw23497-hg5d-t59b-fd35k-dsf234kafs24")
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$Relationship = "members"

# Create hashtable
$Parameters = @{
  ClientID          = $ClientID
  ClientSecret      = $ClientSecret
  TenantDomain      = $TenantDomain
  GroupID           = $GroupID
  RelationshipIDs   = $RelationshipIDs
  Relationship      = $Relationship
}

# Add new relationships to the specified group, splat the parameters (including the service principal to obtain an access token)
New-WTAzureADGroupRelationship @Parameters

# Or pipe specific relationship IDs to create the association with the group, including an access token previously obtained
$RelationshipIDs | New-WTAzureADGroupRelationship -AccessToken $AccessToken -GroupID $GroupID -Relationship $Relationship

# Or specify each parameter individually, including an access token previously obtained
New-WTAzureADGroupRelationship -AccessToken $AccessToken -GroupID $GroupID -RelationshipIDs $RelationshipIDs -Relationship $Relationship
```

</details>

### What does this do? <!-- omit in toc -->
- This sets specific variables, including the activity and the Graph Uri
  - A group relationship could consist of owners or members which is validated
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- A group ID is required to add a relationship, this forms part of the Uri, the request must in a specific format
- To add a relationship, an object must be created in a specific format, this is done for each relationship ID
- The private function is then called with the collection of object relationships to be added to the group

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function New-WTAzureADGroupRelationship {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Azure AD group Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Azure AD group Graph permissions"
        )]
        [string]$ClientSecret,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The initial domain (onmicrosoft.com) of the tenant"
        )]
        [string]$TenantDomain,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The access token, obtained from executing Get-WTGraphAccessToken"
        )]
        [string]$AccessToken,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD group to add the members or owners to, this must contain valid id(s)"
        )]
        [Alias("GroupID")]
        [string]$ID,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The group relationship to add, such as group members or owners"
        )]
        [ValidateSet("members", "owners")]
        [string]$Relationship,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The relationship ids of the objects to add to the group"
        )]
        [Alias('RelationshipID', 'GroupRelationshipID', 'GroupRelationshipIDs')]
        [string[]]$RelationshipIDs
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Private\Invoke-WTGraphPost.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Activity = "Adding Azure AD group $Relationship"
            $Uri = "groups"
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            # If there is no access token, obtain one
            if (!$AccessToken) {
                $AccessToken = Get-WTGraphAccessToken `
                    -ClientID $ClientID `
                    -ClientSecret $ClientSecret `
                    -TenantDomain $TenantDomain
            }
            if ($AccessToken) {

                # Build Parameters
                $Parameters = @{
                    AccessToken       = $AccessToken
                    Uri               = "$Uri/$Id/$Relationship/`$ref"
                    Activity          = $Activity
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }

                # If there are IDs, for each, create an object with the ID
                if ($RelationshipIDs) {
                    $RelationshipObject = foreach ($RelationshipId in $RelationshipIDs) {
                        [pscustomobject]@{
                            "@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$RelationshipId"
                        }
                    }

                    # Add group relationship
                    Invoke-WTGraphPost `
                        @Parameters `
                        -InputObject $RelationshipObject
                }
                else {
                    $ErrorMessage = "There are no group $Relationship to be added"
                    Write-Error $ErrorMessage
                }
            }
            else {
                $ErrorMessage = "No access token specified, obtain an access token object from Get-WTGraphAccessToken"
                Write-Error $ErrorMessage
                throw $ErrorMessage
            }
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    End {
        try {
            
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
}
```

</details>

## Remove Azure AD group relationships
The last function is [Remove-WTAzureADGroupRelationship][function-remove], which you can access from my GitHub.

This removes Azure AD group relationships, which can be either owners or members.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Relationship\Remove-WTAzureADGroupRelationship.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"
$GroupID = "gb5d3497-78jb-983s-hb5s6-gbv334kafs24"
$RelationshipIDs = @("gkg23497-43gf-983s-5fg36-dsf234kafs24","hsw23497-hg5d-t59b-fd35k-dsf234kafs24")
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$Relationship = "members"

# Create hashtable
$Parameters = @{
  ClientID          = $ClientID
  ClientSecret      = $ClientSecret
  TenantDomain      = $TenantDomain
  GroupID           = $GroupID
  RelationshipIDs   = $RelationshipIDs
  Relationship      = $Relationship
}

# Remove relationships from the specified group, splat the parameters (including the service principal to obtain an access token)
Remove-WTAzureADGroupRelationship @Parameters

# Or pipe specific relationship IDs to remove the association with the group, including an access token previously obtained
$RelationshipIDs | Remove-WTAzureADGroupRelationship -AccessToken $AccessToken -GroupID $GroupID -Relationship $Relationship

# Or specify each parameter individually, including an access token previously obtained
Remove-WTAzureADGroupRelationship -AccessToken $AccessToken -GroupID $GroupID -RelationshipIDs $RelationshipIDs -Relationship $Relationship
```

</details>

### What does this do? <!-- omit in toc -->
- This sets specific variables, including the activity and the Graph Uri
  - A group relationship could consist of owners or members which is validated
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- A group ID is required to remove a relationship, the relationship IDs are also specified as part of the Uri
- The private function is then called for each ID to be removed from the group

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Remove-WTAzureADGroupRelationship {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Azure AD group Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Azure AD group Graph permissions"
        )]
        [string]$ClientSecret,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The initial domain (onmicrosoft.com) of the tenant"
        )]
        [string]$TenantDomain,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The access token, obtained from executing Get-WTGraphAccessToken"
        )]
        [string]$AccessToken,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD group to remove the members or owners from, this must contain valid id(s)"
        )]
        [Alias("GroupID")]
        [string]$ID,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The group relationship to remove, such as group members or owners"
        )]
        [ValidateSet("members", "owners")]
        [string]$Relationship,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The relationship ids of the objects to remove from the group"
        )]
        [Alias('RelationshipID', 'GroupRelationshipID', 'GroupRelationshipIDs')]
        [string[]]$RelationshipIDs
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Private\Invoke-WTGraphDelete.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Activity = "Removing Azure AD group $Relationship"
            $Uri = "groups"
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            # If there is no access token, obtain one
            if (!$AccessToken) {
                $AccessToken = Get-WTGraphAccessToken `
                    -ClientID $ClientID `
                    -ClientSecret $ClientSecret `
                    -TenantDomain $TenantDomain
            }
            if ($AccessToken) {

                # Build Parameters
                $Parameters = @{
                    AccessToken       = $AccessToken
                    Activity          = $Activity
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }

                # If there are IDs, for each, remove the group relationship
                if ($RelationshipIDs) {
                    foreach ($RelationshipId in $RelationshipIDs) {
                        
                        # Remove group relationship
                        Invoke-WTGraphDelete `
                            @Parameters `
                            -Uri "$Uri/$Id/$Relationship/$RelationshipId/`$ref"
                    }
                }
                else {
                    $ErrorMessage = "There are no group $Relationship to be removed"
                    Write-Error $ErrorMessage
                }
            }
            else {
                $ErrorMessage = "No access token specified, obtain an access token object from Get-WTGraphAccessToken"
                Write-Error $ErrorMessage
                throw $ErrorMessage
            }
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    End {
        try {
            
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
}
```

</details>

[function-get]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Relationship/Get-WTAzureADGroupRelationship.ps1
[function-new]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Relationship/New-WTAzureADGroupRelationship.ps1
[function-remove]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Relationship/Remove-WTAzureADGroupRelationship.ps1