---
title: "What is the Graph API?"
categories:
  - blog
tags:
  - graphapi
excerpt: "There are lots of references to this on the site and it's a project I've been working on, to make it easy to manage Azure AD resources as Infrastructure as Code..."
---

## What is the Microsoft Graph API?
Well, this is Microsoft's "API to rule them all", though, it's mostly used for integrating with Microsoft's SaaS services, such as Microsoft 365 as well as Azure AD. A large amount of Microsoft 365 services now use this API to exchange data, especially Microsoft Teams.

The great thing, is that this API allows for you to treat many of Microsoft's services as Infrastructure as Code, and can be used in a more declarative fashion, like Terraform or ARM templates, which people are used to when deploying Azure resources using the Microsoft ARM API (will we see this combined in the future??).

[More reference information on the API][graphapi-reference]

## How can it be used?
Services such as Azure AD Conditional Access policies can be defined in JSON files, and then using the correct Graph API method, it can Create, modify, or remove these policies in Azure AD. You can also do the same with Groups, Users, Named Locations, and lots more.

This opens big opportunities where you can have templated or baseline JSON files, managed in version control, and then combining this with a CI/CD pipeline, deploy these out, or update, or remove. Where you can treat the config files as your "desired state", and then have the pipeline maintain this state.

## How do you use it?

I've been a big fan of PowerShell, and over the years have written many scripts and functions, so it made logical sense to use this as my scripting language. So I've wrapped the Graph API calls in PowerShell functions, and then building on this, I used PowerShell to write the Pipeline logic, to determine what to create, update or remove.

In essence, there are 5 main components,

- CI/CD Pipeline
- PowerShell DSC logic to be executed in the pipeline
- PowerShell functions targeting each specific resource/service
- JSON config definitions of the target state
- PowerShell wrapped Microsoft Graph API calls to deploy

I first saw PowerShell wrapped API calls to deploy Conditional Access on [Daniel Chronlund's blog][daniel-blog], with that inspiration, I scaled this up into a full solution.

## Isn't there already a PowerShell module?

Yes, it's possible to deploy Conditional Access policies with a PowerShell module from Microsoft, but that is in a more imperative way, and I wanted to take a more modern declarative, desired state approach, based on my use of Terraform. Where following a change to a configuration in a repo, a pipeline would be executed that would Validate, Plan and [decide whether to] Deploy if there were changes that would need to be made (following approval).

As the Graph API is also standardised and well documented, it also means that with just a few tweaks to the code (or in some cases just a refactor), it's possible to deploy out an Azure AD group, instead of an Azure AD Conditional Access policy, with just minor changes, and reusing a lot of what I've already written.

So to start at the beginning, it's probably best I cover the PowerShell wrapped API functions first, which will be in my next post.

[graphapi-reference]: https://docs.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0
[daniel-blog]: https://danielchronlund.com/2020/11/25/how-to-manage-conditional-access-as-code-the-ultimate-guide/