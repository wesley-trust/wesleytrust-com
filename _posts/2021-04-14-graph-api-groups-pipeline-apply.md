---
title: "Azure AD groups in a CI/CD Pipeline, Stage 3: Apply & Deploy"
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
excerpt: "This post covers the third stage in the pipeline which will be used to automate creating, updating and removing Azure AD groups..."
---
The third stage in the Azure AD groups CI/CD pipeline applies the changes from the plan provided from the previous stage (should there be any).

This is the third stage, in the three stage pipeline for managing Azure AD groups:
- [Import & Validate][validate-post]
- [Plan & Evaluate][plan-post]
- **Apply & Deploy**

This post covers the YAML and PowerShell involved in the third stage which executes the plan of actions (if any). The PowerShell can also be called directly.

|  Current Import & Validate Status  |   Current Plan & Evaluate Status   |   Current Apply & Deploy Status   |
|:----------------------------------:|:----------------------------------:|:---------------------------------:|
|[![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Groups/SVC-AD%3BENV-P%3B%20Groups?branchName=main&stageName=Validate&jobName=Import)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=9&branchName=main)|[![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Groups/SVC-AD%3BENV-P%3B%20Groups?branchName=main&stageName=Plan&jobName=Evaluate)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=9&branchName=main)|[![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Groups/SVC-AD%3BENV-P%3B%20Groups?branchName=main&stageName=Apply&jobName=Deploy)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=9&branchName=main)|

_The apply stage is skipped when there are no changes to deploy, and so may show as "cancelled"_

## Invoke Apply Azure AD group
This function is [Invoke-WTApplyAzureADGroup][function-apply], which you can access from my GitHub.

Within the pipeline, this imports the plan JSON artifact of groups, which is passed to the function via a parameter. This contains what groups that should be created, updated or removed (as appropriate).

### Pipeline YAML example below:
_Triggered on a change to the Azure AD groups within the [GraphAPIConfig template repo in GitHub][github-repo]_

As Azure AD groups can be created in multiple ways, and by multiple applications, having the config repo being the source of authority didn't seem appropriate, so by default, groups are not removed if they exist in Azure AD and do not exist in the config repo. In the future I might consider a "state" file, similar to Terraform to keep track of this.

_Azure Pipelines automatically downloads artifacts created in the previous stage_

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```yaml
- stage: Apply
  pool:
    vmImage: 'windows-latest'
  dependsOn: Plan
  condition: and(succeeded(), eq(dependencies.Plan.outputs['Evaluate.InvokeWTPlanAzureADGroup.ShouldRun'], 'true'))
  jobs:
  - deployment: Deploy
    continueOnError: false
    environment: $(Environment)
    strategy:
     runOnce:
       deploy:
        steps:
          - checkout: self
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
            name: InvokeWTApplyAzureADGroup
            displayName: Invoke-WTApplyAzureADGroup
            inputs:
              targetType: 'inline'
              script: |

                # Import and convert Groups from JSON, should they exist
                $TestPath = Test-Path $(Pipeline.Workspace)\Evaluate\Plan.json -PathType Leaf
                if ($TestPath){
                    $PlanAzureADGroups = Get-Content -Raw -Path $(Pipeline.Workspace)\Evaluate\Plan.json | ConvertFrom-Json -Depth 10
                }
                
                # Dot source and execute function
                . $(System.ArtifactsDirectory)\GraphAPI\Public\AzureAD\Groups\Pipeline\Invoke-WTApplyAzureADGroup.ps1
                      Invoke-WTApplyAzureADGroup `
                        -TenantDomain $(TenantDomain) `
                        -ClientID ${env:CLIENTID} `
                        -ClientSecret ${env:CLIENTSECRET} `
                        -AzureADGroups $PlanAzureADGroups `
                        -UpdateExistingGroups `
                        -Path $(Build.SourcesDirectory)\AzureAD\Groups `
                        -Pipeline
              pwsh: true
              workingDirectory: '$(System.ArtifactsDirectory)'
            env:
              CLIENTID: $(ClientID)
              CLIENTSECRET: $(ClientSecret)
              GITHUBPAT: $(GitHubPAT)
              REPOHOME: $(Build.Repository.LocalPath)
              BRANCH: $(Branch)
              GITHUBCONFIGREPO: $(GitHubConfigRepo)
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
. .\GraphAPI\Public\AzureAD\Groups\Pipeline\Invoke-WTApplyAzureADGroup.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"
$AccessToken = "HWYLAqz6PipzzdtPwRnSN0Socozs2lZ7nsFky90UlDGTmaZY1foVojTUqFgm1vw0iBslogoP"

# Example groups (mailNickName if missing, is auto-generated upon creation)
$RemoveGroup = [PSCustomObject]@{
    id              = "41fd3497-52hq-983s-sdf23-dsf234kafs24"
    displayName     = "This group will be removed"
    mailEnabled     = $false
    securityEnabled = $true
}
$UpdateGroup = [PSCustomObject]@{
    id              = "52bf4497-f2g7-983s-sdf23-dsf234kafs24"
    displayName     = "This group will be updated"
    mailEnabled     = $false
    securityEnabled = $true
}
$CreateGroup = [PSCustomObject]@{
    displayName     = "This group will be created"
    mailEnabled     = $false
    securityEnabled = $true
}

