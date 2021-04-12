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
excerpt: "I've designed a set of recommended baseline policies for Azure AD Conditional Access based on my experience and research..."
---
Within GitHub, I've created the [GraphAPIConfig][GraphAPIConfig] repo, which contains a set of baseline recommended configurations for the Graph API. _This is set up as a template, so you can duplicate this and modify as appropriate. Please always grab the latest versions from GitHub._

This repo covers recommended definitions for default Azure AD groups, Conditional Access and Endpoint Manager (Intune) policies.

All of which will be created as part of either the Conditional Access pipeline, or a related pipeline (Azure AD groups, Endpoint Manager etc). I'll be covering the design of these in a series of posts.

Firstly, to make full use of Conditional Access policies, there are dependencies of the following to consider:
- Azure AD
  - [Groups][groups-config]
  - [Named Locations][location-config]
- Endpoint Manager
  - [Device Compliance][em-config]
  - [App Protection][em-config]
- Exchange Online
  - App enforced restrictions
- SharePoint Online (including OneDrive)
  - App enforced restrictions

I'm going to cover each of the dependencies in their own series of posts, but today I'm going to cover recommended Conditional Access baseline policies.

_These aren't intended to be fully exhaustive, they're supposed to be customised to suit individual needs and are intended to serve as a good starting point (that I use in my personal Azure AD tenant). Azure AD P1 licensing is required to use Conditional Access._

These policies use the "beta" Microsoft Graph API (as at this date), as they make use of features in "Preview". Typically it's not recommended to use preview features, or a preview API in production, however, in this case, the alternative is to not make use of these features, so I consider this an acceptable risk.

