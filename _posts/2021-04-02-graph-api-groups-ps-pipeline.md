---
title: "Using a CI/CD Pipeline to manage Azure AD groups, Stage 1: Import & Validate"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - azuread
  - pipeline
  - validate
  - import
excerpt: "This post covers the first stage in the pipeline which will be used to automate creating, updating and removing Azure AD groups..."
---
Pipelines are awesome for automating Infrastructure as Code, I'm mostly making use of [Azure DevOps][devops-link] to execute my Pipeline, but the YAML is also compatible with GitHub Actions, making it easy to use on either platform.

For managing Azure AD groups in a pipeline, I'm taking a three stage approach consisting of:
- Import & Validate
- Evaluate & Plan
- Apply & Deploy

This post covers the YAML and PowerShell involved in the first stage of importing and validating the input. The PowerShell can also be called directly.

## Invoke-WTValidateAzureADGroup
This function is [Invoke-WTValidateAzureADGroup][function-validate], which you can access from my GitHub.

This imports JSON definitions of groups, or imports objects via a parameter, and validates these against a set of criteria. Outputting a valid JSON file as a pipeline artifact for the next stage in the pipeline.

Pipeline YAML example below (which is triggered on a change to the GraphAPIConfig repo):

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```yaml
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
        script: 'git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git'
        workingDirectory: '$(System.ArtifactsDirectory)'
    - task: PowerShell@2
      name: InvokeWTValidateAzureADGroup
      displayName: Invoke-WTValidateAzureADGroup
      inputs:
        targetType: 'inline'
        script: |

          # Dot source and execute function
          . $(System.ArtifactsDirectory)\GraphAPI\Public\AzureAD\Groups\Pipeline\Invoke-WTValidateAzureADGroup.ps1
          $ValidateAzureADGroups = Invoke-WTValidateAzureADGroup `
            -Path $(Build.Repository.LocalPath)\AzureAD\Groups
          
          # Create directory for artifact, if it does not exist
          $TestPath = Test-Path $(Pipeline.Workspace)\Output -PathType Container
          if (!$TestPath){
            New-Item -Path $(Pipeline.Workspace)\Output -ItemType Directory | Out-Null
          }

          # If there are Groups (as if there are no groups to import, existing groups are not removed)
          if ($ValidateAzureADGroups){
            
            # Set ShouldRun variable to true, for plan stage
            echo "##vso[task.setvariable variable=ShouldRun;isOutput=true]true"
            
            # Convert to JSON and export
            $ValidateAzureADGroups | ConvertTo-Json -Depth 10 | Out-File -Force -FilePath $(Pipeline.Workspace)\Output\Validate.json
          }
        pwsh: true
        workingDirectory: '$(System.ArtifactsDirectory)'
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: '$(Pipeline.Workspace)\Output'
        artifact: 'Import'
        publishLocation: 'pipeline'
```

</details>

PowerShell example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```powershell
# Clone repo that contains the Graph API functions and config definitions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPIConfig.git

# Dot source function into memory
. .\GraphAPI\Public\AzureAD\Groups\Pipeline\Invoke-WTValidateAzureADGroup.ps1

# Define Variables
$Path = ".\GraphAPIConfig\AzureAD\Groups"
$FilePath = ".\GraphAPIConfig\AzureAD\Groups\SVC-CA\SVC-CA; Exclude from all Conditional Access Policies.json"

# Example valid group (mailNickName if missing, is auto-generated upon creation)
$AzureADGroup = [PSCustomObject]@{
    displayName     = "SVC-CA; Exclude from all Conditional Access Policies"
    mailEnabled     = $false
    securityEnabled = $true
}
# Example invalid group (mailNickName if missing, is auto-generated upon creation)
# Missing property (displayName), as well as null property value (securityEnabled)
$AzureADGroup = [PSCustomObject]@{
    mailEnabled     = $false
    securityEnabled = $null
}

# Import and validate all JSON files from the path specified
Invoke-WTValidateAzureADGroup -Path $Path

# Or import and validate a specific JSON file from the filepath specified
Invoke-WTValidateAzureADGroup -FilePath $FilePath

