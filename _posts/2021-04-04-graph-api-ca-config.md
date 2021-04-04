---
title: "GraphAPIConfig - Recommended Azure AD Conditional Access policies"
categories:
  - blog
tags:
  - graphapi
  - graphapiconfig
  - powershell
  - conditional-access
  - azuread
  - config
  - baseline
  - recommendations
excerpt: "For Azure AD Conditional Access, I've put together a set of recommended baseline polices based on my experience and research..."
---
Within GitHub, I've created a [baseline configuration template repo][GraphAPIConfig], that can be used as recommended definitions for the groups, policies and named locations that will be created as part of the Conditional Access and related pipelines. I'll be covering the design of these in a series of posts.

Firstly, to make full use of Conditional Access policies, there are dependencies on the following:
- Azure AD
  - Groups
  - Named Locations
- Endpoint Manager
  - Device Compliance
  - App Protection

I'm going to cover each of the recommended baseline policies, and over the next series of posts, 

### Recommended Azure AD Conditional Access policies
- [All Users](#all-users)
- [All Guests](#all-guests)
- [SVC-CA; Exclude from all Conditional Access policies](#svc-ca-exclude-from-all-conditional-access-policies)

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

[group-users]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Users.json
[group-guests]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/All%20Guests.json
[group-exclude]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/Groups/SVC-CA/SVC-CA%3B%20Exclude%20from%20all%20Conditional%20Access%20Policies.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig