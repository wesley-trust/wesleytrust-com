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
excerpt: "For the Azure AD Conditional Access pipeline, I'll be using a series of dependent groups to be used in the inclusion/exclusion groups..."
---
Within GitHub, I've created a [baseline configuration template repo][GraphAPIConfig], that can be used as recommended definitions for the groups, policies and named locations that will be created as part of the Conditional Access pipeline. I'll be covering each in a series of posts, including the design of the policies.

Initially, there are three Azure AD groups that will be used within the Conditional Access pipeline, as nested groups that are included in the inclusion/exclusion groups that will be created for (almost) every Conditional Access policy.

The current groups are:
- [All Users](#all-users)
- [All Guests](#all-guests)
- [All Devices](#all-devices)
- [SVC-CA; Exclude from all Conditional Access policies](#svc-ca-exclude-from-all-conditional-access-policies)
- [SVC-EM; Exclude from all Endpoint Manager device policies](#svc-em-exclude-from-all-endpoint-manager-device-policies)
- [SVC-EM; Exclude from all Endpoint Manager user policies](#svc-em-exclude-from-all-endpoint-manager-user-policies)

The definitions of these groups are available in the [GraphAPIConfig][GraphAPIConfig] template repo in GitHub. By defining these groups, rather than using the inbuilt "All Users" options within Conditional Access, allows for greater customisation of each policy.

_For each group, a mailNickName is required, when this does not exist, when executed in the pipeline it will be generated._

## All Users
This definition is available here: [All Users][group-users], which you can access from my GitHub.

This has a dynamic query that includes all users (including members and external users) within the Azure AD tenant.

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

This allows accounts to be added, such as break-glass accounts or others that should be excluded from all policies.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
TO DO
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

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
TODO
```

</details>

## SVC-EM; Exclude from all Endpoint Manager user policies
This definition is available here: [SVC-EM; Exclude from all Endpoint Manager user policies][group-em-user-exclude], which you can access from my GitHub.

This allows accounts to be added, such as break-glass accounts or others that should be excluded from all policies.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
TODO
```

</details>

[group-users]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Users.json
[group-guests]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Guests.json
[group-exclude]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/SVC-CA/SVC-CA%3B%20Exclude%20from%20all%20Conditional%20Access%20Policies.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig