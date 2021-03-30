---
title: "Manage Azure AD groups Using the Graph API with PowerShell"
categories:
  - blog
tags:
  - graphapi
  - powershell
excerpt: "The first of the public functions is managing Azure AD groups, this is a dependency for the Conditional Access policies, so seems a good place to start..."
---
Managing Azure AD groups is a dependency for the Conditional Access policies, as for (almost) every policy we'll be creating both an inclusion and exclusion group, as well as adding members to the groups. We'll also be creating nested groups, such as dynamic ones including "All Users" and "All Guests".

This is so we can have a full solution that we can deploy in an Azure Pipeline. For the Azure AD groups, I'll be covering three main sections, that each contain the public PowerShell functions to be called:

### Managing Azure AD groups
- Get-WTAzureADGroup
- Edit-WTAzureADGroup
- New-WTAzureADGroup
- Remove-WTAzureADGroup

### Exporting Azure AD groups config
- Export-WTAzureADGroup

### Managing Azure AD group relationships
- Get-WTAzureADGroupRelationship
- New-WTAzureADGroupRelationship

## Managing Azure AD groups
