---
title: "Endpoint Manager Compliance & App Protection - GraphAPIConfig"
categories:
  - blog
tags:
  - graphapi
  - graphapiconfig
  - powershell
  - named-locations
  - locations
  - azuread
  - config
excerpt: "Design of Compliance and App Protection policies for Endpoint Manager (Intune), that Conditional Access policies can then enforce..."
---
This post continues the coverage of the [GraphAPIConfig][GraphAPIConfig] repo, which contains a set of baseline recommended configurations for the Graph API. _This is set up as a template, so you can duplicate this and modify as appropriate. Please always grab the latest versions from GitHub._

For the Azure AD Conditional Access policies, I'll be defining named locations, that can be targeted within the policies. So for example, Azure AD sign-ins could be restricted to regions that typically users would be expected to sign-in from, lowering the chance of unexpected sign-in events (that could be malicious).

I've created the below definitions which have been useful for me, with common trade, travel areas and countries:
- [App Protection for Android](#app-protection-for-android)
- [App Protection for iOS & iPadOS](#app-protection-for-ios--ipados)
- [Device Compliance for Windows 10](#device-compliance-for-windows-10)

These definitions are available in the [GraphAPIConfig][GraphAPIConfig] template repo in GitHub.

## App Protection for Android
This definition is available here: [REF-01][em-ref1], which you can access from my GitHub.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#deviceAppManagement/androidManagedAppProtections",
    "value": [
        {
            "displayName": "REF-01;ENV-P;VER-02; App Protection for Android",
            "description": "",
            "createdDateTime": "2020-07-20T22:19:14.7672125Z",
            "lastModifiedDateTime": "2020-07-20T22:19:14.7672125Z",
            "roleScopeTagIds": [
                "0"
            ],
            "id": "T_807c2930-1765-4a70-aa13-c5527c6e1aa0",
            "version": "\"ac9daa58-421d-466d-b8f3-7061b3b67afb\"",
            "periodOfflineBeforeAccessCheck": "PT12H",
            "periodOnlineBeforeAccessCheck": "PT30M",
            "allowedInboundDataTransferSources": "allApps",
            "allowedOutboundDataTransferDestinations": "allApps",
            "organizationalCredentialsRequired": false,
            "allowedOutboundClipboardSharingLevel": "allApps",
            "dataBackupBlocked": false,
            "deviceComplianceRequired": true,
            "managedBrowserToOpenLinksRequired": false,
            "saveAsBlocked": false,
            "periodOfflineBeforeWipeIsEnforced": "P90D",
            "pinRequired": true,
            "maximumPinRetries": 5,
            "simplePinBlocked": false,
            "minimumPinLength": 6,
            "pinCharacterSet": "numeric",
            "periodBeforePinReset": "PT0S",
            "allowedDataStorageLocations": [],
            "contactSyncBlocked": false,
            "printBlocked": false,
            "fingerprintBlocked": false,
            "disableAppPinIfDevicePinIsSet": false,
            "maximumRequiredOsVersion": null,
            "maximumWarningOsVersion": null,
            "maximumWipeOsVersion": null,
            "minimumRequiredOsVersion": null,
            "minimumWarningOsVersion": null,
            "minimumRequiredAppVersion": null,
            "minimumWarningAppVersion": null,
            "minimumWipeOsVersion": null,
            "minimumWipeAppVersion": null,
            "appActionIfDeviceComplianceRequired": "block",
            "appActionIfMaximumPinRetriesExceeded": "block",
            "pinRequiredInsteadOfBiometricTimeout": null,
            "allowedOutboundClipboardSharingExceptionLength": 0,
            "notificationRestriction": "allow",
            "previousPinBlockCount": 0,
            "managedBrowser": "notConfigured",
            "maximumAllowedDeviceThreatLevel": "notConfigured",
            "mobileThreatDefenseRemediationAction": "block",
            "blockDataIngestionIntoOrganizationDocuments": false,
            "allowedDataIngestionLocations": [
                "oneDriveForBusiness",
                "sharePoint",
                "camera"
            ],
            "appActionIfUnableToAuthenticateUser": null,
            "dialerRestrictionLevel": "allApps",
            "isAssigned": true,
            "targetedAppManagementLevels": "unspecified",
            "screenCaptureBlocked": false,
            "disableAppEncryptionIfDeviceEncryptionIsEnabled": false,
            "encryptAppData": true,
            "deployedAppCount": 45,
            "minimumRequiredPatchVersion": "0000-00-00",
            "minimumWarningPatchVersion": "0000-00-00",
            "minimumWipePatchVersion": "0000-00-00",
            "allowedAndroidDeviceManufacturers": null,
            "appActionIfAndroidDeviceManufacturerNotAllowed": "block",
            "requiredAndroidSafetyNetDeviceAttestationType": "none",
            "appActionIfAndroidSafetyNetDeviceAttestationFailed": "block",
            "requiredAndroidSafetyNetAppsVerificationType": "none",
            "appActionIfAndroidSafetyNetAppsVerificationFailed": "block",
            "customBrowserPackageId": "",
            "customBrowserDisplayName": "",
            "minimumRequiredCompanyPortalVersion": null,
            "minimumWarningCompanyPortalVersion": null,
            "minimumWipeCompanyPortalVersion": null,
            "keyboardsRestricted": false,
            "allowedAndroidDeviceModels": [],
            "appActionIfAndroidDeviceModelNotAllowed": "block",
            "customDialerAppPackageId": "",
            "customDialerAppDisplayName": "",
            "biometricAuthenticationBlocked": false,
            "requiredAndroidSafetyNetEvaluationType": "basic",
            "blockAfterCompanyPortalUpdateDeferralInDays": 0,
            "warnAfterCompanyPortalUpdateDeferralInDays": 0,
            "wipeAfterCompanyPortalUpdateDeferralInDays": 0,
            "deviceLockRequired": false,
            "appActionIfDeviceLockNotSet": "block",
            "exemptedAppPackages": [],
            "approvedKeyboards": []
        }
    ]
}
```

</details>

## App Protection for iOS & iPadOS
This definition is available here: [REF-02][em-ref2], which you can access from my GitHub.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#deviceAppManagement/iosManagedAppProtections",
    "value": [
        {
            "displayName": "REF-02;ENV-P;VER-02; App Protection for iOS & iPadOS",
            "description": "",
            "createdDateTime": "2020-07-20T22:59:19.5998226Z",
            "lastModifiedDateTime": "2020-07-20T22:59:19.5998226Z",
            "roleScopeTagIds": [
                "0"
            ],
            "id": "T_481ce110-71cf-4407-926d-de146693e823",
            "version": "\"77e7caa4-af47-4d7d-8acd-109a4347ddc8\"",
            "periodOfflineBeforeAccessCheck": "PT12H",
            "periodOnlineBeforeAccessCheck": "PT30M",
            "allowedInboundDataTransferSources": "allApps",
            "allowedOutboundDataTransferDestinations": "allApps",
            "organizationalCredentialsRequired": false,
            "allowedOutboundClipboardSharingLevel": "allApps",
            "dataBackupBlocked": false,
            "deviceComplianceRequired": true,
            "managedBrowserToOpenLinksRequired": false,
            "saveAsBlocked": false,
            "periodOfflineBeforeWipeIsEnforced": "P90D",
            "pinRequired": true,
            "maximumPinRetries": 5,
            "simplePinBlocked": false,
            "minimumPinLength": 6,
            "pinCharacterSet": "numeric",
            "periodBeforePinReset": "PT0S",
            "allowedDataStorageLocations": [],
            "contactSyncBlocked": false,
            "printBlocked": false,
            "fingerprintBlocked": false,
            "disableAppPinIfDevicePinIsSet": true,
            "maximumRequiredOsVersion": null,
            "maximumWarningOsVersion": null,
            "maximumWipeOsVersion": null,
            "minimumRequiredOsVersion": null,
            "minimumWarningOsVersion": null,
            "minimumRequiredAppVersion": null,
            "minimumWarningAppVersion": null,
            "minimumWipeOsVersion": null,
            "minimumWipeAppVersion": null,
            "appActionIfDeviceComplianceRequired": "block",
            "appActionIfMaximumPinRetriesExceeded": "block",
            "pinRequiredInsteadOfBiometricTimeout": null,
            "allowedOutboundClipboardSharingExceptionLength": 0,
            "notificationRestriction": "allow",
            "previousPinBlockCount": 0,
            "managedBrowser": "notConfigured",
            "maximumAllowedDeviceThreatLevel": "notConfigured",
            "mobileThreatDefenseRemediationAction": "block",
            "blockDataIngestionIntoOrganizationDocuments": false,
            "allowedDataIngestionLocations": [
                "oneDriveForBusiness",
                "sharePoint",
                "camera"
            ],
            "appActionIfUnableToAuthenticateUser": null,
            "dialerRestrictionLevel": "allApps",
            "isAssigned": true,
            "targetedAppManagementLevels": "unspecified",
            "appDataEncryptionType": "whenDeviceLocked",
            "minimumRequiredSdkVersion": null,
            "deployedAppCount": 54,
            "faceIdBlocked": false,
            "minimumWipeSdkVersion": null,
            "allowedIosDeviceModels": null,
            "appActionIfIosDeviceModelNotAllowed": "block",
            "thirdPartyKeyboardsBlocked": false,
            "filterOpenInToOnlyManagedApps": false,
            "disableProtectionOfManagedOutboundOpenInData": false,
            "protectInboundDataFromUnknownSources": false,
            "customBrowserProtocol": "",
            "customDialerAppProtocol": "",
            "exemptedAppProtocols": [
                {
                    "name": "Default",
                    "value": "skype;app-settings;calshow;itms;itmss;itms-apps;itms-appss;itms-services;"
                }
            ]
        }
    ]
}
```

</details>

## Device Compliance for Windows 10
This definition is available here: [REF-03][em-ref3], which you can access from my GitHub.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#deviceManagement/deviceCompliancePolicies",
    "value": [
        {
            "@odata.type": "#microsoft.graph.windows10CompliancePolicy",
            "roleScopeTagIds": [
                "0"
            ],
            "id": "e8c8b379-af43-4c5c-8df0-a72b7276148d",
            "createdDateTime": "2020-07-07T15:08:20.0938467Z",
            "description": null,
            "lastModifiedDateTime": "2020-11-01T13:56:55.5928833Z",
            "displayName": "REF-03;ENV-P;VER-02; Device Compliance for Windows 10",
            "version": 6,
            "passwordRequired": true,
            "passwordBlockSimple": true,
            "passwordRequiredToUnlockFromIdle": false,
            "passwordMinutesOfInactivityBeforeLock": 15,
            "passwordExpirationDays": null,
            "passwordMinimumLength": 6,
            "passwordMinimumCharacterSetCount": null,
            "passwordRequiredType": "deviceDefault",
            "passwordPreviousPasswordBlockCount": null,
            "requireHealthyDeviceReport": false,
            "osMinimumVersion": null,
            "osMaximumVersion": null,
            "mobileOsMinimumVersion": null,
            "mobileOsMaximumVersion": null,
            "earlyLaunchAntiMalwareDriverEnabled": false,
            "bitLockerEnabled": true,
            "secureBootEnabled": true,
            "codeIntegrityEnabled": false,
            "storageRequireEncryption": true,
            "activeFirewallRequired": true,
            "defenderEnabled": true,
            "defenderVersion": null,
            "signatureOutOfDate": true,
            "rtpEnabled": true,
            "antivirusRequired": true,
            "antiSpywareRequired": true,
            "deviceThreatProtectionEnabled": false,
            "deviceThreatProtectionRequiredSecurityLevel": "unavailable",
            "configurationManagerComplianceRequired": false,
            "tpmRequired": true,
            "deviceCompliancePolicyScript": null,
            "validOperatingSystemBuildRanges": []
        }
    ]
}
```

</details>

[em-ref1]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/EndpointManager/AppProtection/Policies/ENV-P/REF-01%3BENV-P%3BVER-02%3B%20App%20Protection%20for%20Android.json
[em-ref2]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/EndpointManager/AppProtection/Policies/ENV-P/REF-02%3BENV-P%3BVER-02%3B%20App%20Protection%20for%20iOS%20%26%20iPadOS.json
[em-ref3]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/EndpointManager/DeviceCompliance/Policies/ENV-P/REF-03%3BENV-P%3BVER-02%3B%20Device%20Compliance%20for%20Windows%2010.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig