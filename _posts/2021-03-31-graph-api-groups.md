---
title: "Manage Azure AD groups using the Graph API with PowerShell"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - azuread
excerpt: "The first of the public functions is managing Azure AD groups, this is a dependency for the Conditional Access policies, so seems a good place to start..."
---
Managing Azure AD groups is a dependency for the Conditional Access policies, as for (almost) every policy we'll be creating both an inclusion and exclusion group. I'll also be creating nested groups, such as dynamic groups such as "All Users" and "All Guests".

This creates a complete solution that can be deployed in an Azure Pipeline.

### Managing Azure AD groups
- [Get-WTAzureADGroup](#get-wtazureadgroup)
  - [What does this do?](#what-does-this-do)
- [Edit-WTAzureADGroup](#edit-wtazureadgroup)
  - [What does this do?](#what-does-this-do-1)
- [New-WTAzureADGroup](#new-wtazureadgroup)
  - [What does this do?](#what-does-this-do-2)
- [Remove-WTAzureADGroup](#remove-wtazureadgroup)
  - [What does this do?](#what-does-this-do-3)
- [Export-WTAzureADGroup](#export-wtazureadgroup)
  - [What does this do?](#what-does-this-do-4)

## Get-WTAzureADGroup
The first function is [Get-WTAzureADGroup][function-get], which you can access from my GitHub.

This gets the Azure AD groups, including all, specific IDs and specific group properties. This is needed in order to compare what's in Azure AD, to what may need to be updated or removed within the pipeline.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API and ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

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

# Or pipe specific IDs to get to the function, splat the hashtable containing the service principal
$IDs | Get-WTAzureADGroup @ServicePrincipal

# Or specify each parameter individually, including an access token previously obtained
Get-WTAzureADGroup -AccessToken $AccessToken -IDs $IDs
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

## Edit-WTAzureADGroup
The next function is [Edit-WTAzureADGroup][function-edit], which you can access from my GitHub.

This performs an edit (update) to the Azure AD groups. This allows changes such as the displayName to be altered for the group within the pipeline, if the config files have been updated with a new name.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

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
$AzureADGroup = [PSCustomObject]@{
  id          = $Id
  displayName = $DisplayName
}

# Pipe the Azure AD group to the function, specify an access token previously obtained
$AzureADGroup | Edit-WTAzureADGroup -AccessToken $AccessToken

# Or specify each parameter individually, including an access token previously obtained
Edit-WTAzureADGroup -AccessToken $AccessToken -AzureADGroup $AzureADGroup
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
        [PSCustomObject]$AzureADGroups
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

## New-WTAzureADGroup
The next function is [New-WTAzureADGroup][function-new], which you can access from my GitHub.

This creates new Azure AD groups, which will typically be imported from config files, or defined by another function, such as a Conditional Access inclusion/exclusion group in the pipeline.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API and ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\New-WTAzureADGroup.ps1

# Define Variables
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$DisplayName = "SVC-CA; Service Accounts"

# Create input object
$AzureADGroup = [PSCustomObject]@{
  displayName     = $DisplayName
  mailEnabled     = $false
  securityEnabled = $true
}

# Pipe the Azure AD group to the function, specify an access token previously obtained
$AzureADGroup | New-WTAzureADGroup -AccessToken $AccessToken

# Or specify each parameter individually, including an access token previously obtained
New-WTAzureADGroup -AccessToken $AccessToken -AzureADGroup $AzureADGroup
```

</details>

### What does this do?
- This sets specific variables, including the activity and the Graph Uri
  - As well as properties to remove from the input that would cause errors as they are readonly (or not recognised)
  - _The properties are cleaned up within the private patch function_
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- A mailNickName is a required unique parameter, if one is not specified in the Azure AD group, one is generated
  - This uses the [New-WTRandomString][blog-tagging] function to generate a string, which is concatenated with the Service variable
- The private function is then called

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function New-WTAzureADGroup {
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
            ValueFromPipeLine = $true,
            HelpMessage = "Specify the Azure AD Groups to create"
        )]
        [Alias('AzureADGroup')]
        [PSCustomObject]$AzureADGroups
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Private\Invoke-WTGraphPost.ps1",
                "Toolkit\Public\New-WTRandomString.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Activity = "Creating Azure AD groups"
            $Uri = "groups"
            $CleanUpProperties = (
                "id",
                "createdDateTime",
                "modifiedDateTime",
                "SideIndicator",
                "securityIdentifier",
                "createdByAppId",
                "renewedDateTime",
                "SVC",
                "REF",
                "ENV"
            )
            $Service = "AD"

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
                
                # If there are groups to deploy, for each
                if ($AzureADGroups) {

                    # Foreach group, check whether the required mailNickname exists, if not, generate this, append and return group
                    $AzureADGroups = foreach ($Group in $AzureADGroups){
                        if (!$Group.mailNickname){
                            $mailNickname = $null
                            $mailNickname = $Service + "-" + (New-WTRandomString -CharacterLength 24 -Alphanumeric)
                            $Group | Add-Member -MemberType NoteProperty -Name "mailNickname" -Value $mailNickname
                        }
                        
                        # Return group
                        $Group
                    }
                    
                    # Create groups
                    Invoke-WTGraphPost `
                        @Parameters `
                        -InputObject $AzureADGroups
                }
                else {
                    $ErrorMessage = "There are no groups to be created"
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

## Remove-WTAzureADGroup
The next function is [Remove-WTAzureADGroup][function-remove], which you can access from my GitHub.

This removes Azure AD groups by Id, and can be used in the pipeline for example, to remove Conditional Access inclusion/exclusion groups when a Conditional Access policy is deleted.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Remove-WTAzureADGroup.ps1

# Define Variables
$IDs = @("gkg23497-43gf-983s-5fg36-dsf234kafs24","hsw23497-hg5d-t59b-fd35k-dsf234kafs24")
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"

# Pipe specific IDs to get to the function, including an access token previously obtained
$IDs | Remove-WTAzureADGroup -AccessToken $AccessToken

# Or specify each parameter individually, including an access token previously obtained
Remove-WTAzureADGroup -AccessToken $AccessToken -IDs $IDs
```

</details>

### What does this do?
- This sets specific variables, including the activity and the Graph Uri
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- The private function is then called

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Remove-WTAzureADGroup {
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
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD Groups to remove, this must contain valid id(s)"
        )]
        [Alias("id", "GroupID", "GroupIDs")]
        [string[]]$IDs
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
            $Activity = "Removing Azure AD groups"
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
                    Uri               = $Uri
                    Activity          = $Activity
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }
                
                # If there are policies to be removed,  remove them
                if ($IDs) {
                    Invoke-WTGraphDelete `
                        @Parameters `
                        -IDs $IDs
                }
                else {
                    $ErrorMessage = "There are no Ids specified which are required to remove groups"
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

## Export-WTAzureADGroup
The last function is [Export-WTAzureADGroup][function-export], which you can access from my GitHub.

This exports the group config information from Azure AD to a JSON file. Within the pipeline this allows newly created or updated groups to have the updated config committed back to the repo for version control.

Examples below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API and ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Export-WTAzureADGroup.ps1

# Define Variables
$IDs = @("gkg23497-43gf-983s-5fg36-dsf234kafs24","hsw23497-hg5d-t59b-fd35k-dsf234kafs24")
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"
$Path = "GraphAPIConfig\AzureAD\Groups"

# Export all groups from Azure AD to the path specified, including an access token previously obtained
Export-WTAzureADGroup -AccessToken $AccessToken -Path $Path

# Or pipe specific IDs to the function to export to the path specified, including an access token previously obtained
$IDs | Export-WTAzureADGroup -AccessToken $AccessToken -Path $Path

# Or specify each parameter individually, including an access token previously obtained
Export-WTAzureADGroup -AccessToken $AccessToken -Path $Path -IDs $IDs
```

</details>

### What does this do?
- This sets specific variables, including optional properties to cleanup from the config prior to export
  - As well as the RegEx for unsupported characters for Windows, which are replaced with underscores
  - Plus the tags to be evaluated for the groups, as well as the property 'displayName' to tag
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- if no groups are specified, all groups are obtained unless there are specific ids provided
- A subdirectory is created if it does not exist to store the policies, if a tag exists, and they should be included
  - A JSON file is then created per group as required

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Export-WTAzureADGroup {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with AzureAD Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with AzureAD Graph permissions"
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
            HelpMessage = "The path where the JSON file(s) will be created"
        )]
        [string]$Path,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The file path where the JSON file will be created"
        )]
        [string]$FilePath,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude the cleanup operations of the groups to be exported"
        )]
        [switch]$ExcludeExportCleanup,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude tag processing of groups"
        )]
        [switch]$ExcludeTagEvaluation,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The AzureAD groups to get, this must contain valid id(s), when not specified, all groups are returned"
        )]
        [Alias("Group", "AzureADGroup")]
        [pscustomobject]$AzureADGroups,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The AzureAD groups to get, this must contain valid id(s), when not specified, all groups are returned"
        )]
        [Alias("id", "GroupID", "GroupIDs")]
        [string[]]$IDs,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The tag to use as the subdirectory to organise the export, default is 'SVC'"
        )]
        [Alias("Tag")]
        [string]$DirectoryTag = "SVC"
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Public\AzureAD\Groups\Get-WTAzureADGroup.ps1",
                "Toolkit\Public\Invoke-WTPropertyTagging.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }
            
            # Variables
            $CleanUpProperties = (
                "id",
                "createdDateTime",
                "modifiedDateTime"
            )
            $UnsupportedCharactersRegEx = '[\\\/:*?"<>|]'
            $Tags = @("SVC", "REF", "ENV")
            $PropertyToTag = "DisplayName"
            $Delimiter = "-"
            $Counter = 1
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            # If group object is provided, tag these
            if ($AzureADGroups) {

                # Evaluate the tags on the policies to be created, if not set to exclude
                if (!$ExcludeTagEvaluation) {
                    $AzureADGroups = Invoke-WTPropertyTagging -Tags $Tags -QueryResponse $AzureADGroups -PropertyToTag $PropertyToTag
                }
            }
            
            # If there are no groups to export, get groups based on specified parameters
            if (!$AzureADGroups) {
                
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
                    }
                    if ($ExcludeTagEvaluation) {
                        $Parameters.Add("ExcludeTagEvaluation", $true)
                    }
                    if ($ExcludePreviewFeatures) {
                        $Parameters.Add("ExcludePreviewFeatures", $true)
                    }
                    if ($IDs) {
                        $Parameters.Add("GroupIDs", $IDs)
                    }
                    
                    # Get all AzureAD groups
                    $AzureADGroups = Get-WTAzureADGroup @Parameters

                    if (!$AzureADGroups) {
                        $ErrorMessage = "Microsoft Graph did not return a valid response"
                        Write-Error $ErrorMessage
                        throw $ErrorMessage
                    }
                }
                else {
                    $ErrorMessage = "No access token specified, obtain an access token object from Get-WTGraphAccessToken"
                    Write-Error $ErrorMessage
                    throw $ErrorMessage
                }
            }

            # If there are groups
            if ($AzureADGroups) {

                # Sort and filter (if applicable) groups
                $AzureADGroups = $AzureADGroups | Sort-Object displayName
                if (!$ExcludeExportCleanup) {
                    $AzureADGroups | Foreach-object {
                            
                        # Cleanup properties for export
                        foreach ($Property in $CleanUpProperties) {
                            $_.PSObject.Properties.Remove("$Property")
                        }
                    }
                }

                # Export to JSON
                Write-Host "Exporting AzureAD Groups (Count: $($AzureADGroups.count))"

                # If a file path is specified, output all groups in one JSON formatted file
                if ($FilePath) {
                    $AzureADGroups | ConvertTo-Json -Depth 10 `
                    | Out-File -Force -FilePath $FilePath
                }
                else {
                    foreach ($Group in $AzureADGroups) {

                        # Remove characters not supported in Windows file names
                        $GroupDisplayName = $Group.displayname -replace $UnsupportedCharactersRegEx, "_"
                        
                        # Concatenate directory, if not set to exclude, else, append tag
                        if (!$ExcludeTagEvaluation) {
                            if ($Group.$DirectoryTag) {
                                $Directory = "$DirectoryTag$Delimiter$($Group.$DirectoryTag)"
                            }
                            else {
                                $Directory = "\"
                            }
                        }
                        else {
                            $Directory = "\"
                        }
                            
                        # If directory path does not exist for export, create it
                        $TestPath = Test-Path $Path\$Directory -PathType Container
                        if (!$TestPath) {
                            New-Item -Path $Path\$Directory -ItemType Directory | Out-Null
                        }

                        # Output current status
                        Write-Host "Processing Group $Counter with file name: $GroupDisplayName.json"
                            
                        # Output individual Group JSON file
                        $Group | ConvertTo-Json -Depth 10 `
                        | Out-File -Force -FilePath "$Path\$Directory\$GroupDisplayName.json"

                        # Increment counter
                        $Counter++
                    }
                }
            }
            else {
                $WarningMessage = "There are no AzureAD groups to export"
                Write-Warning $WarningMessage
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
[function-new]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/New-WTAzureADGroup.ps1
[function-remove]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Remove-WTAzureADGroup.ps1
[function-export]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Export-WTAzureADGroup.ps1
[blog-tagging]: /blog/generating-random-string