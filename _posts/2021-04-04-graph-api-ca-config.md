---
title: "Recommended Azure AD Conditional Access policies - GraphAPIConfig"
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
Within GitHub, I've created the [GraphAPIConfig][GraphAPIConfig] repo, which contains a set of baseline recommended configurations for the Graph API. _This is set up as a template, so you can duplicate this and modify as appropriate._

This repo covers recommended definitions for default Azure AD groups, Conditional Access and Endpoint Manager (Intune) policies.

All of which will be created as part of either the Conditional Access pipeline, or a related pipeline (Azure AD groups, Endpoint Manager etc). I'll be covering the design of these in a series of posts.

Firstly, to make full use of Conditional Access policies, there are dependencies of the following to consider:
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

These polices use the "beta" Microsoft Graph API (as at this date), as they make use of features in "Preview". Typically it's not recommended to use preview features in production, however, in this case, the alternative is to not make use of these features, so it's an acceptable risk.

### Recommended Azure AD Conditional Access policies
- [Block access, for all cloud apps, for any location, excluding trusted or named locations](#block-access-for-all-cloud-apps-for-any-location-excluding-trusted-or-named-locations)
- [Block access, for registering security information, for any location, excluding trusted or named locations, hybrid joined or compliant devices](#block-access-for-registering-security-information-for-any-location-excluding-trusted-or-named-locations-hybrid-joined-or-compliant-devices)
- [Block access, for guests, for all cloud apps, except approved apps](#block-access-for-guests-for-all-cloud-apps-except-approved-apps)
- [Block access, for all cloud apps, for all client apps supporting legacy authentication](#block-access-for-all-cloud-apps-for-all-client-apps-supporting-legacy-authentication)
- [Require MFA, for all cloud apps, for any location, excluding trusted locations, hybrid joined or compliant devices](#require-mfa-for-all-cloud-apps-for-any-location-excluding-trusted-locations-hybrid-joined-or-compliant-devices)
- [Require hybrid joined or compliant device, for all cloud apps, for all desktop devices](#require-hybrid-joined-or-compliant-device-for-all-cloud-apps-for-all-desktop-devices)

## Block access, for all cloud apps, for any location, excluding trusted or named locations
This definition is available here: [REF-01][policy-ref1], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users" automatically added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded (IE break-glass accounts)
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Locations: Include: Any location
- Locations: Exclude: Selected named locations: MFA Trusted IPs, United Kingdom, IPv6 and unknown

_It's important to include IPv6 and unknown locations, to reduce the chance that legitimate users will be blocked_
</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Block access
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This is used to restrict the ability to sign-in, to only locations that are trusted (such as an office), or named locations (such as the countries that users would be likely to sign-in from). Blocking all signs-ins from locations other than these, reducing the attack vector so it's less likely malicious sign-ins could occur.

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

## Block access, for registering security information, for any location, excluding trusted or named locations, hybrid joined or compliant devices
This definition is available here: [REF-02][policy-ref2], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users" automatically added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded (IE break-glass accounts)
#### Cloud apps or actions  <!-- omit in toc -->
- User action: Register security information
#### Conditions  <!-- omit in toc -->
- Locations: Include: Any location
- Locations: Exclude: Selected named locations: MFA Trusted IPs, United Kingdom, IPv6 and unknown
- Device state: Include: All device state
- Device state: Exclude: Device Hybrid Azure AD joined, Device marked as compliant

_It's important to include IPv6 and unknown locations, to reduce the chance that legitimate users will be blocked_
</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Block access
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This is used to restrict the ability to register security information (IE MFA registration), to only locations that are trusted (such as an office), or named locations (such as the countries that users would be likely to sign in from). Or devices that are already hybrid joined or compliant with security policies. 

Blocking all attempts from locations other than these, reducing the attack vector so it's less likely malicious registration could occur.

_This can be customised to restrict further._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "02",
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
    "clientApplications": null,
    "applications": {
      "includeApplications": [],
      "excludeApplications": [],
      "includeUserActions": [
        "urn:user:registersecurityinfo"
      ],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "0e9ab8c1-c538-446a-8ad9-0b2113d10ac7"
      ],
      "excludeGroups": [
        "b623e336-1aca-4837-a36d-c4feeb3d2a2d"
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
    },
    "devices": {
      "includeDeviceStates": [],
      "excludeDeviceStates": [],
      "includeDevices": [
        "All"
      ],
      "excludeDevices": [
        "Compliant",
        "DomainJoined"
      ],
      "deviceFilter": null
    }
  },
  "createdDateTime": "2021-03-19T20:10:31.5706787Z",
  "displayName": "REF-02;ENV-P;VER-2; Block access, for registering security information, for any location, excluding trusted or named locations, hybrid joined or compliant devices",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "block"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "50352922-ee14-4374-9378-2c275dce663b",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Block access, for guests, for all cloud apps, except approved apps
This definition is available here: [REF-03][policy-ref3], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Guests" automatically added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded (IE break-glass accounts)
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
- Cloud apps: Exclude: Office 365 Exchange Online, Office 365 SharePoint Online, Microsoft Planner, Microsoft Stream, Microsoft Teams
#### Conditions  <!-- omit in toc -->
- Client apps: Modern authentication clients: Browser, Mobile apps and desktop clients

_Other client apps will be blocked by another policy (IE disabling legacy authentication)_
</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Block access
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This is used to restrict the ability of guests and external users to only access approved cloud apps (such as collaboration apps). Blocking all attempts to access other cloud apps to increase security. Combining this with approved external domains for sharing within SharePoint Online will enhance security further.

_This can be customised to restrict or relax the apps included within the policy._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "03",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "browser",
      "mobileAppsAndDesktopClients"
    ],
    "platforms": null,
    "locations": null,
    "deviceStates": null,
    "devices": null,
    "clientApplications": null,
    "applications": {
      "includeApplications": [
        "All"
      ],
      "excludeApplications": [
        "00000002-0000-0ff1-ce00-000000000000",
        "00000003-0000-0ff1-ce00-000000000000",
        "00000004-0000-0ff1-ce00-000000000000",
        "09abbdfd-ed23-44ee-a2d9-a627aa1c90f3",
        "2634dd23-5e5a-431c-81ca-11710d9079f4",
        "cc15fd57-2c6c-4117-a88c-83b1d56b4bbe"
      ],
      "includeUserActions": [],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "44e33914-4a7e-46f8-85f7-c10f75c39fb0"
      ],
      "excludeGroups": [
        "51864daf-1d49-4d7c-bade-3aeea99c458c"
      ],
      "includeRoles": [],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-03-19T20:10:33.1289159Z",
  "displayName": "REF-03;ENV-P;VER-2; Block access, for guests, for all cloud apps, except approved apps",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "block"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "3fc2f375-20ba-4f8c-94c8-3aad88e2638d",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Block access, for all cloud apps, for all client apps supporting legacy authentication
This definition is available here: [REF-04][policy-ref4], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users" automatically added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded (IE break-glass accounts)
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Client apps: Legacy authentication clients: Exchange ActiveSync clients, Other clients

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Block access
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This is used to restrict the ability of users to only authenticate with modern authentication clients (which fully support Conditional Access, MFA and the latest security protocols). Blocking all attempts to access using legacy authentication clients.

_This can be customised to allow ActiveSync, but this is not recommended as app protection policies will not apply, lowering security._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "04",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "exchangeActiveSync",
      "other"
    ],
    "platforms": null,
    "locations": null,
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
        "e3560bd8-19a9-464a-a37b-61a05fa6fef7"
      ],
      "excludeGroups": [
        "d77bcccc-1a7e-4d71-80d3-64bbcbff2bfc"
      ],
      "includeRoles": [],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-03-19T20:10:35.0719324Z",
  "displayName": "REF-04;ENV-P;VER-2; Block access, for all cloud apps, for all client apps supporting legacy authentication",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "block"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "794fdafa-39cc-4219-9df3-7d3e804dd779",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require MFA, for all cloud apps, for any location, excluding trusted locations, hybrid joined or compliant devices
This definition is available here: [REF-05][policy-ref5], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users" automatically added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded (IE break-glass accounts)
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Locations: Include: Any location
- Locations: Exclude: All trusted locations
- Device state: Include: All device state
- Device state: Exclude: Device Hybrid Azure AD joined, Device marked as compliant

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require multi-factor authentication
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires that users are challenged for multi-factor authentication during sign-in for compatible client apps. Unless they're within a trusted location or have devices that are Hybrid Azure AD joined or marked as compliant.

As trusted locations are excluded, as well as specific devices, locations marked as trusted should be limited and devices that are Windows AD joined should be within a trust boundary (IE can only be joined by administrators or a trusted join workflow where security can be assured).

When used in combination with [REF-06](#require-hybrid-joined-or-compliant-device-for-all-cloud-apps-for-all-desktop-devices), common scenarios might include a Windows Virtual Desktop, or Windows Server mult-session environment, which are locked down within a cloud provider. When combined with Endpoint Manager, device security can be assured with a compliance policy which can be revoked.

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "05",
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
        "1217be06-fdb0-4ad4-bd2b-be95fa5cd39e"
      ],
      "excludeGroups": [
        "1e578ca6-5b86-48e1-ab8a-63dfea8b7374"
      ],
      "includeRoles": [],
      "excludeRoles": []
    },
    "locations": {
      "includeLocations": [
        "All"
      ],
      "excludeLocations": [
        "AllTrusted"
      ]
    },
    "devices": {
      "includeDeviceStates": [],
      "excludeDeviceStates": [],
      "includeDevices": [
        "All"
      ],
      "excludeDevices": [
        "Compliant",
        "DomainJoined"
      ],
      "deviceFilter": null
    }
  },
  "createdDateTime": "2021-03-19T20:10:36.8989474Z",
  "displayName": "REF-05;ENV-P;VER-2; Require MFA, for all cloud apps, for any location, excluding trusted locations, hybrid joined or compliant devices",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "mfa"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "457c0169-b8d3-4469-9c16-9415ebce5a52",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}

