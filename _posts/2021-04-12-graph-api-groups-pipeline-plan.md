---
title: "Azure AD groups in a CI/CD Pipeline, Stage 2: Plan & Evaluate"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - azuread
  - pipeline
  - plan
  - evaluate
excerpt: "This post covers the second stage in the pipeline which will be used to automate creating, updating and removing Azure AD groups..."
---
The PowerShell I'm writing, which in part mimics the stages that Terraform goes through when deploying Azure resources, adds some "smarts" to the Pipeline.

This is the second stage, in the three stage pipeline for managing Azure AD groups:
- [Import & Validate][validate-post]
- Plan & Evaluate
- Apply & Deploy

This post covers the YAML and PowerShell involved in the second stage which creates a plan of actions (if any), after evaluating the validated group input against Azure AD. The PowerShell can also be called directly.

|  Current Import & Validate Status  |   Current Plan & Evaluate Status   |
|:----------------------------------:|:----------------------------------:|
|[![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Groups/SVC-AD%3BENV-P%3B%20Groups?branchName=main&stageName=Validate&jobName=Import)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=9&branchName=main)|[![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Groups/SVC-AD%3BENV-P%3B%20Groups?branchName=main&stageName=Plan&jobName=Evaluate)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=9&branchName=main)|

## Invoke Plan Azure AD group
This function is [Invoke-WTPlanAzureADGroup][function-plan], which you can access from my GitHub.

Within the pipeline, this imports the validated JSON artifact of groups (should they exist), which is passed to the function via a parameter. This then creates a plan of what should be created, updated or removed (as appropriate).

Outputting a JSON plan file (as appropriate) as a pipeline artifact for the next stage in the pipeline.

### Pipeline YAML example below:
_Triggered on a change to the [GraphAPIConfig template repo in GitHub][github-repo]_

As Azure AD groups can be created in multiple ways, and by multiple applications, having the config repo being the source of authority didn't seem appropriate, so by default, groups are not removed if they exist in Azure AD and do not exist in the config repo. In the future I might consider a "state" file, similar to Terraform to keep track of this.

_Azure Pipelines automatically downloads artifacts created in the previous stage_

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```yaml
- stage: Plan
  pool:
    vmImage: 'windows-latest'
  dependsOn: Validate
  condition: and(succeeded(), eq(dependencies.Validate.outputs['Import.InvokeWTValidateAzureADGroup.ShouldRun'], 'true'))
  jobs:
  - job: Evaluate
    continueOnError: false
    steps:
    - task: DownloadPipelineArtifact@2
      inputs:
        buildType: 'current'
        targetPath: '$(Pipeline.Workspace)'
    - task: CmdLine@2
      name: CloneGraphAPI
      displayName: Clone Graph API repo
      inputs:
        script: 'git clone --branch $(Branch) --single-branch https://github.com/wesley-trust/GraphAPI.git'
        workingDirectory: '$(System.ArtifactsDirectory)'
    - task: CmdLine@2
      name: CloneToolKit
      displayName: Clone Toolkit repo
      inputs:
        script: 'git clone --branch $(Branch) --single-branch https://github.com/wesley-trust/ToolKit.git'
        workingDirectory: '$(System.ArtifactsDirectory)'
    - task: PowerShell@2
      name: InvokeWTPlanAzureADGroup
      displayName: Invoke-WTPlanAzureADGroup
      inputs:
        targetType: 'inline'
        script: |

          # Import and convert Groups from JSON, should they exist
          $TestPath = Test-Path $(Pipeline.Workspace)\Import\Validate.json -PathType Leaf
          if ($TestPath){
              $ValidateAzureADGroups = Get-Content -Raw -Path $(Pipeline.Workspace)\Import\Validate.json | ConvertFrom-Json -Depth 10
          }

          # Dot source and execute function
          . $(System.ArtifactsDirectory)\GraphAPI\Public\AzureAD\Groups\Pipeline\Invoke-WTPlanAzureADGroup.ps1
            $PlanAzureADGroups = Invoke-WTPlanAzureADGroup `
              -TenantDomain $(TenantDomain) `
              -ClientID ${env:CLIENTID} `
              -ClientSecret ${env:CLIENTSECRET} `
              -AzureADGroups $ValidateAzureADGroups `
              -UpdateExistingGroups

          # Create directory for artifact, if it does not exist
          $TestPath = Test-Path $(Pipeline.Workspace)\Output -PathType Container
          if (!$TestPath){
              New-Item -Path $(Pipeline.Workspace)\Output -ItemType Directory | Out-Null
          }

          # If there are Groups
          if ($PlanAzureADGroups.RemoveGroups -or $PlanAzureADGroups.UpdateGroups -or $PlanAzureADGroups.CreateGroups){

            # Set ShouldRun variable to true, for apply stage
            echo "##vso[task.setvariable variable=ShouldRun;isOutput=true]true"

            # Convert to JSON and export
            $PlanAzureADGroups | ConvertTo-Json -Depth 10 | Out-File -Force -FilePath $(Pipeline.Workspace)\Output\Plan.json
          }
        pwsh: true
        workingDirectory: '$(System.ArtifactsDirectory)'
      env:
        CLIENTID: $(ClientID)
        CLIENTSECRET: $(ClientSecret)
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(Pipeline.Workspace)\Output'
        artifact: 'Evaluate'
        publishLocation: 'pipeline'
