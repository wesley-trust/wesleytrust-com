---
title: "Updated recommendations for Azure AD Conditional Access with resilience defaults"
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
excerpt: "Microsoft have been working on a 'backup authentication' mechanism for Azure AD for some time now, and with resilience defaults..."
---
Microsoft have been working on a ['backup authentication service'][msblog] for Azure AD for some time now, and with resilience defaults, we're seeing the first tangible sign of this.

Time to update the [recommended Azure AD Conditional Access policies][blog-policies]:

- [What is the 'backup authentication service', and why is it important?](#what-is-the-backup-authentication-service-and-why-is-it-important)
- [Why do we need to consider 'Resilience Defaults'?](#why-do-we-need-to-consider-resilience-defaults)
- [What are the changes to the recommended policies?](#what-are-the-changes-to-the-recommended-policies)
- [What are the potential repercussions?](#what-are-the-potential-repercussions)

## What is the 'backup authentication service', and why is it important? ##
The cause of many of Microsoft's biggest global outages in the past few years have been down to Azure AD. Whilst Azure AD has multiple redundant servers globally distributed, it has become a single point of failure for many Microsoft and third party services.

There was no "backup Azure AD" that could be switched to, so Microsoft created one.

As the scale and distribution of Azure AD is massive, creating such as a service wouldn't be easy, and design decisions, likely needed to make such a project feasible (at least in the short term), have meant a series of security compromises to consider. Leading to the introduction of 'resilience defaults' within Conditional Access.

## Why do we need to consider 'Resilience Defaults'? ##

Conditional Access is an important security feature of Azure AD, and people need to be confident that the policies configured, will be enforced. However, Microsoft have announced that when using the 'backup authentication service' (as a last resort in a BCDR situation), the service may not be able to effectively evaluate policies dependent on real, or near real-time data, or to guarantee all, or recent changes to policies are enforced.

In addition, Microsoft have said that by default, the failure to evaluate these conditions, would not result in blocking access, effectively meaning that it can no longer be guaranteed that polices are enforced in a disaster situation.

This sound scary, and a potential security risk, but this is a balancing act between security and productivity. Blocking access across the board, would result in significant disruption.

To address the security concern, Microsoft is allowing the ability to override this default, for specific, considered policies, such as ones that might be considered as having the highest risk of malicious inadvertent access.

## What are the changes to the recommended policies? ##

When evaluating policies to exclude from the ‘backup authentication service’, two access situations, high-privileged access, and low-privileged access must be considered, as well as the potential loss of risk analysis.

The recommended policies are designed to require conditions such as MFA by default, unless another condition, such as a trusted location, or hybrid-joined device is specified. So it's expected that low-privileged users would maintain a security baseline in a BCDR situation, from policies not dependent on real, or near real-time risk analysis.

Where the tables turn however, is high-privileged situations, such as users accessing the Azure Management portal, or users having Administrative rights. Due to the potential high security risk, but low productivity loss, two policies: REF-11 and REF-12 will be amended, so that access is blocked when the 'backup authentication service' is in use.

## What are the potential repercussions? ##

For normal, day-to-day user accounts, which should not have administrative rights (as a secondary admin account, or Privileged Identity Management should be in use), minimal productivity loss is expected, but users with administrative rights would be blocked access temporarily.

In the event that user accounts are compromised, the Azure Management portal would continue to be protected by blocking all access.

*Emergency Break-Glass Accounts, that are excluded from all policies, would not be affected, so any critical access would be maintained.*

Two other policies may not be enforced in an outage, REF-13, REF-14. This may result in access being granted without user and sign-in risk being evaluated (however, the alternative would be high productivity loss by blocking all users).

*The specific needs for including or excluding apps or users, when policies cannot be evaluated, is an important decision to evaluate.*

These changes are in preview, and are being tested. [Updated policies definitions][template] (which will be automated in a pipeline), will be available shortly.

[template]: https://github.com/wesley-trust/GraphAPIConfig/tree/main/AzureAD/ConditionalAccess/Policies/ENV-P
[blog-policies]: /blog/graph-api-ca-config/
[msblog]: https://techcommunity.microsoft.com/t5/azure-active-directory-identity/99-99-uptime-for-azure-active-directory/ba-p/1999628