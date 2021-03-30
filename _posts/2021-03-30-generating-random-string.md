---
title: "Another ToolKit function, to generate a random string"
categories:
  - blog
tags:
  - random-string
  - random-password
  - password-generator
  - toolkit
  - powershell
excerpt: "Using tags in Azure or AWS is a native experience, but to tag a response from the Graph API, required that I create a PowerShell function..."
---

This is an oldie, but a goodie, it's one of the first functions I wrote about four years ago which has come in super handy since. I originally wrote this to generate a random string that I could make use of as a password.

The original version only allowed for each character from the character set to appear once in the final string, which was a fairly big limitation... _I try to keep the mantra that “perfect is the enemy of good”, but I’ve since refactored this so that each character is randomised each time from the character set (which is slower but more random)._

The use case for the Graph API, is using this for Azure AD groups. When creating an Azure AD group, there are certain required fields, one of which is mailNickName, which must be a unique string in the Azure AD tenant. _Annoyingly, even for security groups, you must also specify this parameter._

## Obtaining a random string
To get a random string, we use the [New-WTRandomString][function-link] function I wrote which is on my GitHub.

Examples below:

```powershell
# Clone repo that contains the ToolKit functions
git clone --branch main --single-branch https://github.com/wesley-trust/ToolKit.git

# Dot source function into memory
. .\ToolKit\Public\New-WTRandomString.ps1

# Define Variables
$CharacterLength = 24

# Build parameters
$Parameters = @{
  CharacterLength = $CharacterLength
  Alphanumeric    = $true
}

# Executing as is, uses the defaults of all character sets and a character length of 12
New-WTRandomString

# Or splat the hashtable of parameters
New-WTRandomString @Parameters

# Or specify each parameter individually
New-WTRandomString -CharacterLength $CharacterLength -Alphanumeric
```

### What does this do?
- First the variables are set, these are the specific character sets for lowercase, uppercase, numbers and special characters
- By default, all character sets are used with a length of 12, but you can also specify "Simplified" for letters only, as well as "Alphanumeric"
- The character sets are built up as individual objects, for each character, I randomise the character set by combining "Sort-Object" with "Get-Random"
  - Selecting the first character returned, and adding this to the object collection (which I then randomise again for good measure)
- I then join all the objects together into a single string and return this

```powershell
function New-WTRandomString {
    [cmdletbinding()]
    Param(
        [Parameter(
            Mandatory = $false,
            HelpMessage = "Specify the character length (maximum 256)"
        )]
        [ValidateRange(1, 256)]
        [int]
        $CharacterLength = 12,
        [Parameter(
            Mandatory = $false,
            HelpMessage = "Specify whether to use alphabetic characters only"
        )]
        [switch]
        $Simplified,
        [Parameter(
            Mandatory = $false,
            HelpMessage = "Specify whether to use alphabetic and numeric characters only"
        )]
        [switch]
        $Alphanumeric
    )
    Begin {
        try {
            
            # Variables
            $LowerCase = ([char[]](97..122))
            $UpperCase = ([char[]](65..90))
            $Numbers = ([char[]](48..57))
            $Special = ([char[]](33..47))
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {

            # Build the character sets
            if ($Simplified) {
                $CharacterSet = $LowerCase + $UpperCase
            }
            elseif ($Alphanumeric) {
                $CharacterSet = $LowerCase + $UpperCase + $Numbers
            }
            else {
                $CharacterSet = $LowerCase + $UpperCase + $Numbers + $Special
            }

            # For each character, randomise the set and return the first character
            $Object = foreach ($Character in 1..$CharacterLength){
                $RandomisedSet = $CharacterSet | Sort-Object { Get-Random }
                $RandomisedSet[0]
            }
            
            # Randomise object
            $Object = $Object | Sort-Object { Get-Random }
            
            # Join objects to form string
            $RandomString = $Object -join ""
            
            # Return string
            $RandomString
        }
        Catch {
            Write-Error -Message $_.exception
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

[function-link]:https://github.com/wesley-trust/ToolKit/blob/main/Public/New-WTRandomString.ps1