---
title: "Azure AD subscription licences, with Azure AD group assignment"
categories:
  - blog
tags:
  - graphapi
  - powershell
  - groups
  - azuread
  - licences
  - subscriptions
excerpt: "Using the Graph API & PowerShell, it's possible to assign Azure AD subscription licences to groups for easy and efficient management..."
---
For the next post I had intended on continuing the documentation leading to the Conditional Access pipeline, but over the last week or so, I've been writing code for managing Azure AD subscriptions (licences) and directory roles. All based upon group membership to make management far more efficient. So in short, I got distracted.

Group based licensing can be done manually, but automated this in a CI/CD pipeline is way more fun, and far more efficient. It also wasn't the easiest, so I had to get my thinking cap on a bit and do a fair bit of research and testing. As ever, Microsoft's API documentation has been somewhat lacking.

One of the big obstacles was the dependencies that subscriptions can have, as when assigning licences to groups, all dependent conditions must be met. Let's break this down.

## Subscriptions and Service Plans <!-- omit in toc -->
In Azure AD, a subscriptions is defined as a "subscribedSku" (stock-keeping unit). A subscription such as "Microsoft 365 E3" has a skuPartNumber of "SPE_E3". Within that subscription there are service plans, these are the individual licences that when bundled together make up the subscription.

For example, "Microsoft 365 E3" contains the service plan "EXCHANGE_S_ENTERPRISE", which corresponds to "Exchange Online (Plan 2)". That service plan is also available as a standalone subscription with a skuPartNumber of "EXCHANGEENTERPRISE".

However, there are some service plans that are only available as addons to an existing subscription, and are not available as a standalone subscription in their own right. This means that when you attempt to assign an addon subscription, it will fail unless a dependent subscription is already assigned.

For my Microsoft 365 Dev sandbox tenant, this wasn't a problem, as you get E5 licences for development and testing. However, my production tenant, is licenced with the Microsoft Action Pack from being a Microsoft Partner.

This comes with "Office 365 E3" and "EMS E3", which is great, however it's limiting as I can't develop and rollout the full feature set, so I've purchased the "E5 Security" addon, to make use of feature like Privileged Identity Management, Microsoft Defender for Endpoint and Microsoft Defender for Identity.

This is where the addon complexity came into play, as "E5 Security" is an addon that relies on service plans contained within "Office 365 E3" and "EMS E3", so all three must be assigned together, with the dependencies assigned first.

### Managing Subscriptions
- [Getting subscriptions in an Azure AD tenant](#getting-subscriptions-in-an-azure-ad-tenant)
- [Evaluating service plan dependencies for subscriptions](#evaluating-service-plan-dependencies-for-subscriptions)
- [Assigning subscriptions to Azure AD groups](#assigning-subscriptions-to-azure-ad-groups)

## Getting subscriptions in an Azure AD tenant
The first function is [Get-WTAzureADSubscription][function-getsub], which you can access from my GitHub.

This gets the commercial (IE Microsoft 365) subscriptions deployed in an Azure AD tenant. I'll be using this in the CI/CD pipeline to create a group per subscription and then assign the subscription to the group. Allowing members to then be added to the groups to be assigned those licences.

### What does this do? <!-- omit in toc -->
- This sets specific variables, including the activity and the Graph Uri
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- The private function is then called, with the query altered as appropriate depending on the parameters

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Get-WTAzureADSubscription {
    [CmdletBinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Azure AD subscription Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Azure AD subscription Graph permissions"
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
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD subscriptions to get, this must contain valid id(s)"
        )]
        [Alias("id", "SubscriptionID", "SubscriptionIDs")]
        [string[]]$IDs
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Private\Invoke-WTGraphGet.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Variables
            $Activity = "Getting Azure AD Commercial Subscriptions"
            $Uri = "subscribedSkus"

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
                
                # Build Parameters
                $Parameters = @{
                    AccessToken = $AccessToken
                    Activity    = $Activity
                }
                if ($ExcludePreviewFeatures) {
                    $Parameters.Add("ExcludePreviewFeatures", $true)
                }
                if ($IDs) {
                    $Parameters.Add("IDs", $IDs)
                }

                # Get Azure AD subscriptions with default properties
                $QueryResponse = Invoke-WTGraphGet @Parameters -Uri $Uri

                # Return response if one is returned
                if ($QueryResponse) {
                    $QueryResponse
                }
                else {
                    $WarningMessage = "No Azure AD subscriptions exist in Azure AD, or with parameters specified"
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

## Evaluating service plan dependencies for subscriptions
The next function is [Get-WTAzureADSubscriptionDependency][function-getsubdep], which you can access from my GitHub.

There is no API that Microsoft provide for service plan dependencies for subscriptions (that I've found), so I had to define these myself for the subscriptions that I use.

I defined a [JSON dependency object][dep], which contains service plans that depend on other service plans. This PowerShell function then uses that definition to evaluate the subscriptions, to decide which subscriptions have dependent service plans, and so need to be grouped together with the subscriptions that contain those service plans.

### What does this do? <!-- omit in toc -->
- An access token is obtained, if one is not provided, this allows the same token to be shared within the pipeline
- If no subscriptions are provided to the function, they're obtained from Azure AD
- A check is performed on each subscription, to see if any contain service plans defined as dependencies
- If there are subscriptions with service plan dependencies, a check is performed on which subscription contains the dependent service plan
- An object is built and returned for which subscriptions have dependent subscriptions (as they contain the service plan, and must be assigned together)
  - By default this returns the SkuId, which is required for assigning the licence
  - Alternatively the SkuPartNumber can be returned, or just the subscription with dependent service plans instead

The complete function as at this date, is below:

<details>
  <summary><em><strong>Expand code block</strong> (always grab the latest version from GitHub)</em></summary>

```powershell
function Get-WTAzureADSubscriptionDependency {
    [CmdletBinding()]
    param (
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Azure AD subscription Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Azure AD subscription Graph permissions"
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
            HelpMessage = "Specify whether to exclude features in preview, a production API version will be used instead"
        )]
        [switch]$ExcludePreviewFeatures,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $true,
            HelpMessage = "The Azure AD subscriptions to check for dependencies"
        )]
        [Alias("Subscription", "subscribedSkus")]
        [PSCustomObject]$Subscriptions,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $false,
            HelpMessage = "The Azure AD subscription service plan objects with dependencies"
        )]
        [Alias("ServicePlan")]
        [PSCustomObject]$ServicePlans,
        [parameter(
            Mandatory = $false,
            ValueFromPipeLineByPropertyName = $true,
            ValueFromPipeLine = $false,
            HelpMessage = "Specify whether to return the required ServicePlan or skuPartNumber instead of the default skuId of subscriptions with dependencies"
        )]
        [ValidateSet("ServicePlan", "SkuPartNumber", "SkuId")]
        [string]$DependencyType = "skuId"
    )
    Begin {
        try {
            # Function definitions
            $Functions = @(
                "GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1",
                "GraphAPI\Public\AzureAD\Subscriptions\Get-WTAzureADSubscription.ps1"
            )

            # Function dot source
            foreach ($Function in $Functions) {
                . $Function
            }

            # Output current activity
            Write-Host "Getting Azure AD Commercial Subscription Dependencies"
        }
        catch {
            Write-Error -Message $_.Exception
            throw $_.exception
        }
    }
    Process {
        try {
            
            # If there are no subscriptions, get all subscriptions
            if (!$Subscriptions) {
                
                # If there is no access token, obtain one
                if (!$AccessToken) {
                    $AccessToken = Get-WTGraphAccessToken `
                        -ClientID $ClientID `
                        -ClientSecret $ClientSecret `
                        -TenantDomain $TenantDomain
                }
                if ($AccessToken) {
                
                    # Build Parameters
                    $Parameters = @{
                        AccessToken = $AccessToken
                    }
                    if ($ExcludePreviewFeatures) {
                        $Parameters.Add("ExcludePreviewFeatures", $true)
                    }

                    # Get Azure AD subscriptions with default properties
                    $Subscriptions = Get-WTAzureADSubscription @Parameters
                }
                else {
                    $ErrorMessage = "No access token specified, obtain an access token object from Get-WTGraphAccessToken"
                    Write-Error $ErrorMessage
                    throw $ErrorMessage
                }
            }
            if ($Subscriptions) {
                if ($ServicePlans) {
                    
                    # Output current activity
                    Write-Host "Evaluating Service Plans for subscriptions with dependencies"
                    
                    # Find subscriptions with dependencies
                    $DependentSubscriptionServicePlans = foreach ($Subscription in $Subscriptions) {
                        $RequiredServicePlans = $null
                        $RequiredServicePlans = foreach ($ServicePlan in $ServicePlans) {
                            if ($Subscription.servicePlans.servicePlanName -eq $ServicePlan.ServicePlanName) {
                                $ServicePlan.dependency.servicePlanName
                            }
                        }

                        # If there are dependencies, build object to return
                        if ($RequiredServicePlans) {
                            [PSCustomObject]@{
                                skuId                = $Subscription.skuId
                                skuPartNumber        = $Subscription.skuPartNumber
                                RequiredServicePlans = $RequiredServicePlans
                            }
                        }
                    }

                    # Find the skuPartNumbers with the dependent Service Plans for each subscription with dependencies
                    if ($DependencyType -eq "SkuPartNumber" -or $DependencyType -eq "SkuId") {
                        
                        # Output current activity
                        Write-Host "Evaluating SKUs containing Service Plans for subscriptions with dependencies"
                        
                        # Find the skuPartNumbers with the dependent Service Plans for each subscription with dependencies
                        $DependentSubscriptionSkus = foreach ($DependentSubscription in $DependentSubscriptionServicePlans) {
                            $RequiredSkus = foreach ($Subscription in $Subscriptions) {
                                foreach ($DependentSubscriptionServicePlan in $DependentSubscription.RequiredServicePlans) {
                                    if ($DependentSubscriptionServicePlan -in $Subscription.servicePlans.servicePlanName) {
                                        $Subscription.$DependencyType
                                    }
                                }
                            }
                            
                            if ($RequiredSkus) {
                                [PSCustomObject]@{
                                    skuId                     = $DependentSubscription.skuId
                                    skuPartNumber             = $DependentSubscription.skuPartNumber
                                    "Required$DependencyType" = $RequiredSkus
                                }
                            }
                            else {
                                $WarningMessage = "There are no SKUs containing Service Plans for subscriptions with dependencies"
                                Write-Warning $WarningMessage
                            }
                        }

                        # Return dependent subscription with required skuPartNumbers
                        if ($DependentSubscriptionSkus) {
                            $DependentSubscriptionSkus
                        }
                    }
                    elseif ($DependencyType -eq "servicePlan") {

                        # Return dependent subscription with required servicePlans
                        $DependentSubscriptionServicePlans
                    }
                }
                else {
                    $WarningMessage = "No Azure AD subscription service plans to check for dependencies"
                    Write-Warning $WarningMessage
                }
            }
            else {
                $WarningMessage = "No Azure AD subscriptions exist in Azure AD, or with parameters specified"
                Write-Warning $WarningMessage
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

## Assigning subscriptions to Azure AD groups
Assigning licences to groups is performed by creating an "assignLicense" group relationship.

This uses the [New-WTAzureADGroupRelationship][function-new] function, which you can access on my GitHub.

[For more information, see the Azure AD group relationship post here][assign]

[function-getsub]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Get-WTAzureADSubscription.ps1
[function-getsubdep]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Subscriptions/Get-WTAzureADSubscriptionDependency.ps1
[dep]: https://github.com/wesley-trust/GraphAPIConfig/tree/main/AzureAD/Subscriptions/Dependencies
[assign]: /blog/graph-api-groups-relationship/#create-azure-ad-group-relationships
[function-new]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/AzureAD/Groups/Relationship/New-WTAzureADGroupRelationship.ps1