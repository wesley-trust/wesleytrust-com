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
|------------|-----------------|---------|--------------------------------------------|
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

#### What does this do?
- This allows you to specify the REST method and the Uri (uniform resource identifier), which executes against the Graph API
- This also sets up the required parameters, such as the headers and provides the Access Token in the request
- The private functions below provide the method to this query function, and the public functions provide the Uri (such as 'groups' for Azure AD groups)

<details>
  <summary>Expand code block (always grab the latest version from GitHub)</summary>

```powershell
function Invoke-WTGraphQuery {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The HTTP method for the Microsoft Graph call"
        )]
        [ValidateSet("Get", "Patch", "Post", "Delete", "Put")]
        [string]$Method,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The Uniform Resource Identifier for the Microsoft Graph API call"
        )]
        [string]$Uri,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The request body of the Microsoft Graph API call"
        )]
        [string]$Body,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The access token, obtained from executing Get-WTGraphAccessToken"
        )]
        [string]$AccessToken,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures
    )
    Begin {
        try {
            # Variables
            $ResourceUrl = "https://graph.microsoft.com"
            $ContentType = "application/json"
            $ApiVersion = "beta" # If preview features are in use, the "beta" API must be used

            # Force TLS 1.2
            [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            if ($AccessToken) {

                # Change the API version if features in preview are to be excluded
                if ($ExcludePreviewFeatures) {
                    $ApiVersion = "v1.0"
                }

                $HeaderParameters = @{
                    "Content-Type"  = "application\json"
                    "Authorization" = "Bearer $AccessToken"
                }

                # Create an empty array to store the result
                $QueryRequest = @()
                $QueryResult = @()

                # If the request is to get data, invoke without a body, otherwise append body
                if ($Method -eq "GET") {
                    $QueryRequest = Invoke-RestMethod `
                        -Headers $HeaderParameters `
                        -Uri $ResourceUrl/$ApiVersion/$Uri `
                        -UseBasicParsing `
                        -Method $Method `
                        -ContentType $ContentType
                }
                else {
                    $QueryRequest = Invoke-RestMethod `
                        -Headers $HeaderParameters `
                        -Uri $ResourceUrl/$ApiVersion/$Uri `
                        -UseBasicParsing `
                        -Method $Method `
                        -ContentType $ContentType `
                        -Body $Body
                }
                
                # Check if a value, and if not, an ID is returned, adding either to the query result, ignoring null objects
                if ($QueryRequest.value) {
                    $QueryResult += $QueryRequest.value
                }
                elseif ($QueryRequest.id) {
                    $QueryResult += $QueryRequest
                }

                # Invoke REST methods and fetch data until there are no pages left
                if ("$ResourceUrl/$Uri" -notlike "*`$top*") {
                    while ($QueryRequest."@odata.nextLink") {
                        $QueryRequest = Invoke-RestMethod `
                            -Headers $HeaderParameters `
                            -Uri $QueryRequest."@odata.nextLink" `
                            -UseBasicParsing `
                            -Method $Method `
                            -ContentType $ContentType

                        $QueryResult += $QueryRequest.value
                    }
                }
                
                # Return query result
                $QueryResult
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

### Invoke-WTGraphGet
The [Invoke-WTGraphGet][function-get] function, which you can access from my GitHub, passes the "Get" method to the query.

#### What does this do?
-


<details>
  <summary>Expand code block (always grab the latest version from GitHub)</summary>

```powershell
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
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
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

                # If there is a response, and tags are defined, evaluate the response for tags or return without tagging
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