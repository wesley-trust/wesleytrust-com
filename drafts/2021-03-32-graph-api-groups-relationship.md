---
title: "Update Azure AD group relationships using the Graph API with PowerShell"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - azuread
excerpt: "For Azure AD groups, owners or members of the group are defined as group 'relationships' this is a series of PowerShell functions to manage these..."
---
Managing Azure AD group members is a dependency for the Conditional Access policies, as for (almost) every policy we'll be adding members to groups. Such as the nested groups that will be added to the inclusion/exclusion groups that are created within the pipeline.

This creates a complete solution that can be deployed in an Azure Pipeline.

### Managing Azure AD group relationships
- [Get-WTAzureADGroupRelationship](#get-wtazureadgrouprelationship)
  - [What does this do?](#what-does-this-do)
- [New-WTAzureADGroupRelationship](#new-wtazureadgrouprelationship)
  - [What does this do?](#what-does-this-do-1)

## Get-WTAzureADGroupRelationship
The first function is [Get-WTAzureADGroupRelationship][function-get], which you can access from my GitHub.

This gets the Azure AD group members, including all, specific IDs and specific group properties. This is needed in order to compare what's in Azure AD, to what may need to be updated or removed within the pipeline.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Relationships\Get-WTAzureADGroupRelationship.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"
$IDs = @("gkg23497-43gf-983s-5fg36-dsf234kafs24","hsw23497-hg5d-t59b-fd35k-dsf234kafs24")
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"

# Create hashtable
$ServicePrincipal = @{
  ClientID     = $ClientID
  ClientSecret = $ClientSecret
  TenantDomain = $TenantDomain
}

# Get all groups, splat the hashtable containing the service principal to obtain an access token
Get-WTAzureADGroupRelationship @ServicePrincipal

# Pipe specific IDs to get to the function, splat the hashtable containing the service principal
$IDs | Get-WTAzureADGroupRelationship @ServicePrincipal

# Or specify each parameter individually, including an access token previously obtained
Get-WTAzureADGroupRelationship -AccessToken $AccessToken -IDs $IDs
```

</details>

### What does this do?
- This sets specific variables, including the activity, the tags to be evaluated against the groups, and the Graph Uri
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- A set of group properties are returned by default, but a select query to return just specific properties is included
  - I tidy this up removing spaces, as well as adding the requirement of id and displayName for tagging
- The private function is then called, with the query altered as appropriate depending on the parameters

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
                    $WarningMessage = "No group relationship exists in Azure AD for any of the group IDs specified"
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

## New-WTAzureADGroupRelationship
The next function is [New-WTAzureADGroup][function-new], which you can access from my GitHub.

This performs an edit (update) to the Azure AD groups. This allows changes such as the displayName to be altered for the group within the pipeline, if the config files have been updated with a new name.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Relationships\New-WTAzureADGroupRelationship.ps1

# Define Variables
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$Id = "gve33497-hb48-983s-5fg36-dsf234kafs24"
$DisplayName = "SVC-CA; Updated displayName"

# Create input object
$AzureADGroup = [PSCustomObject]@{
  id          = $Id
  displayName = $DisplayName
}

# Pipe the Azure AD group to the function, specify an access token previously obtained
$AzureADGroup | New-WTAzureADGroupRelationship -AccessToken $AccessToken

# Or specify each parameter individually, including an access token previously obtained
New-WTAzureADGroupRelationship -AccessToken $AccessToken -AzureADGroup $AzureADGroup
```

</details>

### What does this do?
- This sets specific variables, including the activity and the Graph Uri
  - As well as properties to remove from the input that would cause errors as they are readonly (or not recognised)
  - _The properties are cleaned up within the private patch function_
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- An id is required in order to update a group, but this check is done within the private patch function
- The private function is then called

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
            $CleanUpProperties = (
                "id",
                "createdDateTime",
                "modifiedDateTime"
            )

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
                    CleanUpProperties = $CleanUpProperties
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


[function-get]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Relationship/Get-WTAzureADGroupRelationship.ps1
[function-new]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Relationship/New-WTAzureADGroupRelationship.ps1