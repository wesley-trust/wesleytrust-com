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
You can access the [trigger pipeline on my GitHub here][trigger-link]. This trigger contains an extend, so that each stage of the rest of the pipeline is included.

#### Pipeline YAML example below: <!-- omit in toc -->

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
You can access the [shared pipeline on my GitHub here][shared-link].

#### Pipeline YAML example below: <!-- omit in toc -->
_Azure Pipelines automatically clones the config repo for the first stage, and any artifacts created in subsequent stages_

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```yaml
variables:
- group: 'GitHubAuth'
- group: 'ServicePrincipal'
- group: 'SubscriptionMemberGroups'
stages:
- stage: Validate
  pool:
    vmImage: 'windows-latest'
  jobs:
  - job: Import
    pool:
      vmImage: 'windows-latest'
    continueOnError: false
    steps:
    - task: CmdLine@2
      name: CloneGraphAPI
      displayName: Clone Graph API repo
      inputs:
        script: 'git clone --branch $(Branch) --single-branch https://github.com/wesley-trust/GraphAPI.git'
        workingDirectory: '$(System.ArtifactsDirectory)'
    - task: PowerShell@2
      name: InvokeWTValidateSubscription
      displayName: Invoke-WTValidateSubscription
      inputs:
        targetType: 'inline'
        script: |

          # Dot source function
          . $(System.ArtifactsDirectory)\GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTValidateSubscription.ps1
          
          # Test if directory exist and execute function as appropriate
          $TestPath = Test-Path $(Build.Repository.LocalPath)\AzureAD\Subscriptions\Definitions -PathType Container
          if ($TestPath){
            $ValidateDefinedSubscriptions = Invoke-WTValidateSubscription `
              -Path $(Build.Repository.LocalPath)\AzureAD\Subscriptions\Definitions
          }

          # Create directory for artifact, if it does not exist
          $TestPath = Test-Path $(Pipeline.Workspace)\Output -PathType Container
          if (!$TestPath){
            New-Item -Path $(Pipeline.Workspace)\Output -ItemType Directory | Out-Null
          }

          # If there are Subscriptions (as if there are no Subscriptions to import, existing Subscriptions are not removed)
          if ($ValidateDefinedSubscriptions){
            
            # Convert to JSON and export
            $ValidateDefinedSubscriptions | ConvertTo-Json -Depth 10 | Out-File -Force -FilePath $(Pipeline.Workspace)\Output\Validate.json
          }
        pwsh: true
        workingDirectory: '$(System.ArtifactsDirectory)'
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(Pipeline.Workspace)\Output'
        artifact: 'Import'
        publishLocation: 'pipeline'
