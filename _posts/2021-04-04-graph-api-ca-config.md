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
Within GitHub, I've created a [baseline configuration template repo][GraphAPIConfig] for the Graph API, that can be used as recommended definitions for the groups, policies and named locations that will be created as part of the Conditional Access and related pipelines. I'll be covering the design of these in a series of posts.

Firstly, to make full use of Conditional Access policies, there are dependencies on the following to consider:
- Azure AD
  - Groups
  - Named Locations
- Endpoint Manager
  - Device Compliance
  - App Protection
- Exchange Online
  - App enforced restrictions
- SharePoint Online (including OneDrive)
  - App enforced restrictions

I'm going to cover each of the dependencies in their own series of posts, but today I'm going to cover the recommended Conditional Access baseline policies.

_These aren't intended to be fully exhaustive, they're supposed to be customised to suit individual needs and are intended to serve as a good starting point (that I use in my personal Azure AD tenant)._

### Recommended Azure AD Conditional Access policies
- [Block access, for all cloud apps, for any location, excluding trusted or named locations](#block-access-for-all-cloud-apps-for-any-location-excluding-trusted-or-named-locations)

## Block access, for all cloud apps, for any location, excluding trusted or named locations
This definition is available here: [REF-01][policy-ref1], which you can access from my GitHub.

This is used to restrict the ability to sign in to only locations that are trusted (such as an office), or named locations (such as the countries that users would be likely to sign in from), and block all signs ins from locations other than these. To reduce the chance of malicious sign in events.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "01",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "all"
    ],
    "platforms": null,
    "deviceStates": null,
    "devices": null,
    "clientApplications": null,
    "applications": {
      "includeApplications": [
        "All"
      ],
      "excludeApplications": [],
      "includeUserActions": [],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "9c6b6939-ded5-43ed-8d2a-70838fccb2ed"
      ],
      "excludeGroups": [
        "e7f5a73c-2029-4a34-8734-a21484843732"
      ],
      "includeRoles": [],
      "excludeRoles": []
    },
    "locations": {
      "includeLocations": [
        "All"
      ],
      "excludeLocations": [
        "00000000-0000-0000-0000-000000000000",
        "102afe99-db6a-49d1-bdb6-45f973812aaf"
      ]
    }
  },
  "createdDateTime": "2021-03-19T20:10:29.9780638Z",
  "displayName": "REF-01;ENV-P;VER-2; Block access, for all cloud apps, for any location, excluding trusted or named locations",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "block"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "f3dc1672-18a9-493e-9b6f-e50bda10c2cc",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

[policy-ref1]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-01%3BENV-P%3BVER-2%3B%20Block%20access%2C%20for%20all%20cloud%20apps%2C%20for%20any%20location%2C%20excluding%20trusted%20or%20named%20locations.json

[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig