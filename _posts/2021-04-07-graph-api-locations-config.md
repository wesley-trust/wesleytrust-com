---
title: "Location, location, Azure AD Named Location - GraphAPIConfig"
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
excerpt: "For the Azure AD Conditional Access policies, I'll be defining named locations, that can be targeted within the policies..."
---
This post continues the coverage of the [GraphAPIConfig][GraphAPIConfig] repo, which contains a set of baseline recommended configurations for the Graph API. _This is set up as a template, so you can duplicate this and modify as appropriate. Please always grab the latest versions from GitHub._

For the Azure AD Conditional Access policies, I'll be defining named locations, that can be targeted within the policies. So for example, Azure AD sign-ins could be restricted to regions that typically users would be expected to sign-in from, lowering the chance of unexpected sign-in events (that could be malicious).

It's important to remember, that a location policy, such as this, could be expected to be quite broad, as users could be legitimately signing-in from multiple locations, or have their location not be identified (such as if via IPv6).

So it's important to combine policies such as MFA protection for administrators, for administrative actions, or sign-in or user risk to also reduce malicious sign-in events.

_Within Azure AD, the countries and regions are defined with their two-letter format specified by ISO 3166-2._

I've created the below definitions which have been useful for me, with common trade, travel areas and countries:
- [British Isles Common Travel Area](#british-isles-common-travel-area)
- [European Schengen Area, Gibraltar](#european-schengen-area-gibraltar)
- [European Economic Area](#european-economic-area)
- [European Free Trade Area](#european-free-trade-area)
- [European Union](#european-union)
- [Trans-Tasman Travel Arrangement](#trans-tasman-travel-arrangement)
- [United Kingdom](#united-kingdom)
- [United States](#united-states)

These definitions (and more country definitions) are available in the [GraphAPIConfig][GraphAPIConfig] template repo in GitHub.

## British Isles Common Travel Area
This definition is available here: [REF-01][location-ref1], which you can access from my GitHub.

This contains the UK, Ireland, Isle of Man, Jersey and Guernsey who all share a Common Travel Area (CTA).

_Often combined with the [SA, Gibraltar](#european-schengen-area-gibraltar), [EEA](#european-economic-area) and [EFTA](#european-free-trade-area) to target the majority of Europe._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": "01",
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "GB",
    "IE",
    "IM",
    "JE",
    "GG"
  ],
  "createdDateTime": "2021-03-19T16:34:42.012456Z",
  "displayName": "REF-01; British Isles Common Travel Area, IPv6 and Unknown",
  "id": "102afe99-db6a-49d1-bdb6-45f973812aaf",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-03-19T16:34:42.012456Z"
}
```

</details>

## European Schengen Area, Gibraltar
This definition is available here: [REF-02][location-ref2], which you can access from my GitHub.

This contains all members of the Schengen Area (SA), as well as de facto members: Monaco, San Marino and the Vatican City as well as Gibraltar.

_Often combined with the [CTA](#british-isles-common-travel-area), [EEA](#european-economic-area) and [EFTA](#european-free-trade-area) to target the majority of Europe._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": "02",
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "BG",
    "CZ",
    "DK",
    "DE",
    "EE",
    "GR",
    "ES",
    "FR",
    "IT",
    "LV",
    "LT",
    "LU",
    "HU",
    "MT",
    "NL",
    "AT",
    "PL",
    "PT",
    "SI",
    "SK",
    "FI",
    "SE",
    "IS",
    "LI",
    "NO",
    "CH",
    "SM",
    "MC",
    "VA",
    "GI"
  ],
  "createdDateTime": "2021-04-07T17:00:29.0646195Z",
  "displayName": "REF-02; European Schengen Area, Gibraltar, IPv6 and unknown",
  "id": "1a464618-4117-4814-acd2-49430ea52ae1",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T17:00:29.0646195Z"
}
```

</details>

## European Economic Area
This definition is available here: [REF-03][location-ref3], which you can access from my GitHub.

This contains all members of the European Union as well as Iceland, Liechtenstein and Norway, forming the EEA.

_Often used when targeting EU GDPR data protection residency, that applies to countries within the EEA, which are outside the [EU](#european-union) but not [EFTA](#european-free-trade-area). This also is typically combined with [UK](#united-kingdom) for UK GDPR data protection residency._

_Often combined with the [CTA](#british-isles-common-travel-area), [SA, Gibraltar](#european-schengen-area-gibraltar) and [EFTA](#european-free-trade-area) to target the majority of Europe._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": "03",
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "BE",
    "BG",
    "CZ",
    "DK",
    "DE",
    "EE",
    "IE",
    "GR",
    "ES",
    "FR",
    "HR",
    "IT",
    "CY",
    "LV",
    "LT",
    "LU",
    "HU",
    "MT",
    "NL",
    "AT",
    "PL",
    "PT",
    "RO",
    "SI",
    "SK",
    "FI",
    "SE",
    "IS",
    "LI",
    "NO"
  ],
  "createdDateTime": "2021-04-07T14:42:58.5561438Z",
  "displayName": "REF-03; European Economic Area, IPv6 and unknown",
  "id": "189ca541-390a-4a36-843e-d6ee76c45b2b",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T14:42:58.5561438Z"
}
```

</details>

## European Free Trade Area
This definition is available here: [REF-04][location-ref4], which you can access from my GitHub.

This contains EEA members: Iceland, Liechtenstein and Norway and non-EEA member: Switzerland, forming the EFTA.

_Often combined with the [CTA](#british-isles-common-travel-area), [SA, Gibraltar](#european-schengen-area-gibraltar) and [EEA](#european-economic-area) to target the majority of Europe._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": "04",
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "IS",
    "LI",
    "NO",
    "CH"
  ],
  "createdDateTime": "2021-04-07T14:43:00.2575629Z",
  "displayName": "REF-04; European Free Trade Area, IPv6 and unknown",
  "id": "185c3f18-c730-4500-a023-4f57ca1456ea",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T14:43:00.2575629Z"
}
```

</details>

## European Union
This definition is available here: [REF-05][location-ref5], which you can access from my GitHub.

This contains all members of the European Union (EU) only.

_This may not typically be used, as [SA, Gibraltar](#european-schengen-area-gibraltar) or [EEA](#european-economic-area) may be more appropriate._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": "05",
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "BE",
    "BG",
    "CZ",
    "DK",
    "DE",
    "EE",
    "IE",
    "GR",
    "ES",
    "FR",
    "HR",
    "IT",
    "CY",
    "LV",
    "LT",
    "LU",
    "HU",
    "MT",
    "NL",
    "AT",
    "PL",
    "PT",
    "RO",
    "SI",
    "SK",
    "FI",
    "SE"
  ],
  "createdDateTime": "2021-04-07T14:43:02.3787977Z",
  "displayName": "REF-05; European Union, IPv6 and unknown",
  "id": "13f4c7af-8f08-4b97-9d62-8cf21e6e521d",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T14:43:02.3787977Z"
}
```

</details>

## Trans-Tasman Travel Arrangement
This definition is available here: [REF-06][location-ref6], which you can access from my GitHub.

This contains Australia and New Zealand who have a travel agreement between both countries.

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": "06",
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "AU",
    "NZ"
  ],
  "createdDateTime": "2021-04-07T17:44:34.6135093Z",
  "displayName": "REF-06; Trans-Tasman Travel Arrangement, IPv6 and Unknown",
  "id": "12a1a810-77cc-4050-862d-caed0dca1b56",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T17:44:34.6135093Z"
}
```

</details>

## United Kingdom
This definition is available here: [UK][location-uk], which you can access from my GitHub.

This contains the United Kingdom of Great Britain and Northern Ireland only, using the two-digit country code "GB".

_Often used for UK GDPR data protection residency and combined with [EEA](#european-economic-area) for EU GDPR data protection residency._

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": null,
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "GB"
  ],
  "createdDateTime": "2021-04-07T17:00:27.1836021Z",
  "displayName": "United Kingdom, IPv6 and Unknown",
  "id": "1f090c87-d633-4118-bf6f-3f18a8e9be30",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T17:00:27.1836021Z"
}
```

</details>

## United States
This definition is available here: [US][location-ref6], which you can access from my GitHub.

This contains the United States of America only, using the two-digit country code "US".

Example below:

<details>
  <summary><em><strong>Expand code block</strong></em></summary>

```json
{
  "SVC": null,
  "REF": null,
  "ENV": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#identity/conditionalAccess/namedLocations/$entity",
  "@odata.type": "#microsoft.graph.countryNamedLocation",
  "countriesAndRegions": [
    "US"
  ],
  "createdDateTime": "2021-04-07T14:43:07.2557482Z",
  "displayName": "United States, IPv6 and unknown",
  "id": "10643880-d240-4e26-ba7e-49630255e53b",
  "includeUnknownCountriesAndRegions": true,
  "modifiedDateTime": "2021-04-07T14:43:07.2557482Z"
}
```

</details>

[location-ref1]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/REF-01%3B%20British%20Isles%20Common%20Travel%20Area%2C%20IPv6%20and%20Unknown.json
[location-ref2]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/REF-02%3B%20European%20Schengen%20Area%2C%20Gibraltar%2C%20IPv6%20and%20unknown.json
[location-ref3]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/REF-03%3B%20European%20Economic%20Area%2C%20IPv6%20and%20unknown.json
[location-ref4]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/REF-04%3B%20European%20Free%20Trade%20Area%2C%20IPv6%20and%20unknown.json
[location-ref5]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/REF-05%3B%20European%20Union%2C%20IPv6%20and%20unknown.json
[location-ref6]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/REF-06%3B%20Trans-Tasman%20Travel%20Arrangement%2C%20IPv6%20and%20Unknown.json
[location-uk]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/United%20Kingdom%2C%20IPv6%20and%20unknown.json
[location-us]: https://github.com/wesley-trust/GraphAPIConfig/blob/main/AzureAD/NamedLocations/United%20States%2C%20IPv6%20and%20unknown.json
[GraphAPIConfig]: https://github.com/wesley-trust/GraphAPIConfig