---
title: "A series of baseline definitions for Azure AD groups - GraphAPIConfig"
categories:
  - blog
tags:
  - graphapi
  - graphapiconfig
  - powershell
  - groups
  - azuread
  - config
  - baseline
excerpt: "For the Azure AD Conditional Access and Endpoint Manager (Intune) policies, I'll be using a series of dependent groups to be used in the inclusion/exclusion groups..."
---
This post continues the coverage of the [GraphAPIConfig][GraphAPIConfig] repo, which contains a set of baseline recommended configurations for the Graph API. _This is set up as a template, so you can duplicate this and modify as appropriate. Please always grab the latest versions from GitHub._

There are six Azure AD groups that will be used within the Conditional Access and Endpoint Manager (Intune) pipelines, as nested groups that are included in the inclusion/exclusion groups that will be created for the Conditional Access and Endpoint Manager policies.

The current defined groups are:
- [All Users](#all-users)
- [All Guests](#all-guests)
- [All Devices](#all-devices)
- [All Windows Devices](#all-windows-devices)
- [SVC-CA; Exclude from all Conditional Access policies](#svc-ca-exclude-from-all-conditional-access-policies)
- [SVC-EM; Exclude from all Endpoint Manager device policies](#svc-em-exclude-from-all-endpoint-manager-device-policies)
- [SVC-EM; Exclude from all Endpoint Manager user policies](#svc-em-exclude-from-all-endpoint-manager-user-policies)

The definitions of these groups are available in the [GraphAPIConfig][GraphAPIConfig] template repo in GitHub. By defining these groups, rather than using the inbuilt "All Users" options within Conditional Access, allows for greater customisation of each policy.

_These are created with the PowerShell [Group][group-link] and [Group relationship][group-relationship-link] functions I wrote that use the Graph API, deployed in an [Azure DevOps pipeline][pipeline-link]._

_For each group, a mailNickName is required, when this does not exist, when executed in the pipeline it will be generated._

## All Users
This definition is available here: [All Users][group-users], which you can access from my GitHub.

This has a dynamic query that includes all users (including members and external users) within the Azure AD tenant.

Used to target Azure AD Conditional Access and Endpoint Manager App Protection policies.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Dynamic query that includes all users (including guests and external users) within the directory",
  "displayName": "All Users",
  "groupTypes": [
    "DynamicMembership"
  ],
  "mailEnabled": false,
  "membershipRule": "(user.objectId -ne null)",
  "membershipRuleProcessingState": "On",
  "securityEnabled": true,
}
```

</details>

## All Guests
This definition is available here: [All Guests][group-guests], which you can access from my GitHub.

This has a dynamic query that includes all guests (which is all external users excluding members) within the Azure AD tenant.

Used to target Azure AD Conditional Access policies.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Dynamic query that includes all quests (including external users) within the directory",
  "displayName": "All Guests",
  "groupTypes": [
    "DynamicMembership"
  ],
  "mailEnabled": false,
  "membershipRule": "(user.userType -ne \"member\")",
  "membershipRuleProcessingState": "On",
  "securityEnabled": true,
}
```

</details>

## All Devices
This definition is available here: [All Devices][group-devices], which you can access from my GitHub.

This has a dynamic query that includes all devices within the Azure AD tenant.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Dynamic query that includes all devices within the directory",
  "displayName": "All Devices",
  "groupTypes": [
    "DynamicMembership"
  ],
  "mailEnabled": false,
  "membershipRule": "(device.deviceId -ne null)",
  "membershipRuleProcessingState": "On",
  "securityEnabled": true,
}
```

</details>

## All Windows Devices
This definition is available here: [All Windows Devices][group-windows-devices], which you can access from my GitHub.

This has a dynamic query that includes all Windows devices within the Azure AD tenant.

Used to target Endpoint Manager Device Compliance for Windows policy.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Dynamic query that includes all Windows devices within the directory",
  "displayName": "All Windows Devices",
  "groupTypes": [
    "DynamicMembership"
  ],
  "mailEnabled": false,
  "membershipRule": "(device.deviceId -ne null) and (device.deviceOSType -eq \"Windows\")",
  "membershipRuleProcessingState": "On",
  "securityEnabled": true,
}
```

</details>

## SVC-CA; Exclude from all Conditional Access policies
This definition is available here: [SVC-CA; Exclude from all Conditional Access policies][group-exclude], which you can access from my GitHub.

This allows accounts to be added, such as break-glass accounts or others that should be excluded from all policies.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Contains the Break Glass accounts and any other account that should all be excluded from Conditional Access",
  "displayName": "SVC-CA; Exclude from all Conditional Access Policies",
  "mailEnabled": false,
  "securityEnabled": true,
}
```

</details>

## SVC-EM; Exclude from all Endpoint Manager device policies
This definition is available here: [SVC-EM; Exclude from all Endpoint Manager device policies][group-em-device-exclude], which you can access from my GitHub.

This allows accounts to be added, such as break-glass accounts or others that should be excluded from all policies.

_It's important to remember that for Endpoint Manager, you cannot mix users and devices in the same group._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Contains the Break Glass accounts and any other account that should all be excluded from Endpoint Manager",
  "displayName": "SVC-EM; Exclude from all Endpoint Manager Device Policies",
  "mailEnabled": false,
  "securityEnabled": true,
}
```

</details>

## SVC-EM; Exclude from all Endpoint Manager user policies
This definition is available here: [SVC-EM; Exclude from all Endpoint Manager user policies][group-em-user-exclude], which you can access from my GitHub.

This allows accounts to be added, such as break-glass accounts or others that should be excluded from all policies.

_It's important to remember that for Endpoint Manager, you cannot mix users and devices in the same group._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "description": "Contains the Break Glass accounts and any other account that should all be excluded from Endpoint Manager",
  "displayName": "SVC-EM; Exclude from all Endpoint Manager User Policies",
  "mailEnabled": false,
  "securityEnabled": true,
}
```

</details>

[group-users]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Users.json
[group-guests]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Guests.json
[group-exclude]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/SVC-CA/SVC-CA%3B%20Exclude%20from%20all%20Conditional%20Access%20Policies.json
[group-devices]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Devices.json
[group-windows-devices]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Windows%20Devices.json
[group-em-device-exclude]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/SVC-EM/SVC-EM%3B%20Exclude%20from%20all%20Endpoint%20Manager%20Device%20Policies.json
[group-em-user-exclude]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/SVC-EM/SVC-EM%3B%20Exclude%20from%20all%20Endpoint%20Manager%20User%20Policies.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig
[group-link]: https://www.wesleytrust.com/blog/graph-api-groups/
[group-relationship-link]: https://www.wesleytrust.com/blog/graph-api-groups-relationship/
[pipeline-link]: https://www.wesleytrust.com/blog/graph-api-groups-pipeline-validate/