---
title: "Automating Azure AD group-based licensing in a CI/CD Pipeline"
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

I'm using [Azure DevOps][devops-link] to execute my Pipeline, but with some tweaks the YAML could run in GitHub Actions, making it relatively easy to use on either platform.

_Both Azure Pipelines and GitHub Actions have free tiers for public projects, and free execution minutes for private projects._

This post covers the YAML and PowerShell executed in the pipeline, the PowerShell can also be called directly or executed in a Windows Server docker container, making this quite portable and versatile.

|  Current Import & Validate Status  |   Current Plan & Evaluate Status   |   Current Apply & Deploy Status   |   Overall CI/CD Pipeline Status   |
|:----------------------------------:|:----------------------------------:|:---------------------------------:|:---------------------------------:|
| [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main&stageName=Validate&jobName=Import)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) | [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main&stageName=Plan&jobName=Evaluate)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) | [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main&stageName=Apply&jobName=Deploy)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) | [![Build Status](https://dev.azure.com/wesleytrust/GraphAPI/_apis/build/status/Azure%20AD/Subscriptions/SVC-CS%3BENV-P%3B%20Subscriptions?branchName=main)](https://dev.azure.com/wesleytrust/GraphAPI/_build/latest?definitionId=23&branchName=main) |

_The apply stage is skipped when there are no changes to deploy, and so may show as "cancelled"_

### Pipeline Stages
- [Trigger Pipeline](#trigger-pipeline)
- [Shared Pipeline](#shared-pipeline)
  - [Import & Validate](#import--validate)
  - [Plan & Evaluate](#plan--evaluate)
  - [Apply & Deploy](#apply--deploy)

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
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

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
- Variable groups are defined and included within the pipeline
  - Including the service principal to authenticate with the Graph API and the GitHub PAT for pushing changes
    - These are set as environmental variables for tasks as appropriate
      - This is because they contain secrets that would be exposed in the pipeline if not
  - Subscription group members are defined, for adding members to the groups that are created
    - This automatically licences those members
- The container image for the Azure DevOps agent is defined
- Steps are defined for tasks to clone required repos
- Each stage in the pipeline is defined
  - With a PowerShell task to load the PowerShell pipeline function into memory and execute
  - With artifacts and variables set for subsequent stages as appropriate
    - With conditions set for stages so they only trigger when there is something to do
- In addition, pipeline variables are set to define the branch and environment the pipeline is executing
  - This allows for approvals to be put on the environment, so changes only apply when approved
  - As well as pushing changes to the correct branch in GitHub

#### PowerShell example below: <!-- omit in toc -->
This function is [Invoke-WTSubscriptionImport][function-import], which you can access from my GitHub. This mimics the pipeline stages.

I created this to make it easier to test locally as well as run in a Windows Server docker container using PowerShell 7.

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTAzureADSubscriptionImport {
    [CmdletBinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with the correct Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with the correct Graph permissions"
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
            HelpMessage = "The directory path to the location where the ServicePlan dependencies will be imported"
        )]
        [string]$DependentServicePlansPath,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether defined subscriptions deployed in the tenant will be removed, if not present in the import"
        )]
        [switch]
        $RemoveDefinedSubscriptions,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether the groups used for subscriptions, should not be removed, if the subscription is removed"
        )]
        [switch]
        $ExcludeGroupRemoval,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "If there are no subscriptions, whether to forcibly remove any defined subscriptions"
        )]
        [switch]$Force,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify until what stage the import should invoke. All preceding stages will execute as dependencies"
        )]
        [ValidateSet("Validate", "Plan", "Apply")]
        [string]$Stage = "Apply",
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
                "GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTValidateSubscription.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTPlanSubscription.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Pipeline\Invoke-WTApplySubscription.ps1"
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

            if ($Stage -eq "Validate" -or $Stage -eq "Plan" -or $Stage -eq "Apply") {
                
                # Build Parameters
                $ValidateParameters = @{}
                if ($ExcludePreviewFeatures) {
                    $ValidateParameters.Add("ExcludePreviewFeatures", $true)
                }
                if ($FilePath) {
                    $ValidateParameters.Add("FilePath", $FilePath)
                }
                elseif ($Path) {
                    $ValidateParameters.Add("Path", $Path)
                }
            
                # Import and validate subscriptions
                Write-Host "Stage 1: Validate"
                if ($FilePath -or $Path) {
                    $TestPath = Test-Path $Path -PathType Container
                    if ($TestPath -or $FilePath) {
                        Invoke-WTValidateSubscription @ValidateParameters | Tee-Object -Variable ValidateSubscriptions
                    }
                }
            }

            if ($Stage -eq "Plan" -or $Stage -eq "Apply") {

                # If there is no access token, obtain one
                if (!$AccessToken) {
                    $AccessToken = Get-WTGraphAccessToken `
                        -ClientID $ClientID `
                        -ClientSecret $ClientSecret `
                        -TenantDomain $TenantDomain
                }

                if ($AccessToken) {

                    # Build Parameters
                    $PlanParameters = @{
                        AccessToken = $AccessToken
                    }
                    if ($ExcludePreviewFeatures) {
                        $PlanParameters.Add("ExcludePreviewFeatures", $true)
                    }
                    if ($ValidateSubscriptions) {
                        $PlanParameters.Add("DefinedSubscriptions", $ValidateSubscriptions)
                    }
                    if ($RemoveDefinedSubscriptions) {
                        $PlanParameters.Add("RemoveDefinedSubscriptions", $true)
                    }
                    if ($Force) {
                        $PlanParameters.Add("Force", $true)
                    }
                
                    # Create plan evaluating whether to create, update or remove subscriptions
                    Write-Host "Stage 2: Plan"
                    Invoke-WTPlanSubscription @PlanParameters | Tee-Object -Variable PlanSubscriptions

                }
                else {
                    $ErrorMessage = "No access token specified, obtain an access token object from Get-WTGraphAccessToken"
                    Write-Error $ErrorMessage
                    throw $ErrorMessage
                }

                if ($Stage -eq "Apply") {
                    if ($PlanSubscriptions) {
                        
                        # Import service plan dependencies if they exist and convert from JSON
                        if ($DependentServicePlansPath) {
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
                        }

                        # Build Parameters
                        $ApplyParameters = @{
                            AccessToken          = $AccessToken
                            DefinedSubscriptions = $PlanSubscriptions
                        }
                        if ($ExcludePreviewFeatures) {
                            $ApplyParameters.Add("ExcludePreviewFeatures", $true)
                        }
                        if ($RemoveDefinedSubscriptions) {
                            $ApplyParameters.Add("RemoveDefinedSubscriptions", $true)
                        }
                        if ($ExcludeGroupRemoval) {
                            $ApplyParameters.Add("ExcludeGroupRemoval", $true)
                        }
                        if ($FilePath) {
                            $ApplyParameters.Add("FilePath", $FilePath)
                        }
                        elseif ($Path) {
                            $ApplyParameters.Add("Path", $Path)
                        }
                        if ($Pipeline) {
                            $ApplyParameters.Add("Pipeline", $true)
                        }
                        if ($DependentServicePlans) {
                            $ApplyParameters.Add("DependentServicePlans", $DependentServicePlans)
                        }

                        # Apply plan to Azure AD
                        Write-Host "Stage 3: Apply"
                        Invoke-WTApplySubscription @ApplyParameters
                    }
                    else {
                        $WarningMessage = "No subscriptions will be created, updated or removed, as none exist that are different to the import"
                        Write-Warning $WarningMessage
                    }
                }
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

### Import & Validate
This function is [Invoke-WTValidateSubscription][function-validate], which you can access from my GitHub.

This imports JSON definitions of subscriptions, or imports subscription objects via a parameter, and validates these against a set of criteria.

Outputting a JSON validate file (as appropriate) as a pipeline artifact for the next stage in the pipeline.

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
- If successful, the validated subscription objects are returned

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
            $RequiredProperties = @("skuPartNumber","skuId","servicePlans","capabilityStatus","appliesTo")
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

### Plan & Evaluate
This function is [Invoke-WTPlanSubscription][function-plan], which you can access from my GitHub.

Within the pipeline, this imports the validated JSON artifact of subscriptions (should they exist), which is passed to the function via a parameter. This then creates a plan of what should be created, updated or removed (as appropriate).

Outputting a JSON plan file (as appropriate) as a pipeline artifact for the next stage in the pipeline.

#### What does this do? <!-- omit in toc -->
- Specific variables are set and any dependent functions are imported into memory
- An [access token is obtained][access-token], if one is not provided, this allows the same token to be shared within the pipeline
- Checks are performed about whether to evaluate subscriptions for removal
- Existing subscriptions in Azure AD are obtained from the [get subscriptions function][get-sub], in order to compare against the validated import
- An object comparison is performed on the skuPartNumber, determining:
  - What defined subscriptions could be removed (as they don't exist in Azure AD, but were in the import)
    - So should have their groups removed and the definitions removed in the config repo
  - What existing subscriptions need their definitions creating (as they exist in Azure AD, but were not defined in the import)
    - So should have groups created, subscriptions assigned and definitions created in the config repo
- A safety check is performed if no subscriptions exist but were defined in the import, so removing all defined subscriptions requires a "Force" switch
- If subscriptions should not be removed, the variable for removing subscriptions is cleared
- If no subscriptions exist in the import, any existing subscriptions must all be created, so the variable is updated
- An object is then built containing the subscriptions to be removed or created (as appropriate)
- This object is then returned as a plan of action, which is output as a pipeline artifact for the next stage

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTPlanSubscription {
    [CmdletBinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Subscription Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Subscription Graph permissions"
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
            HelpMessage = "The Subscription object"
        )]
        [Alias("Subscription", "SubscriptionDefinition", "Subscriptions")]
        [PSCustomObject]$DefinedSubscriptions,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether current Subscription deployed in the tenant will be removed, if not present in the import"
        )]
        [switch]
        $RemoveDefinedSubscriptions,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "If there are no Subscription to import, whether to forcibly remove any current Subscription"
        )]
        [switch]$Force
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "Toolkit\Public\Invoke-WTPropertyTagging.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Get-WTAzureADSubscription.ps1"
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
                Write-Host "Evaluating Subscriptions"
                
                # Build Parameters
                $Parameters = @{
                    AccessToken = $AccessToken
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }

                # Get user subscriptions that have not been deleted
                $CurrentSubscriptions = Get-WTAzureADSubscription @Parameters
                $AssignableSubscriptions = $CurrentSubscriptions | Where-Object {
                    $_.capabilityStatus -ne "Deleted" -and $_.appliesTo -eq "User"
                }

                if ($DefinedSubscriptions) {

                    if ($AssignableSubscriptions) {

                        # Compare object on id and pass thru all objects, including those that exist and are to be imported
                        $SubscriptionComparison = Compare-Object `
                            -ReferenceObject $AssignableSubscriptions `
                            -DifferenceObject $DefinedSubscriptions `
                            -Property skuPartNumber `
                            -PassThru

                        # Filter for defined Subscription that should be removed, as they exist only in the import
                        $RemoveSubscriptions = $SubscriptionComparison | Where-Object { $_.sideindicator -eq "=>" }

                        # Filter for defined Subscription that should be created, as they exist only in Azure AD
                        $CreateSubscriptions = $SubscriptionComparison | Where-Object { $_.sideindicator -eq "<=" }
                    }
                    else {

                        # If force is enabled, then if removal of Subscription is specified, all current will be removed
                        if ($Force) {
                            $RemoveSubscriptions = $DefinedSubscriptions
                        }
                    }

                    if (!$RemoveDefinedSubscriptions) {

                        # If Subscription are not to be removed, disregard any Subscription for removal
                        $RemoveSubscriptions = $null
                    }
                }
                else {
                    
                    # If no defined subscription exist, any enabled subscriptions should be defined
                    $CreateSubscriptions = $AssignableSubscriptions
                }
                
                # Build object to return
                $PlanSubscriptions = [ordered]@{}

                if ($RemoveSubscriptions) {
                    $PlanSubscriptions.Add("RemoveSubscriptions", $RemoveSubscriptions)
                    
                    # Output current action
                    Write-Host "Defined Subscription to remove: $($RemoveSubscriptions.count)"

                    foreach ($Subscription in $RemoveSubscriptions) {
                        Write-Host "Remove: Subscription ID: $($Subscription.id) (Subscription Groups will be removed as appropriate)" -ForegroundColor DarkRed
                    }
                }
                else {
                    Write-Host "No Subscription will be removed, as none exist that are different to the import"
                }
                if ($CreateSubscriptions) {
                    $PlanSubscriptions.Add("CreateSubscriptions", $CreateSubscriptions)
                                        
                    # Output current action
                    Write-Host "Defined Subscription to create: $($CreateSubscriptions.count) (Subscription Groups will be created as appropriate)"

                    foreach ($Subscription in $CreateSubscriptions) {
                        Write-Host "Create: Subscription Name: $($Subscription.skuPartNumber)" -ForegroundColor DarkGreen
                    }
                }
                else {
                    Write-Host "No Subscription will be created, as none exist that are different to the import"
                }

                # If there are Subscription, return PS object
                if ($PlanSubscriptions) {
                    $PlanSubscriptions = [PSCustomObject]$PlanSubscriptions
                    $PlanSubscriptions
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

### Apply & Deploy
This function is [Invoke-WTApplySubscription][function-apply], which you can access from my GitHub.

Within the pipeline, this imports the plan JSON artifact of subscriptions, which is passed to the function via a parameter. This contains the subscriptions that should have groups created or removed (as appropriate), as well as licences assigned and definitions created or removed (as appropriate).

#### What does this do? <!-- omit in toc -->
- Specific variables are set and any dependent functions are imported into memory
- An [access token is obtained][access-token], if one is not provided, this allows the same token to be shared within the pipeline
- If subscriptions should be removed,
  - Subscription groups are obtained with the [get subscription group function][get-group] and are tagged with the [property tagging function][tag-post]
  - Then each of the skuPartNumbers have their config removed
  - The group for the subscription is identified, and this is provided to the [remove subscription group function][remove-function]
  - The group config is then removed
- If there are subscription definitions to be created,
  - If there are service plan dependencies, these are evaluated with the [get subscription dependency function][get-dep]
  - Display names for the subscription groups are then created and provided to the [new subscription group function][create-function]
  - The groups are then tagged with the [property tagging function][tag-post]
  - For each subscription, the subscription group is identified
  - If the subscription has a dependency, each dependency is assigned, then the subscription itself, using the [new group relationship function][new-relationship]
  - Then, to get around the lack of nested group support for licence assignment,
    - I get the members of a group defined in the pipeline with the [get group relationship function][get-relationship]
    - And add these with the [new group relationship function][new-relationship] (using different parameter values)
  - The new subscription definitions are then exported using the [export subscription function][export-sub-function]
    - This acts as a system state, storing subscriptions that have been processed
  - The new group config is also exported using the [export group function][export-group-function]
  - Within the pipeline, the files are added, committed and pushed to the [config repo][config-repo]

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTApplySubscription {
    [CmdletBinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Subscription Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Subscription Graph permissions"
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
            HelpMessage = "The Subscription object"
        )]
        [Alias("Subscription", "SubscriptionDefinition", "Subscriptions")]
        [PSCustomObject]$DefinedSubscriptions,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The dependent service plan objects"
        )]
        [Alias("ServicePlan", "ServicePlans", "DependentServicePlan")]
        [PSCustomObject]$DependentServicePlans,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether existing subscriptions deployed in the tenant will be removed, if not present in the import"
        )]
        [switch]
        $RemoveDefinedSubscriptions,
        [Parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to the groups used for subscriptions, should not be removed, if the subscription is removed"
        )]
        [switch]
        $ExcludeGroupRemoval,
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
                "Toolkit\Public\Invoke-WTPropertyTagging.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Groups\Get-WTAADSubscriptionGroup.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Groups\New-WTAADSubscriptionGroup.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Groups\Remove-WTAADSubscriptionGroup.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Get-WTAzureADSubscriptionDependency.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Export-WTAzureADSubscription.ps1",
                "GraphAPI\Public\AzureAD\Groups\Export-WTAzureADGroup.ps1",
                "GraphAPI\Public\AzureAD\Groups\Relationship\Get-WTAzureADGroupRelationship.ps1",
                "GraphAPI\Public\AzureAD\Groups\Relationship\New-WTAzureADGroupRelationship.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Tag = "SKU"
            $PropertyToTag = "displayName"
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
                Write-Host "Deploying Subscriptions"

                # Build Parameters
                $Parameters = @{
                    AccessToken = $AccessToken
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }

                if ($RemoveDefinedSubscriptions) {

                    # If subscriptions require removing, pass the ids to the remove function
                    if ($DefinedSubscriptions.RemoveSubscriptions) {
                        
                        # Get and tag group for the subscriptions
                        $SubscriptionGroups = Get-WTAADSubscriptionGroup
                        $TaggedSubscriptionGroups = Invoke-WTPropertyTagging -Tags $Tag -QueryResponse $SubscriptionGroups -PropertyToTag $PropertyToTag

                        # Path to group config
                        $GroupsPath = $Path + "\..\Groups"

                        # Remove subscription definition and groups
                        $SubscriptionSkuPartNumbers = $DefinedSubscriptions.RemoveSubscriptions.skuPartNumber
                        foreach ($SubscriptionSkuPartNumber in $SubscriptionSkuPartNumbers) {
                            Remove-Item -Path "$Path\$SubscriptionSkuPartNumber.json"
                        
                            # If the switch to not remove groups is not set, remove the groups for each Subscription also
                            if (!$ExcludeGroupRemoval) {

                                # Identify the group for the subscription
                                $SubscriptionGroup = $null
                                $SubscriptionGroup = $TaggedSubscriptionGroups | Where-Object {
                                    $_.$Tag -eq $SubscriptionSkuPartNumber
                                }

                                # If there is a group, pass the id which will perform a check and remove only subscription groups
                                if ($SubscriptionGroup) {
                                    
                                    # Remove group (licences should no longer be assigned to deleted subscriptions)
                                    Remove-WTAADSubscriptionGroup @Parameters -IDs $SubscriptionGroup.id

                                    # Remove group config
                                    Remove-Item -Path "$GroupsPath\$($SubscriptionGroup.displayName).json"
                                }
                            }
                        }
                    }
                    else {
                        $WarningMessage = "No subscriptions will be removed, as none exist that are different to the import"
                        Write-Warning $WarningMessage
                    }
                }

                # If there are new subscriptions create the groups
                if ($DefinedSubscriptions.CreateSubscriptions) {
                    $CreateSubscriptions = $DefinedSubscriptions.CreateSubscriptions

                    # Find subscriptions with service plan dependencies
                    if ($DependentServicePlans) {
                        $DependentSubscriptions = Get-WTAzureADSubscriptionDependency @Parameters `
                            -Subscriptions $CreateSubscriptions `
                            -ServicePlans $DependentServicePlans `
                            -DependencyType SkuId
                    }

                    # Calculate the display names to be used for the Subscription groups
                    $SubscriptionGroupDisplayName = foreach ($Subscription in $CreateSubscriptions) {
                        "$Tag" + "-" + $Subscription.skuPartNumber + ";"
                    }

                    # Create groups
                    $SubscriptionGroups = New-WTAADSubscriptionGroup @Parameters -DisplayName $SubscriptionGroupDisplayName

                    # Tag groups
                    $TaggedSubscriptionGroups = Invoke-WTPropertyTagging -Tags $Tag -QueryResponse $SubscriptionGroups -PropertyToTag $PropertyToTag

                    # For each subscription, perform subscription specific changes
                    foreach ($Subscription in $CreateSubscriptions) {

                        # Find the matching group
                        $SubscriptionGroup = $null
                        $SubscriptionGroup = $TaggedSubscriptionGroups | Where-Object {
                            $_.$Tag -eq $Subscription.skuPartNumber
                        }

                        # If there is a group for this subscription (as subscriptions may not always have groups)
                        if ($SubscriptionGroup) {
                            
                            # If this subscription is in the list of dependent subscriptions
                            if ($Subscription.skuId -in $DependentSubscriptions.skuId) {
                                
                                # Filter to the specific subscription dependency
                                $DependentSubscription = $null
                                $DependentSubscription = $DependentSubscriptions | Where-Object {
                                    $_.skuId -eq $Subscription.skuId
                                }

                                # Assign each required sku for the dependent subscription
                                foreach ($SkuId in $DependentSubscription.RequiredSkuId) {
                                    New-WTAzureADGroupRelationship @Parameters `
                                        -Id $SubscriptionGroup.id `
                                        -Relationship "assignLicense" `
                                        -RelationshipIDs $SkuId `
                                    | Out-Null
                                }
                            }

                            # Assign licence to group
                            New-WTAzureADGroupRelationship @Parameters `
                                -Id $SubscriptionGroup.id `
                                -Relationship "assignLicense" `
                                -RelationshipIDs $Subscription.skuId `
                            | Out-Null
                            
                            # Workaround lack of nested group support, by getting users that should be licenced
                            if (${ENV:UserGroupID}) {
                                $Members = Get-WTAzureADGroupRelationship @Parameters `
                                    -Id ${ENV:UserGroupID} `
                                    -Relationship "members"
                                
                                # Then adding the users that should be licenced directly to the group
                                if ($Members) {
                                    New-WTAzureADGroupRelationship @Parameters `
                                        -Id $SubscriptionGroup.id `
                                        -Relationship "members" `
                                        -RelationshipIDs $Members.id
                                }
                            }
                        }
                    }

                    # Export subscriptions
                    Export-WTAzureADSubscription -DefinedSubscriptions $CreateSubscriptions `
                        -Path $Path `
                        -ExcludeExportCleanup

                    # Path to group config
                    $GroupsPath = $Path + "\..\Groups"

                    # Export groups
                    Export-WTAzureADGroup -AzureADGroups $SubscriptionGroups `
                        -Path $GroupsPath `
                        -ExcludeExportCleanup `
                        -ExcludeTagEvaluation

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
                    $WarningMessage = "No subscriptions will be created, as none exist that are different to the import"
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

[get-sub]: /blog/graph-api-group-licences/#getting-subscriptions-in-an-azure-ad-tenant
[get-dep]: /blog/graph-api-group-licences/#evaluating-service-plan-dependencies-for-subscriptions
[assign-licence]: /blog/graph-api-groups-relationship/#create-azure-ad-group-relationships
[devops-link]: https://dev.azure.com/wesleytrust/GraphAPI
[github-repo]: https://github.com/wesley-trust/GraphAPIConfig
[validate-post]: /blog/graph-api-groups-pipeline-validate/
[config-repo]: https://github.com/wesley-trust/GraphAPIConfig/tree/main/AzureAD/Subscriptions
[access-token]: /blog/obtain-access-token/
[trigger-link]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/Pipeline/AzureAD/Subscriptions/ENV-P/azure-pipelines.yml
[shared-link]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/Pipeline/AzureAD/Subscriptions/Shared/azure-pipelines.yml
[function-validate]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Pipeline/Invoke-WTValidateSubscription.ps1
[function-plan]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Pipeline/Invoke-WTPlanSubscription.ps1
[function-apply]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Pipeline/Invoke-WTApplySubscription.ps1
[function-import]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Pipeline/Invoke-WTSubscriptionImport.ps1
[remove-function]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Groups/Remove-WTAADSubscriptionGroup.ps1
[create-function]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Groups/New-WTAADSubscriptionGroup.ps1
[get-group]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Groups/Get-WTAADSubscriptionGroup.ps1
[tag-post]: /blog/tagging-object-properties/#invoke-property-tagging
[get-relationship]: /blog/graph-api-groups-relationship/#get-azure-ad-group-relationships
[new-relationship]: /blog/graph-api-groups-relationship/#create-azure-ad-group-relationships
[export-sub-function]: /blog/graph-api-group-licences/#exporting-subscriptions
[export-group-function]: /blog/graph-api-groups/#export-an-azure-ad-group