```

</details>

## Require hybrid joined or compliant device, for all cloud apps, for all desktop devices
This definition is available here: [REF-06][policy-ref6], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users" automatically added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded (IE break-glass accounts)
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Device platforms: Include: Any device
- Device platforms: Exclude: Android, iOS, Windows Phone
- Client apps: Modern authentication clients

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require device to be marked as compliant, Require Hybrid Azure AD joined device
_Require one of the selected controls_
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires that users are challenged for multi-factor authentication during sign-in for compatible client apps. Unless they're within a trusted location or have devices that are Hybrid Azure AD joined or marked as compliant.

As trusted locations are excluded, as well as specific devices, locations marked as trusted should be limited and devices that are Windows AD joined should be within a trust boundary (IE can only be joined by administrators or a trusted join workflow where security can be assured).

Common scenarios might include a Windows Virtual Desktop, or Windows Server mult-session environment, which are locked down within a cloud provider.

When combined with Endpoint Manager, device security can be assured with a compliance policy which can be revoked.

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "06",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "mobileAppsAndDesktopClients"
    ],
    "locations": null,
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
        "aa1ab765-496c-4920-871f-96835e961771"
      ],
      "excludeGroups": [
        "3c1b73af-a7a2-4670-8dbb-5827184cc849"
      ],
      "includeRoles": [],
      "excludeRoles": []
    },
    "platforms": {
      "includePlatforms": [
        "all"
      ],
      "excludePlatforms": [
        "android",
        "iOS",
        "windowsPhone"
      ]
    }
  },
  "createdDateTime": "2021-03-19T20:10:38.6351451Z",
  "displayName": "REF-06;ENV-P;VER-2; Require hybrid joined or compliant device, for all cloud apps, for all desktop devices",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "compliantDevice",
      "domainJoinedDevice"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "97458381-1ece-4bc6-8a0e-7bd00ea53ab5",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

[policy-ref1]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-01%3BENV-P%3BVER-2%3B%20Block%20access%2C%20for%20all%20cloud%20apps%2C%20for%20any%20location%2C%20excluding%20trusted%20or%20named%20locations.json
[policy-ref2]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-02%3BENV-P%3BVER-2%3B%20Block%20access%2C%20for%20registering%20security%20information%2C%20for%20any%20location%2C%20excluding%20trusted%20or%20named%20locations%2C%20hybrid%20joined%20or%20compliant%20devices.json
[policy-ref3]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-03%3BENV-P%3BVER-2%3B%20Block%20access%2C%20for%20guests%2C%20for%20all%20cloud%20apps%2C%20except%20approved%20apps.json
[policy-ref4]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-04%3BENV-P%3BVER-2%3B%20Block%20access%2C%20for%20all%20cloud%20apps%2C%20for%20all%20client%20apps%20supporting%20legacy%20authentication.json
[policy-ref5]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-05%3BENV-P%3BVER-2%3B%20Require%20MFA%2C%20for%20all%20cloud%20apps%2C%20for%20any%20location%2C%20excluding%20trusted%20locations%2C%20hybrid%20joined%20or%20compliant%20devices.json
[policy-ref6]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-06%3BENV-P%3BVER-2%3B%20Require%20hybrid%20joined%20or%20compliant%20device%2C%20for%20all%20cloud%20apps%2C%20for%20all%20desktop%20devices.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig