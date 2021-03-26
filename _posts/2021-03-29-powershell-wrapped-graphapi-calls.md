---
title: "PowerShell wrapped functions to call the Microsoft Graph API"
categories:
  - blog
tags:
  - graphapi
excerpt: "For calling the Graph API, I wrote a series of private PowerShell functions, that public PowerShell functions will call to do the work..."
---
## Calling the Graph API
Calling an API at first can seem a bit daunting for an infrastructure guy, who may be thinking that APIs are just for developers, but using an object-orientated scripting language like PowerShell, can make it far less so. You can apply all the knowledge built up over the years within your comfort zone, and interact with APIs without having to learn another programming language.

For me, to simplify things I broke down the API calls into small specific functions, that would be in turn called by approved PowerShell verbs, with each corresponding to the method of the API call:

| API Method | PowerShell verb | Feature | Description                                |
|------------|-----------------|---------|--------------------------------------------|
| Get        | Get             | Get     | To get or list objects in the resource     |
| Patch      | Edit            | Update  | To update existing objects in the resource |
| Post       | New             | Create  | To create new objects in the resource      |
| Delete     | Remove          | Remove  | To remove objects in the resource          |

So this led to four method-specific private functions, that public functions will call, that are feature/service specific (IE Getting Azure AD groups).

These functions then call a generalised Microsoft Graph API Query function, so when public function call the API, they call based upon the method required.

Let's break these down.

### Private Functions
Private functions are not intended to be directly called, so I'll be covering more of a top level overview of what each do, rather than providing example usage.

#### Invoke-WTGraphQuery
The first function is [Invoke-WTGraphQuery][function-query], which you can access from my GitHub, this is a refactored version of one [Daniel][dan-blog] created.

This sets out the format in which the other private APIs will issue their calls.

#### Invoke-WTGraphGet

#### Invoke-WTGraphPatch

#### Invoke-WTGraphPost

#### Invoke-WTGraphDelete


[dan-blog]: https://danielchronlund.com/2020/11/26/azure-ad-conditional-access-policy-design-baseline-with-automatic-deployment-support/
[function-get]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphGet.ps1
[function-patch]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphPatch.ps1
[function-post]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphPost.ps1
[function-delete]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphDelete.ps1
[function-query]: https://github.com/wesley-trust/GraphAPI/blob/main/Private/Invoke-WTGraphQuery.ps1