# Build plan object
$PlanAzureADGroup = [PSCustomObject]@{
    RemoveGroups = $RemoveGroup
    UpdateGroups = $UpdateGroup
    CreateGroups = $CreateGroup
}

# Create hashtable
$Parameters = @{
  ClientID             = $ClientID
  ClientSecret         = $ClientSecret
  TenantDomain         = $TenantDomain
  UpdateExistingGroups = $true
  AzureADGroup         = $PlanAzureADGroup
}

# Apply a plan, splatting the hashtable of parameters
Invoke-WTApplyAzureADGroup @Parameters

# Or pipe specific object definitions to the apply function, with an access token previously obtained
$PlanAzureADGroup | Invoke-WTApplyAzureADGroup -AccessToken $AccessToken

# Or specify each parameter individually, with an access token previously obtained
Invoke-WTApplyAzureADGroup -AzureADGroup $PlanAzureADGroup -AccessToken $AccessToken -UpdateExistingGroups
```

</details>

### What does this do? <!-- omit in toc -->
- An [access token is obtained][access-token], if one is not provided, this allows the same token to be shared within the pipeline
- If groups should be removed, and the objects exist, the group IDs are provided to the [remove group function][remove-function]
- If groups should be updated, and the objects exist, the group objects are provided to the [edit group function][update-function]
- If there are group objects to be created, the objects are provided to the [new group function][create-function]
  - The new group config information is then exported using the [export group function][export-function]
  - This ensures the new group Ids are available in the config to manage in the future 
  - Within the pipeline, the files are added, committed and pushed to the [config repo][config-repo]

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTApplyAzureADGroup {
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
            HelpMessage = "The AzureAD group object"
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
            HelpMessage = "The file path to the JSON file(s) that will be exported"
        )]
        [string]$FilePath,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The directory path(s) of which all JSON file(s) will be exported"
        )]
        [string]$Path,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether the function is operating within a pipeline"
        )]
        [switch]$Pipeline
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Public\AzureAD\Groups\Remove-WTAzureADGroup.ps1",
                "GraphAPI\Public\AzureAD\Groups\New-WTAzureADGroup.ps1",
                "GraphAPI\Public\AzureAD\Groups\Edit-WTAzureADGroup.ps1",
                "GraphAPI\Public\AzureAD\Groups\Export-WTAzureADGroup.ps1"
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
                Write-Host "Deploying Azure AD Groups"
                                
                # Build Parameters
                $Parameters = @{
                    AccessToken = $AccessToken
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }
                
                if ($RemoveExistingGroups) {

                    # If groups require removing, pass the ids to the remove function
                    if ($AzureADGroups.RemoveGroups) {
                        $GroupIDs = $AzureADGroups.RemoveGroups.id
                        Remove-WTAzureADGroup @Parameters -GroupIDs $GroupIDs
                    }
                    else {
                        $WarningMessage = "No groups will be removed, as none exist that are different to the import"
                        Write-Warning $WarningMessage
                    }
                }
                if ($UpdateExistingGroups) {
   
                    # If groups require updating, pass the ids
                    if ($AzureADGroups.UpdateGroups) {
                        Edit-WTAzureADGroup @Parameters -AzureADGroups $AzureADGroups.UpdateGroups
                    }
                    else {
                        $WarningMessage = "No groups will be updated, as none exist that are different to the import"
                        Write-Warning $WarningMessage
                    }
                }

                # If there are new groups to be created, create them, passing through the group state
                if ($AzureADGroups.CreateGroups) {

                    # Create groups
                    $CreatedGroups = New-WTAzureADGroup @Parameters `
                        -AzureADGroups $AzureADGroups.CreateGroups
                        
                    # Update configuration files
                    
                    # Export groups
                    Export-WTAzureADGroup -AzureADGroups $CreatedGroups `
                        -Path $Path `
                        -ExcludeExportCleanup
                    
                    # If executing in a pipeline, stage, commit and push the changes back to the repo
                    if ($Pipeline) {
                        Write-Host "Commit configuration changes post pipeline deployment"
                        Set-Location ${ENV:REPOHOME}
                        git config user.email AzurePipeline@wesleytrust.com
                        git config user.name AzurePipeline
                        git add -A
                        git commit -a -m "Commit configuration changes post deployment [skip ci]"
                        git push https://${ENV:GITHUBPAT}@github.com/wesley-trust/${ENV:GITHUBCONFIGREPO}.git HEAD:${ENV:BRANCH}
                    }
                }
                else {
                    $WarningMessage = "No groups will be created, as none exist that are different to the import"
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

[function-apply]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Pipeline/Invoke-WTApplyAzureADGroup.ps1
[devops-link]: https://dev.azure.com/wesleytrust/GraphAPI
[github-repo]: https://github.com/wesley-trust/GraphAPIConfig
[validate-post]: /blog/graph-api-groups-pipeline-validate/
[plan-post]: /blog/graph-api-groups-pipeline-plan/
[remove-function]: /blog/graph-api-groups/#remove-an-azure-ad-group
[update-function]: /blog/graph-api-groups/#update-an-azure-ad-group
[create-function]: /blog/graph-api-groups/#create-an-azure-ad-group
[export-function]: /blog/graph-api-groups/#export-an-azure-ad-group
[config-repo]: https://github.com/wesley-trust/GraphAPIConfig/tree/main/AzureAD/Groups
[access-token]: https://www.wesleytrust.com/blog/obtain-access-token/