- stage: Plan
  pool:
    vmImage: 'windows-latest'
  dependsOn: Validate
  condition: succeeded()
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
      name: InvokeWTPlanSubscription
      displayName: Invoke-WTPlanSubscription
      inputs:
        targetType: 'inline'
        script: |

          # Import and convert Subscriptions from JSON, should they exist
          $TestPath = Test-Path $(Pipeline.Workspace)\Import\Validate.json -PathType Leaf
          if ($TestPath){
              $ValidateDefinedSubscriptions = Get-Content -Raw -Path $(Pipeline.Workspace)\Import\Validate.json | ConvertFrom-Json -Depth 10
          }

          # Dot source and execute function
          . $(System.ArtifactsDirectory)\GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTPlanSubscription.ps1
            $PlanDefinedSubscriptions = Invoke-WTPlanSubscription `
              -TenantDomain $(TenantDomain) `
              -ClientID ${env:CLIENTID} `
              -ClientSecret ${env:CLIENTSECRET} `
              -DefinedSubscriptions $ValidateDefinedSubscriptions `
              -RemoveDefinedSubscriptions `
              -Force

          # Create directory for artifact, if it does not exist
          $TestPath = Test-Path $(Pipeline.Workspace)\Output -PathType Container
          if (!$TestPath){
              New-Item -Path $(Pipeline.Workspace)\Output -ItemType Directory | Out-Null
          }

          # If there are Subscriptions
          if ($PlanDefinedSubscriptions.RemoveSubscriptions -or $PlanDefinedSubscriptions.CreateSubscriptions){

            # Set ShouldRun variable to true, for apply stage
            echo "##vso[task.setvariable variable=ShouldRun;isOutput=true]true"

            # Convert to JSON and export
            $PlanDefinedSubscriptions | ConvertTo-Json -Depth 10 | Out-File -Force -FilePath $(Pipeline.Workspace)\Output\Plan.json
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
- stage: Apply
  pool:
    vmImage: 'windows-latest'
  dependsOn: Plan
  condition: and(succeeded(), eq(dependencies.Plan.outputs['Evaluate.InvokeWTPlanSubscription.ShouldRun'], 'true'))
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
            name: InvokeWTApplySubscription
            displayName: Invoke-WTApplySubscription
            inputs:
              targetType: 'inline'
              script: |

                # Import and convert Subscriptions from JSON, should they exist
                $TestPath = Test-Path $(Pipeline.Workspace)\Evaluate\Plan.json -PathType Leaf
                if ($TestPath){
                    $PlanDefinedSubscriptions = Get-Content -Raw -Path $(Pipeline.Workspace)\Evaluate\Plan.json | ConvertFrom-Json -Depth 10
                }

                # Import service plan dependencies if they exist and convert from JSON
                $DependentServicePlansPath = "$(Build.Repository.LocalPath)\AzureAD\Subscriptions\Dependencies"
                $PathExists = Test-Path -Path $DependentServicePlansPath
                if ($PathExists) {
                    $DependentServicePlansFilePath = (Get-ChildItem -Path $DependentServicePlansPath -Filter "*.json").FullName
                }
                if ($DependentServicePlansFilePath) {
                    $DependentServicePlansImport = foreach ($DependentServicePlanFile in $DependentServicePlansFilePath) {
                        Get-Content -Raw -Path $DependentServicePlanFile
                    }
                }
                if ($DependentServicePlansImport) {
                    $DependentServicePlans = $DependentServicePlansImport | ConvertFrom-Json -Depth 10
                }

                # Dot source and execute function
                . $(System.ArtifactsDirectory)\GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTApplySubscription.ps1
                      Invoke-WTApplySubscription `
                        -TenantDomain $(TenantDomain) `
                        -ClientID ${env:CLIENTID} `
                        -ClientSecret ${env:CLIENTSECRET} `
                        -DefinedSubscriptions $PlanDefinedSubscriptions `
                        -DependentServicePlans $DependentServicePlans `
                        -RemoveDefinedSubscriptions `
                        -Path $(Build.SourcesDirectory)\AzureAD\Subscriptions\Definitions `
                        -Pipeline
              pwsh: true
              workingDirectory: '$(System.ArtifactsDirectory)'
            env:
              CLIENTID: $(ClientID)
              CLIENTSECRET: $(ClientSecret)
              GITHUBPAT: $(GitHubPAT)
              REPOHOME: $(Build.Repository.LocalPath)
              BRANCH: $(Branch)
              USERGROUPID: $(UserGroupID)
              GITHUBCONFIGREPO: $(GitHubConfigRepo)
```

</details>

#### What does this do? <!-- omit in toc -->
- Variable groups are defined and included within the pipeline, set as environmental variables for tasks as appropriate
- The container image for the Azure DevOps agent is defined
- Steps are defined for tasks to clone required repos
- Each stage in the pipeline is defined
  - With a PowerShell task to load the PowerShell pipeline function into memory and execute
  - With artifacts and variables set for subsequent stages as appropriate

### Import & Validate
This function is [Invoke-WTValidateAzureADGroup][function-validate], which you can access from my GitHub.

This imports JSON definitions of subscriptions, or imports subscription objects via a parameter, and validates these against a set of criteria.

Outputting a JSON validate file (as appropriate) as a pipeline artifact for the next stage in the pipeline.

#### PowerShell example below: <!-- omit in toc -->

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions and config definitions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPIConfig.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTValidateSubscription.ps1

# Define Variables
$Path = ".\GraphAPIConfig\AzureAD\Subscriptions\Definitions"

# Import and validate all JSON files from the path specified
$TestPath = Test-Path $Path -PathType Container
if ($TestPath){
    Invoke-WTValidateSubscription -Path $Path
}
```

</details>

#### What does this do? <!-- omit in toc -->
- This sets specific variables, including the required properties that must be present in the input
- To import, a file path to specific files or a directory path from which all files will be imported is required
  - Alternatively, a subscription or collection of subscriptions can also be passed in a parameter to validate
- This then checks for the properties each subscription has
  - Each required property that is missing is added to a variable
- A check is then performed as to whether the properties contain a value
  - This is again added to a variable if null
- A validate object is then built for each subscription with failed checks
- Information is then returned about whether the subscription passed validation, and if not, why each subscription failed
- If successful, the validated group objects are returned

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTValidateSubscription {
    [CmdletBinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The file path to the JSON file(s) that will be imported"
        )]
        [string[]]$FilePath,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The directory path(s) of which all JSON file(s) will be imported"
        )]
        [string]$Path,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD Subscriptions to be validated if not imported from a JSON file"
        )]
        [Alias('Subscription', 'SubscriptionDefinition')]
        [PSCustomObject]$DefinedSubscriptions,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether files should be imported only, and not validated"
        )]
        [switch]$ImportOnly
    )
    Begin {
        try {
            # Variables
            $RequiredProperties = @("skuPartNumber")
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            # For each directory, get the file path of all JSON files within the directory, if the directory exists
            if ($Path) {
                $PathExists = Test-Path -Path $Path
                if ($PathExists) {
                    $FilePath = (Get-ChildItem -Path $Path -Filter "*.json").FullName
                }
                else {
                    $ErrorMessage = "The provided path does not exist $Path, please check the path is correct"
                    throw $ErrorMessage
                }
            }

            # Import Subscriptions from JSON file, if the files exist
            if ($FilePath) {
                $SubscriptionImport = foreach ($File in $FilePath) {
                    $FilePathExists = Test-Path -Path $File
                    if ($FilePathExists) {
                        Get-Content -Raw -Path $File
                    }
                    else {
                        $ErrorMessage = "The provided filepath $File does not exist, please check the path is correct"
                        throw $ErrorMessage
                    }
                }
                
                # If import was successful, convert from JSON
                if ($SubscriptionImport) {
                    $DefinedSubscriptions = $SubscriptionImport | ConvertFrom-Json
                }
                else {
                    $ErrorMessage = "No JSON files could be imported, please check the filepath is correct"
                    throw $ErrorMessage
                }
            }

            # If there are subscriptions imported, run validation checks
            if ($DefinedSubscriptions) {
                
                # Output current action
                Write-Host "Importing Defined Subscriptions"
                Write-Host "Subscriptions: $($DefinedSubscriptions.count)"
                
                foreach ($Subscription in $DefinedSubscriptions) {
                    if ($Subscription.skuPartNumber) {
                        Write-Host "Import: Subscription Name: $($Subscription.skuPartNumber)"
                    }
                    elseif ($Subscription.id) {
                        Write-Host "Import: Subscription Id: $($Subscription.id)"
                    }
                    else {
                        Write-Host "Import: Subscription Invalid"
                    }
                }

                # If import only is set, return subscriptions without validating
                if ($ImportOnly) {
                    $DefinedSubscriptions
                }
                else {
                        
                    # Output current action
                    Write-Host "Validating Defined Subscriptions"
    
                    # For each policy, run validation checks
                    $InvalidSubscriptions = foreach ($Subscription in $DefinedSubscriptions) {
                        $SubscriptionValidate = $null
    
                        # Check for missing properties
                        $SubscriptionProperties = $null
                        $SubscriptionProperties = ($Subscription | Get-Member -MemberType NoteProperty).name
                        $PropertyCheck = $null

                        # Check whether each required property, exists in the list of properties for the object
                        $PropertyCheck = foreach ($Property in $RequiredProperties) {
                            if ($Property -notin $SubscriptionProperties) {
                                $Property
                            }
                        }

                        # Check whether each required property has a value, if not, return property
                        $PropertyValueCheck = $null
                        $PropertyValueCheck = foreach ($Property in $RequiredProperties) {
                            if ($null -eq $Subscription.$Property) {
                                $Property
                            }
                        }
    
                        # Build and return object
                        if ($PropertyCheck -or $PropertyValueCheck) {
                            $SubscriptionValidate = [ordered]@{}
                            if ($Subscription.skuPartNumber) {
                                $SubscriptionValidate.Add("skuPartNumber", $Subscription.skuPartNumber)
                            }
                            elseif ($Subscription.id) {
                                $SubscriptionValidate.Add("Id", $Subscription.id)
                            }
                        }
                        if ($PropertyCheck) {
                            $SubscriptionValidate.Add("MissingProperties", $PropertyCheck)
                        }
                        if ($PropertyValueCheck) {
                            $SubscriptionValidate.Add("MissingPropertyValues", $PropertyValueCheck)
                        }
                        if ($SubscriptionValidate) {
                            [PSCustomObject]$SubscriptionValidate
                        }
                    }

                    # Return validation result for each policy
                    if ($InvalidSubscriptions) {
                        Write-Host "Invalid subscriptions: $($InvalidSubscriptions.count) out of $($DefinedSubscriptions.count) imported"
                        foreach ($Subscription in $InvalidSubscriptions) {
                            if ($Subscription.skuPartNumber) {
                                Write-Host "INVALID: Subscription Name: $($Subscription.skuPartNumber)" -ForegroundColor Yellow
                            }
                            elseif ($Subscription.id) {
                                Write-Host "INVALID: Subscription Id: $($Subscription.id)" -ForegroundColor Yellow
                            }
                            else {
                                Write-Host "INVALID: No skuPartNumber or Id for policy" -ForegroundColor Yellow
                            }
                            if ($Subscription.MissingProperties) {
                                Write-Warning "Required properties not present ($($Subscription.MissingProperties.count)): $($Subscription.MissingProperties)"
                            }
                            if ($Subscription.MissingPropertyValues) {
                                Write-Warning "Required property values not present ($($Subscription.MissingPropertyValues.count)): $($Subscription.MissingPropertyValues)"
                            }
                        }
    
                        # Abort import
                        $ErrorMessage = "Validation of subscriptions was not successful, review configuration files and any warnings generated"
                        Write-Error $ErrorMessage
                        throw $ErrorMessage
                    }
                    else {

                        # Return validated subscriptions
                        Write-Host "All subscriptions have passed validation for required properties and values"
                        $ValidSubscriptions = $DefinedSubscriptions
                        $ValidSubscriptions
                    }
                }
                
            }
            else {
                $ErrorMessage = "No Subscriptions to be imported, import may have failed or none may exist"
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
[shared-link]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/Pipeline/AzureAD/Subscriptions/Shared/azure-pipelines.yml
[function-validate]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Pipeline/Invoke-WTValidateSubscription.ps1