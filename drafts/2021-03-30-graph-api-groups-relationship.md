---
title: "Manage Azure AD groups Using the Graph API with PowerShell"
categories:
  - blog
tags:
  - graphapi
  - powershell
excerpt: "The first of the public functions is managing Azure AD groups, this is a dependency for the Conditional Access policies, so seems a good place to start..."
---
Managing Azure AD groups is a dependency for the Conditional Access policies, as for (almost) every policy we'll be creating both an inclusion and exclusion group, as well as adding members to the groups. I'll also be creating nested groups, such as dynamic ones including "All Users" and "All Guests".

This creates a complete solution that can be deployed in an Azure Pipeline. For the Azure AD groups, I'll be covering three main sections in this post, each section contains the public PowerShell functions to be called:

### Managing Azure AD group relationships
- Get-WTAzureADGroupRelationship
- New-WTAzureADGroupRelationship

## Managing Azure AD group relationships