```

</details>

### PowerShell example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API and ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Pipeline\Invoke-WTPlanAzureADGroup.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"

# Example valid group (mailNickName if missing, is auto-generated upon creation)
$ValidateAzureADGroup = [PSCustomObject]@{
    displayName     = "SVC-CA; Exclude from all Conditional Access Policies"
    mailEnabled     = $false
    securityEnabled = $true
}

# Create hashtable
$Parameters = @{
  ClientID             = $ClientID
  ClientSecret         = $ClientSecret
  TenantDomain         = $TenantDomain
  UpdateExistingGroups = $true
  AzureADGroup         = $ValidateAzureADGroup
}

# Create a plan, splatting the hashtable of parameters
Invoke-WTPlanAzureADGroup @Parameters

# Or pipe specific object definitions to the plan function, with an access token previously obtained
$ValidateAzureADGroup | Invoke-WTPlanAzureADGroup -AccessToken $AccessToken

# Or specify each parameter individually, with an access token previously obtained
Invoke-WTPlanAzureADGroup -AzureADGroup $ValidateAzureADGroup -AccessToken $AccessToken -UpdateExistingGroups
```

</details>

### What does this do? <!-- omit in toc -->
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- Checks are performed about whether to evaluate groups for updating or removal
- Existing groups in Azure AD are obtained (as appropriate), in order to compare against the validated import
- An object comparison is performed on the group IDs, determining:
  - What groups could be removed (as they exist, but don't have an ID in the import)
  - What groups could be created (as an ID might not exist, or might not match an existing ID in Azure AD)
- A safety check is performed, if no groups are provided, the removal of all existing groups requires a "Force" switch
- If groups should not be removed, the variable for removing groups is cleared
- If groups should be updated, and there are existing groups in Azure AD, only groups with valid IDs are included
- An object comparison is then performed on specific object properties, to check for specific differences (only)
  - If there are differences, they're added to a variable
- If no groups exist, any imported groups must all be created, so the variable is updated
- An object is then built containing the groups to be removed, updated or created (as appropriate)
- This object is then returned as a plan of action, which is output as a pipeline artifact for the next stage

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTPlanAzureADGroup {
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
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD group object"
        )]
        [Alias('AzureADGroup', 'GroupDefinition')]
        [PSCustomObject]$AzureADGroups,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to update existing groups deployed in the tenant, where the IDs match"
        )]
        [switch]
        $UpdateExistingGroups,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether existing groups deployed in the tenant will be removed, if not present in the import"
        )]
        [switch]
        $RemoveExistingGroups,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "If there are no groups to import, whether to forcibly remove any existing groups"
        )]
        [switch]$Force
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "Toolkit\Public\Invoke-WTPropertyTagging.ps1",
                "GraphAPI\Public\AzureAD\Groups\Get-WTAzureADGroup.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

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

                # Output current action
                Write-Host "Evaluating Azure AD Groups"
                
                # Build Parameters
                $Parameters = @{
                    AccessToken = $AccessToken
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }

                # Evaluate groups if parameters exist
                if ($RemoveExistingGroups -or $UpdateExistingGroups) {

                    # Get existing groups for comparison
                    $ExistingGroups = Get-WTAzureADGroup @Parameters

                    if ($ExistingGroups) {

                        if ($AzureADGroups) {

                            # Compare object on id and pass thru all objects, including those that exist and are to be imported
                            $GroupComparison = Compare-Object `
                                -ReferenceObject $ExistingGroups `
                                -DifferenceObject $AzureADGroups `
                                -Property id `
                                -PassThru

                            # Filter for groups that should be removed, as they do not exist in the import
                            $RemoveGroups = $GroupComparison | Where-Object { $_.sideindicator -eq "<=" }

                            # Filter for groups that did not contain an id, and so are groups that should be created
                            $CreateGroups = $GroupComparison | Where-Object { $_.sideindicator -eq "=>" }
                        }
                        else {

                            # If force is enabled, then if removal of groups is specified, all existing will be removed
                            if ($Force) {
                                $RemoveGroups = $ExistingGroups
                            }
                        }

                        if (!$RemoveExistingGroups) {

                            # If groups are not to be removed, disregard any groups for removal
                            $RemoveGroups = $null
                        }
                        if ($UpdateExistingGroups) {
                            if ($AzureADGroups) {
                                
                                # Check whether the groups that could be updated have valid ids (so can be updated, ignore the rest)
                                $UpdateGroups = foreach ($Group in $AzureADGroups) {
                                    if ($Group.id -in $ExistingGroups.id) {
                                        $Group
                                    }
                                }

                                # If groups exist, with ids that matched the import
                                if ($UpdateGroups) {
                            
                                    # Compare again, with all mandatory property elements for differences
                                    $GroupPropertyComparison = Compare-Object `
                                        -ReferenceObject $ExistingGroups `
                                        -DifferenceObject $UpdateGroups `
                                        -Property id, displayName, description, membershipRule

                                    $UpdateGroups = $GroupPropertyComparison | Where-Object { $_.sideindicator -eq "=>" }
                                }
                            }
                        }
                    }
                    else {
                        # If no groups exist, any imported must be created
                        $CreateGroups = $AzureADGroups
                    }
                }
                else {
                    # If no groups are to be removed or updated, any imported must be created
                    $CreateGroups = $AzureADGroups
                }
                
                # Build object to return
                $PlanAzureADGroups = [ordered]@{}

                if ($RemoveGroups) {
                    $PlanAzureADGroups.Add("RemoveGroups", $RemoveGroups)
                    
                    # Output current action
                    Write-Host "Groups to remove: $($RemoveGroups.count)"

                    foreach ($Group in $RemoveGroups) {
                        Write-Host "Remove: Group ID: $($Group.id)" -ForegroundColor DarkRed
                    }
                }
                else {
                    Write-Host "No groups will be removed, as none exist that are different to the import"
                }
                if ($UpdateGroups) {
                    $PlanAzureADGroups.Add("UpdateGroups", $UpdateGroups)
                                        
                    # Output current action
                    Write-Host "Groups to update: $($UpdateGroups.count)"
                    
                    foreach ($Group in $UpdateGroups) {
                        Write-Host "Update: Group ID: $($Group.id)" -ForegroundColor DarkYellow
                    }
                }
                else {
                    Write-Host "No groups will be updated, as none exist that are different to the import"
                }
                if ($CreateGroups) {
                    $PlanAzureADGroups.Add("CreateGroups", $CreateGroups)
                                        
                    # Output current action
                    Write-Host "Groups to create: $($CreateGroups.count)"

                    foreach ($Group in $CreateGroups) {
                        Write-Host "Create: Group Name: $($Group.displayName)" -ForegroundColor DarkGreen
                    }
                }
                else {
                    Write-Host "No groups will be created, as none exist that are different to the import"
                }

                # If there are groups, return PS object
                if ($PlanAzureADGroups) {
                    $PlanAzureADGroups = [PSCustomObject]$PlanAzureADGroups
                    $PlanAzureADGroups
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

[function-plan]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Pipeline/Invoke-WTPlanAzureADGroup.ps1
[devops-link]: https://dev.azure.com/wesleytrust/GraphAPI
[github-repo]: https://github.com/wesley-trust/GraphAPIConfig
[validate-post]: /blog/graph-api-groups-pipeline-validate/
[apply-post]: /blog/graph-api-groups-pipeline-apply/