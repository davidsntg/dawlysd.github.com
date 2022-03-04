---
layout: post
title:  "Azure Service Principal - 8 Best Practices"
author: davidsantiago
categories: [ azure, network, monitoring ]
image: assets/images/azure-service-principal-2.png
featured: true
---

Azure Service Principal (SP) management at scale can be time consuming for Cloud Center Of Excellence (CCOE) / Cloud Platforms. This article provides some tips & best practices to put in place.

# Azure Service Principal

**Definition from [Official documentation](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?toc=%2Fazure%2Fazure-resource-manager%2Ftoc.json&view=azure-cli-latest):**
> An Azure SP is a security identity used by user-created applications, services, and automation tools to access specific Azure resources. Think of it as a 'user identity' (username and password or certificate) with a specific role, and tightly controlled permissions. It only needs to be able to do specific things, unlike a general user identity. It improves security if you grant it only the minimum permissions level needed to perform its management tasks.

Most of the time, SPs are used by IT teams to automate Azure/Azure AD tasks in CI/CD pipelines and to [authenticate users](https://docs.microsoft.com/en-us/azure/developer/javascript/core/nodejs-sdk-azure-authenticate?tabs=azure-sdk-for-javascript).

# 8 Best Practices

## \#1 - Allow users to register SP by default

Any person or team that needs to automate something in Azure **should be able to create a SP on their own**.

*"On their own"* means be able to execute something like:
```bash
$ az ad sp create-for-rbac -n "AZ_SP_MYENTITY_MYAPPLICATION" --skip-assignement
```

To provide this self-service SP creation capability, *"Users can register applications"* must be set to *"Yes"* on User settings of Azure Active Directory:

  ![azure-service-principal]({{ site.baseurl }}/assets/images/azure-service-principal-1.png)

Now that everyone can create SPs, it is important to define naming conventions.

## \#2 - Define naming convention

There is not yet a "Naming Policy" for SPs as it already exists for [Azure AD groups](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/groups-naming-policy) for example.

It is therefore necessary to set up a "home-made" mechanism to detect deviations from a defined naming convention.

This mechanism can be developed based on [Azure Identities and Roles Governance Dashboard](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/azure-identities-and-roles-governance-dashboard-at-your/ba-p/3068613) project. This project provides many workbooks around Azure AD application credentials, roles ... 

It is possible to add a custom tab to existing "Azure AD and Azure Identities and Roles Overview Dashboard" workbook to check naming convention compliance (and even configure alerting).

# \#3 - Monitor secret expiration

SPs management at scale implies monitoring of SPs secrets. It is necessary to bring out **credentials about to expire**, **credentials not set to expire** and **credentials expired**.

[Azure Identities and Roles Governance Dashboard](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/azure-identities-and-roles-governance-dashboard-at-your/ba-p/3068613) project already provides this reporting capability:

  ![azure-service-principal]({{ site.baseurl }}/assets/images/azure-service-principal-2.png)

Rely on this project to go further and defined alerting looks interesting 😀.

In terms of security policy, monitor (and prohibit!) never-expire secrets is mandatory as well as 1 or 2 years secrets. 

**Secrets must be generated for a short period (3-6months) and renewed periodically**.

Delivery teams (and not CCOE) can simplify the secret renewal operation by adopting below process for a given SP:
* Store the SP credential in a Key Vault and define in Key Vault secret expiration date
* All automation that use the SP must fetch the secret from the Key Vault
* Create a small bot that will periodically regenerate SP secrets and update Key Vault SP secret based on expiration date

# \#4 - Two owners per SP, minimum

When a SP is created, the default owner is the creator. I would recommend as **good practice to always have at least 2 owners per SP** because there is always turnover in organizations.

Obviously, it is also interesting to **monitor the SPs which have less than two Owners** by relying on [Azure Identities and Roles Governance Dashboard](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/azure-identities-and-roles-governance-dashboard-at-your/ba-p/3068613) project and **create alerting**.

# \#5 - Monitor high privilege SPs

In all organizations, sooner or later, it is necessary to have one (or more) SPs with very high privileges on Azure and/or Azure AD.

These SPs :
- **must be identified** - can be achieved by relying on [Azure Identities and Roles Governance Dashboard](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/azure-identities-and-roles-governance-dashboard-at-your/ba-p/3068613) project.
- **sign-in logs must be review** and any **anomalies must be identified** - can be achieved by relying on [Sign-in logs in Azure Active Directory](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-all-sign-ins).
- **must be executed from a controlled and trusted environement** - it makes possible to set up Conditional Access (explained below) on this kind of very critical SP


# \#6 - Leverage Conditionnal Access for SPs

SP can sometimes be seen a high risk in term of security by security teams because **if SP credential leaks** (SP id, secret and tenant id) on GitHub for example, **anyone** (mostly robots) **anywhere can appropriate it** and start performing operations on 3rd tenants.

This is not a fictional scenario. It still happens to find SP credentials on GitHub and to have robots that use these SPs to instantiate infrastructure on Azure (and other CSP) to mine cryptocurrency.

A solution to avoid this risk is to set up the **[Conditional Access for workload identities](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/workload-identity)**. **This feature is still in Preview**, but I recommend using it especially on the most critical SPs (having high privileges). This implies to **control and master executions environments** for automation ran by critical SPs.

# \#7 - Monitor unused SPs

SPs lifecycle management is often overlooked. When a project in Azure is decommissioned, it is rare to see the SP associated with the decommissioned project as well.

A good practice here is to monitor **[SP sign-in logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-all-sign-ins)** and define processes to contact owners of an SP that has not logged in for 1 year.

# \#8 - 1 SP == 1 perimeter

In general, it is necessary to **limit the perimeters of possible actions for a SP**. Avoid having SPs that can do everything, both in Azure and Azure AD.

**Example of perimeters**:
* Delivery Team - Subscription PROD App A - Service Principal `AZ_SP_ENTITY_APP_PROD` will be Contributor to the subscription
* CCOE Team - Azure AD - Service Principal `AZ_SP_CCOE_AAD_PROD` will have Read/Write permissions on Azure AD
* etc...

This tip is a general idea: it must be contextualized on each Azure environment with discussions between Delivery team, Security team and CCOE team to define a global policy on this topic.

# Conclusion

Even if SPs can be seen as just robot accounts, managing them at scale for centralized teams can quickly become a nightmare.

I hope that these few hygiene rules will simplify this management, do not hesitate to contact me if you have any questions or comments.

### Aknowledgment

I would like to thank the people below for their advice and proofreading:
* [Remy Sabile](https://www.linkedin.com/in/r-sabile/)
* [Alexis Plantin](https://www.linkedin.com/in/alexis-plantin/)
* [Jean-Pierre Dussourd](https://www.linkedin.com/in/jpdussourd/)