### Recommended Azure AD Conditional Access policies
- [Block access, for all cloud apps, for any location, excluding trusted or named locations](#block-access-for-all-cloud-apps-for-any-location-excluding-trusted-or-named-locations)
- [Block access, for registering security information, for any location, excluding trusted or named locations, hybrid joined or compliant devices](#block-access-for-registering-security-information-for-any-location-excluding-trusted-or-named-locations-hybrid-joined-or-compliant-devices)
- [Block access, for guests, for all cloud apps, except approved apps](#block-access-for-guests-for-all-cloud-apps-except-approved-apps)
- [Block access, for all cloud apps, for all client apps supporting legacy authentication](#block-access-for-all-cloud-apps-for-all-client-apps-supporting-legacy-authentication)
- [Require MFA, for all cloud apps, for any location, excluding trusted locations, hybrid joined or compliant devices](#require-mfa-for-all-cloud-apps-for-any-location-excluding-trusted-locations-hybrid-joined-or-compliant-devices)
- [Require hybrid joined or compliant device, for all cloud apps, for all desktop devices](#require-hybrid-joined-or-compliant-device-for-all-cloud-apps-for-all-desktop-devices)
- [Require approved client app or compliant device, for all cloud apps, for all mobile devices](#require-approved-client-app-or-compliant-device-for-all-cloud-apps-for-all-mobile-devices)
- [Require app protection policy or compliant device, for Exchange and SharePoint, for all mobile devices](#require-app-protection-policy-or-compliant-device-for-exchange-and-sharepoint-for-all-mobile-devices)
- [Require app-enforced restrictions, for Exchange and SharePoint, for all browsers, excluding hybrid joined or compliant devices](#require-app-enforced-restrictions-for-exchange-and-sharepoint-for-all-browsers-excluding-hybrid-joined-or-compliant-devices)
- [Require daily sign-in frequency with no persistent sessions, for all cloud apps, for all browsers, excluding hybrid joined or compliant devices](#require-daily-sign-in-frequency-with-no-persistent-sessions-for-all-cloud-apps-for-all-browsers-excluding-hybrid-joined-or-compliant-devices)
- [Require MFA, for administrators, for all cloud apps](#require-mfa-for-administrators-for-all-cloud-apps)
- [Require MFA, for Azure Management](#require-mfa-for-azure-management)
- [Require MFA, for risky sign-in events, for all cloud apps](#require-mfa-for-risky-sign-in-events-for-all-cloud-apps)
- [Require password change with MFA, for users at risk, for all cloud apps](#require-password-change-with-mfa-for-users-at-risk-for-all-cloud-apps)
- [Require MFA, for registering or joining devices](#require-mfa-for-registering-or-joining-devices)

## Block access, for all cloud apps, for any location, excluding trusted or named locations
This definition is available here: [REF-01][policy-ref1], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Locations: Include: Any location
- Locations: Exclude: Selected named locations: MFA Trusted IPs, British Isles Common Travel Area, IPv6 and unknown

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
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- User action: Register security information
#### Conditions  <!-- omit in toc -->
- Locations: Include: Any location
- Locations: Exclude: Selected named locations: MFA Trusted IPs, British Isles Common Travel Area, IPv6 and unknown
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
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Guests", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
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
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
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

_This can be customised to allow ActiveSync, but this is not recommended as app protection policies will not apply._

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
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
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
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
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
This requires that users must sign-in from a device Hybrid Azure AD joined or marked as compliant. Applying to modern authentication clients only as legacy authentication is already blocked by another policy. This is a 'catch all' policy, so all device platforms are targeted, with mobile platforms excluded as they will be targeted in a separate policy.

This means that for desktop operating systems, device based MDM will be the only option, rather than user based "App Protection" policies which for desktop operating systems isn't the most appropriate choice, as the majority will be corporate devices, than personally owned.

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

## Require approved client app or compliant device, for all cloud apps, for all mobile devices
This definition is available here: [REF-07][policy-ref7], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Device platforms: Include: Any device
- Device platforms: Exclude: Windows, macOS
- Client apps: Modern authentication clients

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require device to be marked as compliant, Require approved client app

_Require one of the selected controls_
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires that users must sign-in from a device marked as compliant, or use an approved client app. Applying to modern authentication clients only as legacy authentication is already blocked by another policy. This is a 'catch all' policy, so all device platforms are targeted, with desktop platforms excluded as they will be targeted in a separate policy.

This means that for mobile operating systems, there is a choice to enrol a device for MDM, typically used for corporate devices, where administrators will have full control of the device, or users must use an approved client app (where app protection policy can be applied), typically used for personal devices. This can be combined with policy [REF-08](#require-app-protection-policy-or-compliant-device-for-exchange-and-sharepoint-for-all-mobile-devices), to further restrict access to supported apps that have App Protection policies enforced.

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "07",
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
        "25ecb756-5ccc-4ba2-bab6-1e5fc8b47390"
      ],
      "excludeGroups": [
        "5ba862c8-507d-4a3f-b840-915ba909855e"
      ],
      "includeRoles": [],
      "excludeRoles": []
    },
    "platforms": {
      "includePlatforms": [
        "all"
      ],
      "excludePlatforms": [
        "windows",
        "macOS"
      ]
    }
  },
  "createdDateTime": "2021-03-19T20:10:40.214084Z",
  "displayName": "REF-07;ENV-P;VER-2; Require approved client app or compliant device, for all cloud apps, for all mobile devices",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "compliantDevice",
      "approvedApplication"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "6e9de2fc-30cd-40ea-a036-5ff013837bbc",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require app protection policy or compliant device, for Exchange and SharePoint, for all mobile devices
This definition is available here: [REF-08][policy-ref8], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: Office 365 Exchange Online, Office 365 SharePoint Online
#### Conditions  <!-- omit in toc -->
- Device platforms: Include: Any device
- Device platforms: Exclude: Windows, macOS
- Client apps: Modern authentication clients

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require device to be marked as compliant, Require app protection policy

_Require one of the selected controls_
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires that users must sign-in from a device marked as compliant, or use a client app with an app protection policy applied for when accessing Exchange or SharePoint. Applying to modern authentication clients only as legacy authentication is already blocked by another policy. This is a 'catch all' policy, so all device platforms are targeted, with desktop platforms excluded as they will be targeted in a separate policy.

This means that for mobile operating systems, there is a choice to enrol a device for MDM, typically used for corporate devices, where administrators will have full control of the device, or users must use a client app with an app protection policy applied, typically used for personal devices. This extends security for supported apps, above just requiring an approved app, which applies for apps that do not support this control.

_Only specific client apps support this setting, so it's recommended to not apply to "All cloud apps", [more info here][policy-ref8-applink]._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "08",
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
        "00000002-0000-0ff1-ce00-000000000000",
        "00000003-0000-0ff1-ce00-000000000000"
      ],
      "excludeApplications": [],
      "includeUserActions": [],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "f151ad51-a08f-4bf6-ba2d-4ecf1075ea93"
      ],
      "excludeGroups": [
        "864d3780-b0c6-43a4-8b32-53e33b5d9bd5"
      ],
      "includeRoles": [],
      "excludeRoles": []
    },
    "platforms": {
      "includePlatforms": [
        "all"
      ],
      "excludePlatforms": [
        "windows",
        "macOS"
      ]
    }
  },
  "createdDateTime": "2021-03-19T20:10:41.7933966Z",
  "displayName": "REF-08;ENV-P;VER-2; Require app protection policy or compliant device, for Exchange and SharePoint, for all mobile devices",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "compliantDevice",
      "compliantApplication"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "5b10dda8-3a78-4090-af59-4627fad1e561",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require app-enforced restrictions, for Exchange and SharePoint, for all browsers, excluding hybrid joined or compliant devices
This definition is available here: [REF-09][policy-ref9], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: Office 365 Exchange Online, Office 365 SharePoint Online
#### Conditions  <!-- omit in toc -->
- Client apps: Browser
- Device state: Include: All device state
- Device state: Exclude: Device Hybrid Azure AD joined, Device marked as compliant

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- None
#### Session  <!-- omit in toc -->
- Use app enforced restrictions
</details>

### What does this do? <!-- omit in toc -->
This requires that users have app-enforced restrictions (such as the inability to download data) when using a web browser accessing Exchange or SharePoint. Unless accessed using devices that are Hybrid Azure AD joined or marked as compliant.

_This requires configuration changes for Exchange Online and SharePoint Online to enforce restrictions, [more info here][policy-ref9-applink]._

_When configured in SharePoint Online, default Conditional Access policies are created and enabled, I remove these and replace with my recommended policies._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "09",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "browser"
    ],
    "platforms": null,
    "locations": null,
    "deviceStates": null,
    "clientApplications": null,
    "applications": {
      "includeApplications": [
        "00000002-0000-0ff1-ce00-000000000000",
        "00000003-0000-0ff1-ce00-000000000000"
      ],
      "excludeApplications": [],
      "includeUserActions": [],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "e0f518ce-ce0a-41f7-87cf-13d2d33aa896"
      ],
      "excludeGroups": [
        "e57028b1-8e6d-4cdf-8f04-88fa78a3a694"
      ],
      "includeRoles": [],
      "excludeRoles": []
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
  "createdDateTime": "2021-03-19T20:10:43.4104464Z",
  "displayName": "REF-09;ENV-P;VER-2; Require app-enforced restrictions, for Exchange and SharePoint, for all browsers, excluding hybrid joined or compliant devices",
  "grantControls": null,
  "id": "e9225761-c32b-4882-9c49-755508fdf427",
  "modifiedDateTime": null,
  "sessionControls": {
    "cloudAppSecurity": null,
    "signInFrequency": null,
    "persistentBrowser": null,
    "applicationEnforcedRestrictions": {
      "isEnabled": true
    }
  },
  "state": "disabled"
}
```

</details>

## Require daily sign-in frequency with no persistent sessions, for all cloud apps, for all browsers, excluding hybrid joined or compliant devices
This definition is available here: [REF-10][policy-ref10], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Client apps: Browser
- Device state: Include: All device state
- Device state: Exclude: Device Hybrid Azure AD joined, Device marked as compliant

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- None
#### Session  <!-- omit in toc -->
- Sign-in frequency: 1 day
- Persistent browser session: Never persistent
</details>

### What does this do? <!-- omit in toc -->
This requires that users must sign-in each day, with no persistent sessions allowed, when using a web browser. Unless accessed using devices that are Hybrid Azure AD joined or marked as compliant. This enhances security, so if users are using a shared or guest computer for instance, sign-in information is not retained, and must be refreshed on a daily basis. This will override the "remember sign-in information" tenant wide.

_This can be customised further depending on the required risk appetite and level of tolerable disruption to users._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "10",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "browser"
    ],
    "platforms": null,
    "locations": null,
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
        "fdcfdc2d-1107-4f3f-a035-bc618bdb476b"
      ],
      "excludeGroups": [
        "5983bd07-f078-4f95-a0e5-501641a52093"
      ],
      "includeRoles": [],
      "excludeRoles": []
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
  "createdDateTime": "2021-03-19T20:10:45.4417413Z",
  "displayName": "REF-10;ENV-P;VER-2; Require daily sign-in frequency with no persistent sessions, for all cloud apps, for all browsers, excluding hybrid joined or compliant devices",
  "grantControls": null,
  "id": "ebbd0414-7ac5-4b79-8d36-b456cce0817d",
  "modifiedDateTime": null,
  "sessionControls": {
    "applicationEnforcedRestrictions": null,
    "cloudAppSecurity": null,
    "signInFrequency": {
      "value": 1,
      "type": "days",
      "isEnabled": true
    },
    "persistentBrowser": {
      "mode": "never",
      "isEnabled": true
    }
  },
  "state": "disabled"
}
```

</details>

## Require MFA, for administrators, for all cloud apps
This definition is available here: [REF-11][policy-ref11], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Directory roles, all administrator roles are selected (so must be updated as new roles are added)
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- None

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require multi-factor authentication
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires multi-factor authentication for all users holding administrator roles, irregardless of the app they are attempting to access, location or device. This enhances security as these users hold privileged roles and so every sign-in attempt is treated with high security. This encourages users to not hold privileged roles permanently and instead to make use of alternatives such as Privileged Identity Management.

_The roles included can be customised to suit individual needs._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "11",
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
      "includeGroups": [],
      "excludeGroups": [
        "53a7352b-c546-47dd-9880-3bca0aa36534"
      ],
      "includeRoles": [
        "fe930be7-5e62-47db-91af-98c3a49a38b1",
        "69091246-20e8-4a56-aa4d-066075b2a7a8",
        "baf37b3a-610e-45da-9e62-d9d1e5e8914b",
        "75941009-915a-4869-abe7-691bff18279e",
        "f28a1f50-f6e7-4571-818b-6a12f2af6b6c",
        "f023fd81-a637-4b56-95fd-791ac0226033",
        "194ae4cb-b126-40b2-bd5b-6091b380977d",
        "0964bb5e-9bdb-4d7b-ac29-58e794862a40",
        "e8611ab8-c189-46e8-94e1-60213ab1f814",
        "7be44c8a-adaf-4e2a-84d6-ab2649e08a13",
        "11648597-926c-4cf3-9c36-bcebb0ba8dcc",
        "a9ea8996-122f-4c74-9520-8edcd192826c",
        "966707d0-3269-4727-9be2-8c3a10f19b9d",
        "2b745bdf-0803-4d80-aa65-822c4493daac",
        "d37c8bed-0711-4417-ba38-b4abe66ce4c2",
        "4d6ac14f-3453-41d0-bef9-a3e0c569773a",
        "74ef975b-6605-40af-a5d2-b9539d836353",
        "3a2c62db-5318-420d-8d74-23affee5d9d5",
        "8ac3fc64-6eca-42ea-9e69-59f4c7b60eb2",
        "729827e3-9c14-49f7-bb1b-9608f156bbb8",
        "fdd7a751-b60b-444a-984c-02652fe8fa1c",
        "62e90394-69f5-4237-9190-012177145e10",
        "be2f45a1-457d-42af-a067-6ec1fa63bc45",
        "0f971eea-41eb-4569-a71e-57bb8a3eff1e",
        "6e591065-9bad-43ed-90f3-e9424366d2f0",
        "29232cdf-9323-42fd-ade2-1d097af3e4de",
        "44367163-eba1-44c3-98af-f5787879f96a",
        "38a96431-2bdf-4b4c-8b6e-5d3d8abac1a4",
        "b1be1c3e-b65d-4f19-8427-f6fa0d97feb9",
        "e6d1a23a-da11-4be4-9570-befc86d067a7",
        "17315797-102d-40b4-93e0-432062caca18",
        "7698a772-787b-4ac8-901f-60d6b08affd2",
        "158c047a-c907-4556-b7ef-446551a6b5f7",
        "b0f54661-2d74-4c50-afa3-1ec803f12efe",
        "3edaf663-341e-4475-9f94-5c398ef6c070",
        "aaf43236-0c0d-4d5f-883a-6955382ac081",
        "7495fdc4-34c4-4d15-a289-98788ce399fd",
        "e3973bdf-4987-49ae-837a-ba8e231c7286",
        "c4e39bd9-1100-46d3-8c65-fb160da0071f",
        "9b895d92-2cd3-44c7-9d02-a6ac2d5ea5c3",
        "644ef478-e28f-4e28-b9dc-3fdde9aa0b1f",
        "c430b396-e693-46cc-96f3-db01bf8bb62a",
        "0526716b-113d-4c15-b2c8-68e3c22b9f80",
        "8329153b-31d0-4727-b945-745eb3bc5f31",
        "eb1f4a8d-243a-41f0-9fbd-c7cdf6c5ef7c",
        "b5a8dcf3-09d5-43a9-a639-8e29ef291470",
        "3d762c5a-1b6c-493f-843e-55a3b42923d4"
      ],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-03-19T20:10:46.9810516Z",
  "displayName": "REF-11;ENV-P;VER-2; Require MFA, for administrators, for all cloud apps",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "mfa"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "a3c22764-df26-412f-82c3-b3d9bcdbab63",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require MFA, for Azure Management
This definition is available here: [REF-12][policy-ref12], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: Microsoft Azure Management
#### Conditions  <!-- omit in toc -->
- None

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require multi-factor authentication
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires multi-factor authentication for all users accessing Microsoft Azure Management. This is a sensitive application as this allows the ability to view, change or remove Azure resources (with the correct permissions on the resource) which does not require an Azure AD role, and so management is protected.

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "12",
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
    "locations": null,
    "deviceStates": null,
    "devices": null,
    "clientApplications": null,
    "applications": {
      "includeApplications": [
        "797f4846-ba00-4fd7-ba43-dac1f8f63013"
      ],
      "excludeApplications": [],
      "includeUserActions": [],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "8a9e3089-a6f2-4ce7-8376-d367c9a80315"
      ],
      "excludeGroups": [
        "75e2b7ce-45d6-4fd2-a88f-24e2711ab456"
      ],
      "includeRoles": [],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-03-19T20:10:48.5904465Z",
  "displayName": "REF-12;ENV-P;VER-2; Require MFA, for Azure Management",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "mfa"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "c8107af6-4fd4-47d5-b52a-5608c43ee2d7",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require MFA, for risky sign-in events, for all cloud apps
This definition is available here: [REF-13][policy-ref13], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- Sign-in risk: Medium, High

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require multi-factor authentication
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires multi-factor authentication for sign-in events that Azure AD Identity Protection has determined to be at a medium or high risk level. This allows these users to be challenged with MFA when another policy would not have triggered this. Enhancing security as well as reducing user frustration by not requiring MFA for every authentication workflow and instead just for ones with the highest risk.

_This can be customised to increase or decrease the risk appetite for triggering MFA._

_A user without MFA configured, will not be allowed to configure MFA when at risk, so it's important that users have this configured in advance._

_This uses Azure AD Identity Protection, so requires Azure AD P2 licensing._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "13",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [],
    "signInRiskLevels": [
      "high",
      "medium"
    ],
    "clientAppTypes": [
      "all"
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
        "09497c93-b02b-41c5-acf9-2fa24f1c61df"
      ],
      "excludeGroups": [
        "34ffdbb1-0807-4c2c-8b8c-6892f6ac2103"
      ],
      "includeRoles": [],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-03-19T20:10:50.463577Z",
  "displayName": "REF-13;ENV-P;VER-2; Require MFA, for risky sign-in events, for all cloud apps",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "mfa"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "0671dc87-ed32-44f7-80c4-b8d75d4e492e",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require password change with MFA, for users at risk, for all cloud apps
This definition is available here: [REF-14][policy-ref14], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- Cloud apps: Include: All cloud apps
#### Conditions  <!-- omit in toc -->
- User risk: High

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require multi-factor authentication, Require password change

_Require all of the selected controls_
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires multi-factor authentication for users Azure AD Identity Protection has determined to be at a high risk level of being compromised (such as through leaked credentials). This allows these users to be challenged with MFA, which if successful, requires that they change their password. Enhancing security by requiring automatically that users with compromised accounts are identified with action taken.

_This can be customised to increase or decrease the risk appetite for triggering the password change and also whether the user is blocked instead._

_A user without MFA configured, will not be allowed to configure MFA when at risk, so it's important that users have this configured in advance._

_This uses Azure AD Identity Protection, so requires Azure AD P2 licensing._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "14",
  "VER": "2",
  "ENV": "P",
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/policies/$entity",
  "conditions": {
    "userRiskLevels": [
      "high"
    ],
    "signInRiskLevels": [],
    "clientAppTypes": [
      "all"
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
        "9049a02d-ca8b-4770-824f-d58512853d72"
      ],
      "excludeGroups": [
        "5379848a-ba03-4087-a068-e923fddd6969"
      ],
      "includeRoles": [],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-03-19T20:10:52.1294078Z",
  "displayName": "REF-14;ENV-P;VER-2; Require password change with MFA, for users at risk, for all cloud apps",
  "grantControls": {
    "operator": "AND",
    "builtInControls": [
      "mfa",
      "passwordChange"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "4a270024-e039-4cdd-8f31-be112cdab10e",
  "modifiedDateTime": null,
  "sessionControls": null,
  "state": "disabled"
}
```

</details>

## Require MFA, for registering or joining devices
This definition is available here: [REF-15][policy-ref15], which you can access from my GitHub.

<details>
  <summary><em><strong>Assignments</strong></em></summary>

#### Users & Groups  <!-- omit in toc -->
- Inclusion: Group created by the pipeline, with the dynamic nested group "All Users", added
- Exclusion: Group created by the pipeline, with the nested group containing all accounts to be excluded, added
#### Cloud apps or actions  <!-- omit in toc -->
- User action: Register or join devices
#### Conditions  <!-- omit in toc -->
- None

</details>

<details>
  <summary><em><strong>Access controls</strong></em></summary>

#### Grant  <!-- omit in toc -->
- Grant access: Require multi-factor authentication
#### Session  <!-- omit in toc -->
- None
</details>

### What does this do? <!-- omit in toc -->
This requires multi-factor authentication for when users register or join their devices to Azure AD. When enrolling a device in Endpoint Manager (Intune) or using an App with an "App Protection" policy applied, the device is registered in Azure AD, and so would also be required to complete MFA.

_This replaces the tenant wide setting, allowing customisation, such as including location based exclusions._

Example below:
<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "REF": "15",
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
    "locations": null,
    "deviceStates": null,
    "devices": null,
    "clientApplications": null,
    "applications": {
      "includeApplications": [],
      "excludeApplications": [],
      "includeUserActions": [
        "urn:user:registerdevice"
      ],
      "includeAuthenticationContextClassReferences": []
    },
    "users": {
      "includeUsers": [],
      "excludeUsers": [],
      "includeGroups": [
        "1a00c4a2-e3c9-4d9e-a91e-2e69f828a2e6"
      ],
      "excludeGroups": [
        "c61e4a0b-56c8-4cc6-96ec-0878f929f56c"
      ],
      "includeRoles": [],
      "excludeRoles": []
    }
  },
  "createdDateTime": "2021-04-04T12:54:42.2651265Z",
  "displayName": "REF-15;ENV-P;VER-2; Require MFA, for registering or joining devices",
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "mfa"
    ],
    "customAuthenticationFactors": [],
    "termsOfUse": []
  },
  "id": "c9c3eb48-2d32-4014-aae7-aa613b529d07",
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
[policy-ref7]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-07%3BENV-P%3BVER-2%3B%20Require%20approved%20client%20app%20or%20compliant%20device%2C%20for%20all%20cloud%20apps%2C%20for%20all%20mobile%20devices.json
[policy-ref8]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-08%3BENV-P%3BVER-2%3B%20Require%20app%20protection%20policy%20or%20compliant%20device%2C%20for%20Exchange%20and%20SharePoint%2C%20for%20all%20mobile%20devices.json
[policy-ref8-applink]: https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-grant#require-app-protection-policy
[policy-ref9]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-09%3BENV-P%3BVER-2%3B%20Require%20app-enforced%20restrictions%2C%20for%20Exchange%20and%20SharePoint%2C%20for%20all%20browsers%2C%20excluding%20hybrid%20joined%20or%20compliant%20devices.json
[policy-ref9-applink]: https://docs.microsoft.com/en-gb/azure/active-directory/conditional-access/concept-conditional-access-session#application-enforced-restrictions
[policy-ref10]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-10%3BENV-P%3BVER-2%3B%20Require%20daily%20sign-in%20frequency%20with%20no%20persistent%20sessions%2C%20for%20all%20cloud%20apps%2C%20for%20all%20browsers%2C%20excluding%20hybrid%20joined%20or%20compliant%20devices.json
[policy-ref11]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-11%3BENV-P%3BVER-2%3B%20Require%20MFA%2C%20for%20administrators%2C%20for%20all%20cloud%20apps.json
[policy-ref12]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-12%3BENV-P%3BVER-2%3B%20Require%20MFA%2C%20for%20Azure%20Management.json
[policy-ref13]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-13%3BENV-P%3BVER-2%3B%20Require%20MFA%2C%20for%20risky%20sign-in%20events%2C%20for%20all%20cloud%20apps.json
[policy-ref14]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-14%3BENV-P%3BVER-2%3B%20Require%20password%20change%20with%20MFA%2C%20for%20users%20at%20risk%2C%20for%20all%20cloud%20apps.json
[policy-ref15]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/ConditionalAccess/Policies/ENV-P/REF-15%3BENV-P%3BVER-2%3B%20Require%20MFA%2C%20for%20registering%20or%20joining%20devices.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig
[groups-config]: https://www.wesleytrust.com/blog/graph-api-groups-config/
[em-config]: https://www.wesleytrust.com/blog/graph-api-em-config/
[location-config]: https://www.wesleytrust.com/blog/graph-api-locations-config/