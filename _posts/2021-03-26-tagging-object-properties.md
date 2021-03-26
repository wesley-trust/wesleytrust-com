---
title: "Tag-You're it; using PowerShell to evaluate defined tags in an object property"
categories:
  - blog
tags:
  - graphapi
  - tagging
  - toolkit
excerpt: "Using tags in Azure or AWS is a native experience, but to tag a response from the Graph API, required that I create a PowerShell function..."
---

## Why do you need tags?
If you're an infrastructure guy like me, you'll appreciate, and perhaps even love tags, and will likely have been using them for years in Azure or AWS to tag your resources. They're great as you can define things like the resources' lifecycle (prod, dev) or assign the resource to a business cost centre or customer.

You can also do some clever things like use an Azure Function that checks the tags to determine whether to shutdown or start a VM on a schedule (important to realise the benefits the cloud provides), or even use them to define an Azure Update Management schedule (super handy to make sure certain VM nodes only patch on certain days).

In the context of my Conditional Access policies, I already had pseudo tags that I'd put in the display name of the policies, so I could match up what I'd deployed to the design in my Confluence docs. However, quickly after starting work on the Graph API, I realised I needed a way to evaluate these tags on responses returned from the API, so I could do something clever with them.

However there's no native way to tag these responses. They do have an id, as a unique identifier, but these ids don't exist until after you create something, and are randomly generated. I needed something that I was able to control, and crucially, know before the object was created in Azure AD.

## What's the use case?
Each of my Conditional Access policies has a defined reference number, as well as a version, and an environment tag (in [P]roduction use, only in [A]cceptance testing with certain users etc etc).

As well as this being handy visual indicators, this reference number I need to use to match to the inclusion and exclusion groups that I'll be creating for each policy. As you can imagine, this is very important to get right. It also gives me the flexibility to have more use cases in the future.

## Obtaining a tagged object
To get a tagged object, we use the [Invoke-WTPropertyTagging][function-link] I wrote which is on my GitHub, examples below:

```
# Clone repo that contains the ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

# Dot source function into memory
. .\ToolKit\Public\Invoke-WTPropertyTagging.ps1

# Define Variables
$PropertyToTag = "DisplayName"
$Tags = @("REF", "VER", "ENV")

# Build parameters
$Parameters = @{
  PropertyToTag  = $PropertyToTag
  Tags           = $Tags
}

# Create input object
$InputObject = [PSCustomObject]@{
  displayName  = "REF-12;ENV-P;VER-2; Require MFA, for Azure Management"
  conditions = ""
  grantControls = ""
  sessionControls = ""
  state = ""
}

# Pipe the input object to the function, splat the hashtable of parameters and return the tagged input object
$TaggedInputObject = $InputObject | Invoke-WTPropertyTagging @Parameters

# Or specify the inputobject as a parameter, splat the hashtable of the rest of the parameters
$TaggedInputObject = Invoke-WTPropertyTagging -InputObject $InputObject @Parameters

# Or specify each parameter individually
$TaggedInputObject = Get-WTGraphAccessToken -InputObject $InputObject -PropertyToTag $PropertyToTag -Tags $Tags

# Returned $TaggedInputObject:

  REF             : 12
  VER             : 2
  ENV             : P
  displayName     : REF-12;ENV-P;VER-2; Require MFA, for Azure Management
  conditions      : 
  grantControls   : 
  sessionControls : 
  state           : 
```
### What does this do?
This takes an array of tags, as well as a single property to tag, and the input object that will be evaluated for the tags.
- The inputobject can be a collection of objects, and the function contains a foreach loop to support that
- For the object, I get all the property names of the object, as I'll be adding these back to the new tagged object
- A split is done on the defined property for the defined delimiters
  - The delimiters are defined as ";" and "-" so the displayName can be used as the JSON file name (avoiding characters that can't be used in Windows file names)
- For each tag, I check whether the defined property contained the tag
  - If it does, I get the array index of the tag, increment by 1 to get the tag's value, and add to the hastable I created earlier
  - Otherwise I add that tag with null
- I then add the rest of the properties to the hashtable, convert to an object, and return the new tagged object

The complete function as at this date, is below, but please make sure you get the latest version from [GitHub][function-link]:
```
function Invoke-WTPropertyTagging {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The tags to be evaluated"
        )]
        [string[]]$Tags,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The property to tag within the input object"
        )]
        [string]$PropertyToTag,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The input object"
        )]
        [alias("QueryResponse")]
        [psobject]$InputObject
    )
    Begin {
        try {
            
            # Variables
            $MajorDelimiter = ";"
            $MinorDelimiter = "-"
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            foreach ($Object in $InputObject) {
                
                # Get Object properties
                $ObjectProperties = ($Object | Get-Member -MemberType NoteProperty).name 
                
                # Split out Object information by defined delimiter(s) and tag(s)
                $ObjectPropertySplit = ($Object.$PropertyToTag.split($MajorDelimiter)).Split($MinorDelimiter)

                $TaggedInputObject = [ordered]@{}
                foreach ($Tag in $Tags) {

                    # If the tag exists in the display name, 
                    if ($ObjectPropertySplit -contains $Tag) {

                        # Get the object index, increment by one to obtain the tag's value index
                        $TagIndex = $ObjectPropertySplit.IndexOf($Tag)
                        $TagValueIndex = $TagIndex + 1
                        $TagValue = $ObjectPropertySplit[$TagValueIndex]
                        
                        # Add tag to hashtable
                        $TaggedInputObject.Add($Tag, $TagValue)
                    }
                    else {
                        $TaggedInputObject.Add($Tag, $null)
                    }
                }

                # Append all properties and return object
                foreach ($Property in $ObjectProperties) {
                    $TaggedInputObject.Add("$Property", $Object.$Property)
                }

                [pscustomobject]$TaggedInputObject
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
[function-link]: https://github.com/wesley-trust/ToolKit/blob/main/Public/Invoke-WTPropertyTagging.ps1