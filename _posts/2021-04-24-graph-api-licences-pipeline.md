---
title: "Automate the deployment of Azure AD group-based licensing in a CI/CD Pipeline"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - azuread
  - pipeline
  - subscriptions
  - licences
  - licensing
excerpt: "This post covers the CI/CD pipeline which will be used to automate creating and assigning licences to Azure AD groups..."
---
Now that we have the PowerShell to [get subscriptions][get-sub], [calculate dependencies][get-dep] and [assign licences to groups][assign-licence], as well as [create groups][create-function], we can bring this all together to automate in a CI/CD pipeline. This is in part based upon the [Azure AD groups pipeline][validate-post].

I'm using [Azure DevOps][devops-link] to execute my Pipeline, but the YAML should also be compatible with GitHub Actions, making it relatively easy to use on either platform.

_Both Azure Pipelines and GitHub Actions have free tiers for public projects, and free execution minutes for private projects._

This post covers the YAML and PowerShell executed in the pipeline, the PowerShell can also be called directly or executed in a Windows Server docker container, making this quite portable and versatile.

### Pipeline Stages
- [Trigger Pipeline](#trigger-pipeline)
- [Shared Pipeline](#shared-pipeline)
  - [Import & Validate](#import--validate)

|  Current Import & Validate Status  |   Current Plan & Evaluate Status   |   Current Apply & Deploy Status   |   Overall CI/CD Pipeline Status   |
|:----------------------------------:|:----------------------------------:|:---------------------------------:|:---------------------------------:|
| [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main&stageName=Validate&jobName=Import)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) | [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main&stageName=Plan&jobName=Evaluate)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) | [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main&stageName=Apply&jobName=Deploy)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) | [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) |

_The apply stage is skipped when there are no changes to deploy, and so may show as "cancelled"_

## Trigger Pipeline
You can access this [trigger here, on my GitHub][trigger-link]. This trigger contains an extend, so that each stage of the rest of the pipeline is included.

### Pipeline YAML example below: <!-- omit in toc -->

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```yaml
trigger:
  batch: true
  branches:
    include:
    - main
  paths:
    include:
      - AzureAD/Subscriptions/

schedules:
- cron: "0 */1 * * *"
  displayName: Run hourly every day
  branches:
    include:
    - main
  always: true

pr: none

extends:
  template: ../Shared/azure-pipelines.yml
```

</details>

#### What does this do? <!-- omit in toc -->
- This triggers on a change to the main branch, for a specific path within the [Graph API Config repo][config-repo]
  - This prevents runs on changes to other config files within the repo
- Changes are batched, so the pipeline will only execute once concurrently
  - This is necessary as if the pipeline ran more than once concurrently, subsequent runs could use stale data to create the "plan" of actions to "apply"
- This is also executed on an hourly schedule for the main branch, and will always run, even if there are no changes to the branch
  - This is so that new subscription licences are picked up from Azure AD and the pipeline executed
- The pipeline is not triggered on pull requests, as this pipeline runs in my sandbox tenant already
  - If triggered on a production repo, this PR could be used to execute the pipeline against a dev environment
- An extend is included, leading to the shared template that contains all the stages of the pipeline

## Shared Pipeline
You can access this [trigger here, on my GitHub][trigger-link]. This trigger contains an extend, so that each stage of the rest of the pipeline is included.

### Pipeline YAML example below: <!-- omit in toc -->
_Azure Pipelines automatically clones the config repo for the first stage, and any artifacts created in subsequent stages_

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

### What does this do? <!-- omit in toc -->

### Import & Validate
This function is [Invoke-WTValidateAzureADGroup][function-validate], which you can access from my GitHub.

This imports JSON definitions of groups, or imports group objects via a parameter, and validates these against a set of criteria.

Outputting a JSON validate file (as appropriate) as a pipeline artifact for the next stage in the pipeline.



### PowerShell example below: <!-- omit in toc -->

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

[get-sub]: https://www.wesleytrust.com/blog/graph-api-group-licences/#getting-subscriptions-in-an-azure-ad-tenant
[get-dep]: https://www.wesleytrust.com/blog/graph-api-group-licences/#evaluating-service-plan-dependencies-for-subscriptions
[assign-licence]: https://www.wesleytrust.com/blog/graph-api-groups-relationship/#create-azure-ad-group-relationships
[devops-link]: https://dev.azure.com/wesleytrust/GraphAPI
[github-repo]: https://github.com/wesley-trust/GraphAPIConfig
[create-function]: /blog/graph-api-groups/#create-an-azure-ad-group
[validate-post]: /blog/graph-api-groups-pipeline-validate/
[config-repo]: https://github.com/wesley-trust/GraphAPIConfig/tree/main/AzureAD/Groups
[access-token]: https://www.wesleytrust.com/blog/obtain-access-token/
[trigger-link]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/Pipeline/AzureAD/Subscriptions/ENV-P/azure-pipelines.yml