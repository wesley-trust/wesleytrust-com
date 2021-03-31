---
title: "Manage Azure AD groups Using the Graph API with PowerShell"
categories:
  - blog
tags:
  - graphapi
  - powershell
excerpt: "The first of the public functions is managing Azure AD groups, this is a dependency for the Conditional Access policies, so seems a good place to start..."
---
Managing Azure AD groups is a dependency for the Conditional Access policies, as for (almost) every policy we'll be creating both an inclusion and exclusion group, as well as adding members to the groups. I'll also be creating nested groups, such as dynamic ones including "All Users" and "All Guests".

This creates a complete solution that can be deployed in an Azure Pipeline. For the Azure AD groups, I'll be covering three main sections in this post, each section contains the public PowerShell functions to be called:

### Managing Azure AD groups
- Get-WTAzureADGroup
- Edit-WTAzureADGroup
- New-WTAzureADGroup
- Remove-WTAzureADGroup

### Exporting Azure AD groups config
- Export-WTAzureADGroup

### Managing Azure AD group relationships
- Get-WTAzureADGroupRelationship
- New-WTAzureADGroupRelationship

## Managing Azure AD groups

### Get-WTAzureADGroup
The first function is [Get-WTAzureADGroup][function-get], which you can access from my GitHub. This gets the Azure AD groups, including all, specific IDs and specific group properties. This is needed in order to compare what's in Azure AD, to what may need to be updated or removed within the pipeline.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Get-WTAzureADGroup.ps1

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
Get-WTAzureADGroup @ServicePrincipal

# Pipe specific IDs to get to the function, splat the hashtable containing the service principal
$IDs | Get-WTAzureADGroup @ServicePrincipal

# Or specify each parameter individually, including an access token previously obtained
Get-WTAzureADGroup -AccessToken $AccessToken -IDs $IDs
```

</details>

#### What does this do?
- This sets specific variables, including the activity, the tags to be evaluated against the groups, and the Graph Uri
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- A set of group properties are returned by default, but a select query to return just specific properties is included
  - I tidy this up removing spaces, as well as adding the requirement of id and displayName for tagging
- The private function is then called, with the query altered as appropriate depending on the parameters

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Get-WTAzureADGroup {
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
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD groups to get, this must contain valid id(s)"
        )]
        [Alias("id", "GroupID", "GroupIDs")]
        [string[]]$IDs,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Comma separated list of properties, 'id' is always selected, 'displayName' will also be selected if tagging is not excluded"
        )]
        [string]$Select
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
            $Activity = "Getting Azure AD groups"
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

                # If select is specified, a different query should be built
                if ($Select) {
                    
                    # Clean up input to remove remove any spaces
                    $Select = $Select.Replace(" ", "")

                    # Adding 'id' which is required for a result, 'displayName' is also added if tagging is not excluded, as it is a dependency
                    $Select = "id,$Select"
                    if (!$ExcludeTagEvaluation) {
                        $Select = "displayName,$Select"
                    }

                    # If there are Ids, get Azure AD group with selected properties only
                    if ($IDs) {
                        $QueryResponse = foreach ($Id in $IDs) {
                            Invoke-WTGraphGet @Parameters -Uri "$Uri/$($Id)?`$select=$Select"
                        }
                    }
                    else {
                        $WarningMessage = "A select query requires an ID to be specified for the group"
                        Write-Warning $WarningMessage
                    }
                }
                else {
                    if ($IDs) {
                        $Parameters.Add("IDs", $IDs)
                    }

                    # Get Azure AD groups with default properties
                    $QueryResponse = Invoke-WTGraphGet @Parameters -Uri $Uri
                }

                # Return response if one is returned
                if ($QueryResponse) {
                    $QueryResponse
                }
                else {
                    $WarningMessage = "No Azure AD groups exist in Azure AD, or with parameters specified"
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

### Edit-WTAzureADGroup
The next function is [Edit-WTAzureADGroup][function-edit], which you can access from my GitHub. This performs an edit (update) to the Azure AD groups. This allows changes such as the displayName to be altered for the group within the pipeline, if the config files have been updated with a new name.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Edit-WTAzureADGroup.ps1

# Define Variables
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$Id = "gve33497-hb48-983s-5fg36-dsf234kafs24"
$DisplayName = "SVC-CA; Updated displayName"

# Create input object
$AzureADGroup = [pscusomobject@{
  id          = $Id
  displayName = $DisplayName
}

# Pipe the AzureAD group to the function, specify an access token previously obtained
$AzureADGroup | Edit-WTAzureADGroup -AccessToken $AccessToken

# Or specify each parameter individually, including an access token previously obtained
Edit-WTAzureADGroup -AccessToken $AccessToken -AzureADGroup $AzureADGroup
```

</details>

#### What does this do?
- This sets specific variables, including the activity and the Graph Uri
  - As well as properties to remove from the input that would cause errors as they are readonly (or not recognised)
  - _The properties are cleaned up within the private patch function_
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- An id is required in order to update a group, but this check is done within the private patch function
- The private function is then called

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Edit-WTAzureADGroup {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Azure AD Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Azure AD Graph permissions"
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
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD groups to remove, a group must have a valid id"
        )]
        [Alias('AzureADGroup', 'GroupDefinition')]
        [pscustomobject]$AzureADGroups
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Private\Invoke-WTGraphPatch.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Activity = "Updating Azure AD Groups"
            $Uri = "groups"
            $CleanUpProperties = (
                "createdDateTime",
                "modifiedDateTime",
                "SideIndicator"
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
                    Uri               = $Uri
                    CleanUpProperties = $CleanUpProperties
                    Activity          = $Activity
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }

                # If there are groups to update, foreach group with a group id
                if ($AzureADGroups) {
                    
                    # Update groups
                    Invoke-WTGraphPatch `
                        @Parameters `
                        -InputObject $AzureADGroups
                }
                else {
                    $ErrorMessage = "There are no Azure AD groups to be updated"
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

[function-get]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Get-WTAzureADGroup.ps1
[function-edit]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Edit-WTAzureADGroup.ps1