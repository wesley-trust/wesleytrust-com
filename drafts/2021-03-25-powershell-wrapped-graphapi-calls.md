---
title: "PowerShell wrapped Microsoft Graph API calls"
categories:
  - blog
tags:
  - graphapi
excerpt: "As I'm an infrastructure guy, rather than a developer, I'm using PowerShell as my scripting language to call the Graph API..."
---

## Open sesame
First we need to authenticate with the Graph API, as the code I'm writing is mostly intended to be executed in a pipeline, the authentication method needs to be non-interactive, so an Azure AD Service Principal is used to authenticate and gain an access token for the Graph API. The Service Principal is granted permissions to the Graph API, and this access token allows us to execute within the bounds of those permissions.

### Creating the Service Principal

### Granting permissions

### Obtaining an access token