# Or pipe specific object definitions to the validate function
$AzureADGroup | Invoke-WTValidateAzureADGroup
```

</details>

### What does this do?
- This sets specific variables, including the required properties that must be present in the input
- A file path to specific files, a path from which all files will be imported, or a group passed in a parameter is required
- This then checks for the properties each group has
  - Each required property that is missing is added to a variable
- A check is then performed as to whether the properties contain a value
  - This is again added to a variable if null
- A validate object is then built for each group with failed checks
- Information is then returned about whether the group passed validation, and if not, what each group failed
- If successful, the validated group objects are returned

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Invoke-WTValidateAzureADGroup {
    [cmdletbinding()]
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
            HelpMessage = "The Azure AD Groups to be validated if not imported from a JSON file"
        )]
        [Alias('AzureADGroup', 'GroupDefinition')]
        [PSCustomObject]$AzureADGroups,
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
            $RequiredProperties = @("displayName", "mailEnabled", "securityEnabled")

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
                    $FilePath = foreach ($Directory in $Path) {
                        (Get-ChildItem -Path $Directory -Filter "*.json" -Recurse).FullName
                    }
                    if (!$FilePath) {
                        $ErrorMessage = "No JSON files were found in the location specified $Path, please check the path is correct"
                        throw $ErrorMessage
                    }
                }
                else {
                    $ErrorMessage = "The provided path does not exist $Path, please check the path is correct"
                    throw $ErrorMessage
                }
            }

            # Import groups from JSON file, if the files exist
            if ($FilePath) {
                $AzureADGroupImport = foreach ($File in $FilePath) {
                    $FilePathExists = $null
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
                if ($AzureADGroupImport) {
                    $AzureADGroups = $AzureADGroupImport | ConvertFrom-Json
                }
                else {
                    $ErrorMessage = "No JSON files could be imported, please check the filepath is correct"
                    throw $ErrorMessage
                }
            }

            # If a file has been imported, or objects provided in the parameter
            if ($AzureADGroups) {
                
                # Output current action
                Write-Host "Importing Azure AD Groups"
                Write-Host "Groups: $($AzureADGroups.count)"
                
                foreach ($Group in $AzureADGroups) {
                    if ($Group.displayName) {
                        Write-Host "Import: Group Name: $($Group.displayName)"
                    }
                    elseif ($Group.id) {
                        Write-Host "Import: Group Id: $($Group.id)"
                    }
                    else {
                        Write-Host "Import: Group Invalid"
                    }
                }

                # If import only is set, return groups without validating
                if ($ImportOnly) {
                    $AzureADGroups
                }
                else {
                        
                    # Output current action
                    Write-Host "Validating Azure AD Groups"
    
                    # For each group, run validation checks
                    $InvalidGroups = foreach ($Group in $AzureADGroups) {
                        $GroupValidate = $null
    
                        # Check for missing properties
                        $GroupProperties = $null
                        $GroupProperties = ($Group | Get-Member -MemberType NoteProperty).name
                        $PropertyCheck = $null

                        # Check whether each required property, exists in the list of properties for the object
                        $PropertyCheck = foreach ($Property in $RequiredProperties) {
                            if ($Property -notin $GroupProperties) {
                                $Property
                            }
                        }

                        # Check whether each required property has a value, if not, return property
                        $PropertyValueCheck = $null
                        $PropertyValueCheck = foreach ($Property in $RequiredProperties) {
                            if ($null -eq $Group.$Property) {
                                $Property
                            }
                        }

                        # Build and return object
                        if ($PropertyCheck -or $PropertyValueCheck) {
                            $GroupValidate = [ordered]@{}
                            if ($Group.displayName) {
                                $GroupValidate.Add("DisplayName", $Group.displayName)
                            }
                            elseif ($Group.id) {
                                $GroupValidate.Add("Id", $Group.id)
                            }
                        }
                        if ($PropertyCheck) {
                            $GroupValidate.Add("MissingProperties", $PropertyCheck)
                        }
                        if ($PropertyValueCheck) {
                            $GroupValidate.Add("MissingPropertyValues", $PropertyValueCheck)
                        }
                        if ($GroupValidate) {
                            [pscustomobject]$GroupValidate
                        }
                    }

                    # Return validation result for each group
                    if ($InvalidGroups) {
                        Write-Host "Invalid Groups: $($InvalidGroups.count) out of $($AzureADGroups.count) imported"
                        foreach ($Group in $InvalidGroups) {
                            if ($Group.displayName) {
                                Write-Host "INVALID: Group Name: $($Group.displayName)" -ForegroundColor Yellow
                            }
                            elseif ($Group.id) {
                                Write-Host "INVALID: Group Id: $($Group.id)" -ForegroundColor Yellow
                            }
                            else {
                                Write-Host "INVALID: No displayName or Id for group" -ForegroundColor Yellow
                            }
                            if ($Group.MissingProperties) {
                                Write-Warning "Required properties not present ($($Group.MissingProperties.count)): $($Group.MissingProperties)"
                            }
                            if ($Group.MissingPropertyValues) {
                                Write-Warning "Required property values not present ($($Group.MissingPropertyValues.count)): $($Group.MissingPropertyValues)"
                            }
                        }
    
                        # Abort import
                        $ErrorMessage = "Validation of groups was not successful, review configuration files and any warnings generated"
                        throw $ErrorMessage
                    }
                    else {

                        # Return validated groups
                        Write-Host "All groups have passed validation for required properties and values"
                        $ValidGroups = $AzureADGroups
                        $ValidGroups
                    }
                }
                
            }
            else {
                $ErrorMessage = "No Azure AD groups to be imported, import may have failed or none may exist"
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

[function-validate]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Pipeline/Invoke-WTValidateAzureADGroup.ps1
[devops-link]: https://dev.azure.com/wesleytrust/GraphAPI