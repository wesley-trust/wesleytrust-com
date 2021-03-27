---
title: "PowerShell wrapped functions to call the Microsoft Graph API"
categories:
  - blog
tags:
  - graphapi
  - powershell
excerpt: "For calling the Graph API, I wrote a series of private PowerShell functions, that public PowerShell functions will call to do the work..."
---
## Calling the Graph API
Calling an API at first can seem a bit daunting for an infrastructure guy, who may be thinking that APIs are just for developers, but using an object-orientated scripting language like PowerShell, can make it far less so. You can apply all the knowledge built up over the years within your comfort zone, and interact with APIs without having to learn another programming language.

For me, to simplify things I broke down the API calls into small specific private functions, that would be in turn be called by public PowerShell functions, with the PowerShell approved verbs, matching each corresponding method of the API call:

| API Method | PowerShell verb | Feature | Description                                |
| ---------- | --------------- | ------- | ------------------------------------------ |
| Get        | Get             | Get     | To get or list objects in the resource     |
| Patch      | Edit            | Update  | To update existing objects in the resource |
| Post       | New             | Create  | To create new objects in the resource      |
| Delete     | Remove          | Remove  | To remove objects in the resource          |

So this led to four method-specific private functions, that public functions will call, that are feature/service specific (IE Getting Azure AD groups).

These functions then call a generalised Microsoft Graph API Query function, so when public function call the API, they call based upon the method required.

Let's break these down.

## Private Functions
Private functions are not intended to be directly called, so I'll be covering more of a top level overview of what each do, rather than providing example usage.

### Invoke-WTGraphQuery
The first function is [Invoke-WTGraphQuery][function-query], which you can access from my GitHub, this is a refactored version of one [Daniel][dan-blog] created.

This allows you to specific the REST method and the Uri (uniform resource identifier), which executes against the Graph API. This also sets up the required parameters, such as the headers and providing the Access Token in the request. The private functions below provide the method to this function, and the public functions provide the Uri (such as 'groups' for Azure AD groups).

### Invoke-WTGraphGet
The [Invoke-WTGraphGet][function-get] function, which you can access from my GitHub, passes the "Get" method to 

<details>
  <summary>View code block</summary>

```
function Invoke-WTGraphGet {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The access token, obtained from executing Get-WTGraphAccessToken"
        )]
        [string]$AccessToken,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will then be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The specific record ids to be returned"
        )]
        [Alias("id")]
        [string[]]$IDs,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The uniform resource indicator"
        )]
        [string]$Uri,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The optional tags that could be evaluated in the response"
        )]
        [string[]]$Tags,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The activity being performed"
        )]
        [string]$Activity
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Private\Invoke-WTGraphQuery.ps1"
                "Toolkit\Public\Invoke-WTPropertyTagging.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Method = "Get"
            $Counter = 1
            $PropertyToTag = "DisplayName"
            
            # Output current activity
            Write-Host $Activity
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {
            if ($AccessToken) {

                # Build parameters
                $Parameters = @{
                    Method = $Method
                }

                # Change the API version if features in preview are to be excluded
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }
                
                # If specific policies are specified, get each, otherwise, get all policies
                if ($IDs) {
                    $QueryResponse = foreach ($ID in $IDs) {
                        
                        # Output progress
                        if ($IDs.count -gt 1) {
                            Write-Host "Processing Query $Counter of $($IDs.count) with ID: $ID"
                                                
                            # Create progress bar
                            $PercentComplete = (($counter / $IDs.count) * 100)
                            Write-Progress -Activity $Activity `
                                -PercentComplete $PercentComplete `
                                -CurrentOperation $ID
                        }
                        else {
                            Write-Host "Processing Query with ID: $ID"
                        }

                        # Increment counter
                        $counter++

                        # Get Query
                        $AccessToken | Invoke-WTGraphQuery `
                            @Parameters `
                            -Uri $Uri/$ID
                    }
                }
                else {
                    $QueryResponse = $AccessToken | Invoke-WTGraphQuery `
                        @Parameters `
                        -Uri $Uri
                }

                # If there is a response, and tags are defined, evaluate the query response for tags, else return without tagging
                if ($QueryResponse) {
                    if ($Tags) {
                        Invoke-WTPropertyTagging -Tags $Tags -QueryResponse $QueryResponse -PropertyToTag $PropertyToTag
                    }
                    else {
                        $QueryResponse
                    }
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

### Invoke-WTGraphPatch



### Invoke-WTGraphPost

### Invoke-WTGraphDelete


[dan-blog]: https://danielchronlund.com/2020/11/26/azure-ad-conditional-access-policy-design-baseline-with-automatic-deployment-support/
[function-get]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphGet.ps1
[function-patch]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphPatch.ps1
[function-post]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphPost.ps1
[function-delete]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphDelete.ps1
[function-query]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphQuery.ps1