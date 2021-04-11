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

For Endpoint Manager, I'll be defining App Protection policies for Android and iOS (including iPadOS) and Device Compliance policies for Windows 10. This is because for Android and iOS, these are mostly used for personal devices, and for Windows, it's more likely that they'll be corporate devices.

I apply these policies to my personal Azure AD tenant, so they apply to my iPad, Android phone and Windows laptop. I don't have a Mac...so I haven't created a policy for that...

- [App Protection for Android](#app-protection-for-android)
- [App Protection for iOS (including iPadOS)](#app-protection-for-ios-including-ipados)
- [Device Compliance for Windows 10](#device-compliance-for-windows-10)

These definitions are available in the [GraphAPIConfig][GraphAPIConfig] template repo in GitHub.

_The policy definitions include the policy type, such as 'androidManagedAppProtection'. This is not required to create a policy, however, when the policy type is not identified in the config, it must be defined within the Uri for the API call._

## App Protection for Android
This definition is available here: [REF-01][em-ref1], which you can access from my GitHub.

This sets some recommendations such as requiring app encryption, PIN (with minimum length) or biometric protection for apps, blocks rooted devices as well as access to data when offline for 12 hours.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
    "@odata.type": "#microsoft.graph.androidManagedAppProtection",
    "@odata.context": "https://graph.microsoft.com/beta/$metadata#deviceAppManagement/managedAppPolicies/$entity",
    "SVC": null,
    "REF": "01",
    "ENV": "P",
    "allowedAndroidDeviceManufacturers": null,
    "allowedAndroidDeviceModels": [],
    "allowedDataIngestionLocations": [
        "oneDriveForBusiness",
        "sharePoint",
        "camera"
    ],
    "allowedDataStorageLocations": [],
    "allowedInboundDataTransferSources": "allApps",
    "allowedOutboundClipboardSharingExceptionLength": 0,
    "allowedOutboundClipboardSharingLevel": "allApps",
    "allowedOutboundDataTransferDestinations": "allApps",
    "appActionIfAndroidDeviceManufacturerNotAllowed": "block",
    "appActionIfAndroidDeviceModelNotAllowed": "block",
    "appActionIfAndroidSafetyNetAppsVerificationFailed": "block",
    "appActionIfAndroidSafetyNetDeviceAttestationFailed": "block",
    "appActionIfDeviceComplianceRequired": "block",
    "appActionIfDeviceLockNotSet": "block",
    "appActionIfMaximumPinRetriesExceeded": "block",
    "appActionIfUnableToAuthenticateUser": null,
    "approvedKeyboards": [],
    "biometricAuthenticationBlocked": false,
    "blockAfterCompanyPortalUpdateDeferralInDays": 0,
    "blockDataIngestionIntoOrganizationDocuments": false,
    "contactSyncBlocked": false,
    "createdDateTime": "2021-04-08T14:17:18.1393133Z",
    "customBrowserDisplayName": "",
    "customBrowserPackageId": "",
    "customDialerAppDisplayName": "",
    "customDialerAppPackageId": "",
    "dataBackupBlocked": false,
    "deployedAppCount": 0,
    "description": "",
    "deviceComplianceRequired": true,
    "deviceLockRequired": false,
    "dialerRestrictionLevel": "allApps",
    "disableAppEncryptionIfDeviceEncryptionIsEnabled": false,
    "disableAppPinIfDevicePinIsSet": false,
    "displayName": "REF-01;ENV-P;VER-02; App Protection for Android",
    "encryptAppData": true,
    "exemptedAppPackages": [],
    "fingerprintBlocked": false,
    "id": "T_992343b3-1e73-4359-b80e-dc8f30559d3b",
    "isAssigned": false,
    "keyboardsRestricted": false,
    "lastModifiedDateTime": "2021-04-08T14:17:18.1393133Z",
    "managedBrowser": "notConfigured",
    "managedBrowserToOpenLinksRequired": false,
    "maximumAllowedDeviceThreatLevel": "notConfigured",
    "maximumPinRetries": 5,
    "maximumRequiredOsVersion": null,
    "maximumWarningOsVersion": null,
    "maximumWipeOsVersion": null,
    "minimumPinLength": 6,
    "minimumRequiredAppVersion": null,
    "minimumRequiredCompanyPortalVersion": null,
    "minimumRequiredOsVersion": null,
    "minimumRequiredPatchVersion": "0000-00-00",
    "minimumWarningAppVersion": null,
    "minimumWarningCompanyPortalVersion": null,
    "minimumWarningOsVersion": null,
    "minimumWarningPatchVersion": "0000-00-00",
    "minimumWipeAppVersion": null,
    "minimumWipeCompanyPortalVersion": null,
    "minimumWipeOsVersion": null,
    "minimumWipePatchVersion": "0000-00-00",
    "mobileThreatDefenseRemediationAction": "block",
    "notificationRestriction": "allow",
    "organizationalCredentialsRequired": false,
    "periodBeforePinReset": "PT0S",
    "periodOfflineBeforeAccessCheck": "PT12H",
    "periodOfflineBeforeWipeIsEnforced": "P90D",
    "periodOnlineBeforeAccessCheck": "PT30M",
    "pinCharacterSet": "numeric",
    "pinRequired": true,
    "pinRequiredInsteadOfBiometricTimeout": null,
    "previousPinBlockCount": 0,
    "printBlocked": false,
    "requiredAndroidSafetyNetAppsVerificationType": "none",
    "requiredAndroidSafetyNetDeviceAttestationType": "none",
    "requiredAndroidSafetyNetEvaluationType": "basic",
    "roleScopeTagIds": [
        "0"
    ],
    "saveAsBlocked": false,
    "screenCaptureBlocked": false,
    "simplePinBlocked": false,
    "targetedAppManagementLevels": "unspecified",
    "version": "\"c38a2c92-686a-407c-8b08-b8300ea42607\"",
    "warnAfterCompanyPortalUpdateDeferralInDays": 0,
    "wipeAfterCompanyPortalUpdateDeferralInDays": 0
}
```

</details>

## App Protection for iOS (including iPadOS)
This definition is available here: [REF-02][em-ref2], which you can access from my GitHub.

This sets some recommendations such as requiring app encryption, PIN (with minimum length) or biometric protection for apps, blocks jailbroken devices as well as access to data when offline for 12 hours.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
    "SVC":  null,
    "REF":  "02",
    "ENV":  "P",
    "@odata.context":  "https://graph.microsoft.com/beta/$metadata#deviceAppManagement/managedAppPolicies/$entity",
    "@odata.type":  "#microsoft.graph.iosManagedAppProtection",
    "allowedDataIngestionLocations":  [
                                          "oneDriveForBusiness",
                                          "sharePoint",
                                          "camera"
                                      ],
    "allowedDataStorageLocations":  [

                                    ],
    "allowedInboundDataTransferSources":  "allApps",
    "allowedIosDeviceModels":  null,
    "allowedOutboundClipboardSharingExceptionLength":  0,
    "allowedOutboundClipboardSharingLevel":  "allApps",
    "allowedOutboundDataTransferDestinations":  "allApps",
    "appActionIfDeviceComplianceRequired":  "block",
    "appActionIfIosDeviceModelNotAllowed":  "block",
    "appActionIfMaximumPinRetriesExceeded":  "block",
    "appActionIfUnableToAuthenticateUser":  null,
    "appDataEncryptionType":  "whenDeviceLocked",
    "blockDataIngestionIntoOrganizationDocuments":  false,
    "contactSyncBlocked":  false,
    "createdDateTime":  "2021-04-08T17:01:06.5512908Z",
    "customBrowserProtocol":  "",
    "customDialerAppProtocol":  "",
    "dataBackupBlocked":  false,
    "deployedAppCount":  0,
    "description":  "",
    "deviceComplianceRequired":  true,
    "dialerRestrictionLevel":  "allApps",
    "disableAppPinIfDevicePinIsSet":  true,
    "disableProtectionOfManagedOutboundOpenInData":  false,
    "displayName":  "REF-02;ENV-P;VER-02; App Protection for iOS",
    "exemptedAppProtocols":  [
                                 {
                                     "name":  "Default",
                                     "value":  "skype;app-settings;calshow;itms;itmss;itms-apps;itms-appss;itms-services;"
                                 }
                             ],
    "faceIdBlocked":  false,
    "filterOpenInToOnlyManagedApps":  false,
    "fingerprintBlocked":  false,
    "id":  "T_69e55462-5715-4b41-9128-4b67a76d4c64",
    "isAssigned":  false,
    "lastModifiedDateTime":  "2021-04-08T17:01:06Z",
    "managedBrowser":  "notConfigured",
    "managedBrowserToOpenLinksRequired":  false,
    "maximumAllowedDeviceThreatLevel":  "notConfigured",
    "maximumPinRetries":  5,
    "maximumRequiredOsVersion":  null,
    "maximumWarningOsVersion":  null,
    "maximumWipeOsVersion":  null,
    "minimumPinLength":  6,
    "minimumRequiredAppVersion":  null,
    "minimumRequiredOsVersion":  null,
    "minimumRequiredSdkVersion":  null,
    "minimumWarningAppVersion":  null,
    "minimumWarningOsVersion":  null,
    "minimumWipeAppVersion":  null,
    "minimumWipeOsVersion":  null,
    "minimumWipeSdkVersion":  null,
    "mobileThreatDefenseRemediationAction":  "block",
    "notificationRestriction":  "allow",
    "organizationalCredentialsRequired":  false,
    "periodBeforePinReset":  "PT0S",
    "periodOfflineBeforeAccessCheck":  "PT12H",
    "periodOfflineBeforeWipeIsEnforced":  "P90D",
    "periodOnlineBeforeAccessCheck":  "PT30M",
    "pinCharacterSet":  "numeric",
    "pinRequired":  true,
    "pinRequiredInsteadOfBiometricTimeout":  null,
    "previousPinBlockCount":  0,
    "printBlocked":  false,
    "protectInboundDataFromUnknownSources":  false,
    "roleScopeTagIds":  [
                            "0"
                        ],
    "saveAsBlocked":  false,
    "simplePinBlocked":  false,
    "targetedAppManagementLevels":  "unspecified",
    "thirdPartyKeyboardsBlocked":  false,
    "version":  "\"ca39e994-db9a-482f-a4de-7a6d3f069de2\""
}
```

</details>

## Device Compliance for Windows 10
This definition is available here: [REF-03][em-ref3], which you can access from my GitHub.

This sets some recommendations such as requiring device encryption and secure boot, requiring antivirus and the firewall to be enabled.

_The scheduledActionsForRule is required to create a policy, where one does not exist in the config, a default rule of blocking devices after 24 hours is created in the pipeline._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
    "SVC": null,
    "REF": "03",
    "ENV": "P",
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
```

</details>

[em-ref1]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/EndpointManager/AppManagement/Policies/ENV-P/REF-01%3BENV-P%3BVER-02%3B%20App%20Protection%20for%20Android.json
[em-ref2]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/EndpointManager/AppManagement/Policies/ENV-P/REF-02%3BENV-P%3BVER-02%3B%20App%20Protection%20for%20iOS.json
[em-ref3]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/EndpointManager/DeviceManagement/Policies/ENV-P/REF-03%3BENV-P%3BVER-02%3B%20Device%20Compliance%20for%20Windows%2010.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig