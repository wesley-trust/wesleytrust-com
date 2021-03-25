---
title: "Obtain an Access Token for the Microsoft Graph API"
categories:
  - blog
tags:
  - graphapi
excerpt: "As I'm an infrastructure guy, rather than a developer, I'm using PowerShell as my scripting language to obtain a Graph API access token..."
---

## Open sesame
First we need to authenticate with the Graph API. As the code I'm writing is mostly intended to be executed in a pipeline, the authentication method needs to be non-interactive, so an Azure AD Application Service Principal is used to authenticate and gain an access token for the Graph API.

The Service Principal is granted permissions to the Graph API, and the access token allows us to execute within the bounds of the permissions granted.

## Creating the Service Principal
There are lots of ways to create a service principal, and Microsoft's website has detailed information on the options including using the [user interface][gui-service-principal], or via [PowerShell][ps-service-principal], I won't repeat the options here, as no doubt Microsoft will change something and they'll become out of date within minutes.

You'll need all 3 of these to get an access token:
- Client ID (App ID)
- Tenant domain (Azure AD initial onmicrosoft.com domain)
- Client secret

### Granting permissions
There are different Graph API permissions that need to be granted to the service principal, depending on what you intent to do. The Graph API documentation includes the specific permissions you need, for each of the resources, and for each activity against the resource.

So for example,

If you want to "Get", the Azure AD "Conditional Access" "Policies", the "Application" permissions you will need are [listed here][ca-get], which is:
- "Policy.Read.All".

For the CI/CD Conditional Access pipeline, I'll need to get, create, update and delete, which requires a lot more permissions, and under each of the methods within the [Graph API documentation][ca], each permissions is listed:
- Policy.Read.All
- Policy.ReadWrite.ConditionalAccess
- Application.Read.All

I'll also need to grant permissions for each resource dependency, as the pipeline will also create the Azure AD [Groups][group] for the Conditional Access policies, and the Azure AD [Named Locations][nl], so these additional permissions are required:
- GroupMember.Read.All
- Group.Read.All
- Directory.Read.All
- Group.ReadWrite.All
- Directory.ReadWrite.All

## Obtaining an access token
Now that we have a service principal with the correct permissions, we need to obtain an access token to authenticate with the Graph API.

This uses the [Get-WTGraphAccessToken][getaccesstoken], which you can access from my GitHub, this is a refactored version of one [Daniel][dan-blog] created. I always build pipeline support in my functions, to you can pipe in the parameters too. Examples below:

```
# Clone repo that contains the Graph API functions
git clone --branch main --single-branch https://github.com/wesley-trust/GraphAPI.git

# Dot source function into memory
. .\GraphAPI\Public\Authentication\Get-WTGraphAccessToken.ps1

# Define Variables
$ClientID = "sdg23497-sd82-983s-sdf23-dsf234kafs24"
$ClientSecret = "khsdfhbdfg723498345_sdfkjbdf~-SDFFG1"
$TenantDomain = "wesleytrustsandbox.onmicrosoft.com"

# Create hashtable
$ServicePrincipal = @{
  ClientID     = $ClientID
  ClientSecret = $ClientSecret
  TenantDomain = $TenantDomain
}

# Pipe the hashtable to the function, and return access token to variable
$AccessToken = [PSCustomObject]$ServicePrincipal | Get-WTGraphAccessToken

# Or splat the hashtable of parameters
$AccessToken = Get-WTGraphAccessToken @ServicePrincipal

# Or specify each parameter individually
$AccessToken = Get-WTGraphAccessToken -ClientID $ClientID -ClientSecret $ClientSecret -TenantDomain $TenantDomain

```
### What does this do?

This invokes a REST method against the Microsoft Authentication service, for the Graph API resource, using the service principal parameters supplied for the Azure AD tenant. When executed, it returns an access token we can use to make requests against the Graph API (which must be periodically renewed).

The complete function as at this date, is below, but please make sure you get the latest version from GitHub:

```
function Get-WTGraphAccessToken {
    [cmdletbinding()]
    param (
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client ID for the Azure AD service principal with Conditional Access Graph permissions"
        )]
        [string]$ClientID,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "Client secret for the Azure AD service principal with Conditional Access Graph permissions"
        )]
        [string]$ClientSecret,
        [parameter(
            Mandatory = $true,
            ValueFromPipeLineByPropertyName = $true,
            HelpMessage = "The initial domain (onmicrosoft.com) of the tenant"
        )]
        [string]$TenantDomain
    )
    Begin {
        try {

            # Variables
            $Method = "Post"
            $AuthenticationUrl = "https://login.microsoft.com"
            $ResourceUrl = "https://graph.microsoft.com"
            $GrantType = "client_credentials"
            $Uri = "oauth2/token?api-version=1.0"
            
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
            
            # Compose and invoke REST request
            $Body = @{
                grant_type    = $GrantType
                resource      = $ResourceUrl
                client_id     = $ClientID
                client_secret = $ClientSecret
            }
            
            $OAuth2 = Invoke-RestMethod `
                -Method $Method `
                -Uri $AuthenticationUrl/$TenantDomain/$Uri `
                -Body $Body

            # If an access token is returned, return this
            if ($OAuth2.access_token) {
                $OAuth2.access_token
            }
            else {
                $ErrorMessage = "Unable to obtain an access token for $TenantDomain but an exception has not occurred"
                Write-Error $ErrorMessage
            }
        }
        catch {
            $ErrorMessage = "Unable to obtain an access token for $TenantDomain, an exception has occurred which may have more information"
            Write-Error $ErrorMessage
            Write-Error -Message $_.Exception
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
[gui-service-principal]: https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal
[ps-service-principal]: https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-authenticate-service-principal-powershell
[ca-get]: https://docs.microsoft.com/en-us/graph/api/conditionalaccesspolicy-get?view=graph-rest-1.0&tabs=http
[ca]: https://docs.microsoft.com/en-us/graph/api/resources/conditionalaccesspolicy?view=graph-rest-1.0
[nl]: https://docs.microsoft.com/en-us/graph/api/resources/namedlocation?view=graph-rest-1.0
[group]: https://docs.microsoft.com/en-us/graph/api/resources/groups-overview?view=graph-rest-1.0
[getaccesstoken]: https://github.com/wesley-trust/GraphAPI/blob/main/Public/Authentication/Get-WTGraphAccessToken.ps1
[dan-blog]: https://danielchronlund.com/2020/11/26/azure-ad-conditional-access-policy-design-baseline-with-automatic-deployment-support/