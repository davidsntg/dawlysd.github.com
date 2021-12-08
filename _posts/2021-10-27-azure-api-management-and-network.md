---
layout: post
title:  "Azure API Management & Network"
author: davidsantiago
categories: [ azure, network ]
image: assets/images/azure-apim-scenario2.png
---

The intent of this article is to deep dive on Azure API Management network integration possibilities with two scenarios described below.

Let's review what Azure API Management is and is not first.

# Azure API Management

![AzureAPIManagementServicesOverview]({{ site.baseurl }}/assets/images/azure-apim-overview.png)

Azure API Management (APIM) is a PaaS service that helps organizations publish APIs to external, partner, and internal developers to unlock the potential of their data and services. 

It includes a built-in gateway service that brokers network API calls to your backend so that you may enforce user-based access, take advantage of quotas and rate limiting, cache responses, and more. 

Key concepts of Azure APIM are available [here](https://docs.microsoft.com/en-us/azure/api-management/api-management-key-concepts). 

## Network 

From the networking prospective, Azure APIM must be deployed in a dedicated vnet/subnet. This technique is called VNet Injection and it uses Azure’s automation and SDN capabilities to deploy a given PaaS service directly into a specific customer VNet that has been configured with a special, delegated subnet for this purpose. This VNet is typically an extension of the customer’s on-premises private network by way of VPN, ExpressRoute, or both.

Regarding Azure APIM, there is [two access options](https://docs.microsoft.com/en-us/azure/api-management/virtual-network-concepts?tabs=stv2#access-options):

### External

The APIM endpoints (developer portal, API gateway ...) are accessible from the public internet via an external load balancer. The gateway can access resources within the VNet.
  ![azure-apim-external]({{ site.baseurl }}/assets/images/azure-apim-external.png)

### Internal

The APIM endpoints (developer portal, API gateway ...) are accessible only from within the VNet via an internal load balancer. The gateway can access resources withing the VNet.
  ![azure-apim-internal]({{ site.baseurl }}/assets/images/azure-apim-internal.png)

Use APIM in internal mode to:
* Make APIs hosted in your private datacenter securely accessible by third parties by using Azure VPN connections or Azure ExpressRoute.
* Enable hybrid cloud scenarios by exposing your cloud-based APIs and on-premises APIs through a common gateway.
* Manage your APIs hosted in multiple geographic locations, using a single gateway endpoint.

# Scenarios

Suppose a company that wants to deploy Azure APIM and has the following requirements:
* Existing APIs are currently exposed on to the **Internet** (all over the world because it is an international company) or on **Azure Private VNets**. Azure APIM must be able to reach all existing APIs.
* Existing APIs are critical: they take orders & cash! It is mandatory for Azure APIM to be deployed on multiple regions over the world and to have automatic failover in case of regional disaster
* Latency between APIs consumers and Azure APIM should be as low as possible
* Security is not an option: WAF and DDos protections are required.
* APIs consumers can be partners on the Internet or applications in the company corporate network

Based on the previous requirements, I imagined two scenarios:
* **Scenario #1** - Azure APIM External (*All Endpoints will be exposed on to the Internet*) with Azure Front Door in front (DSA, load balancing between APIM multi-locations gateways, Waf policy).
* **Scenario #2** - Azure APIM Internal (*All Endpoints will be exposed internally in an Azure Vnet*) with Application Gateways in front (Internet Exposition of Azure APIM, Waf) and Traffic Manager in front (DNS-based traffic load balancer).

We will review them below but let's talk about High Availability & Azure APIM before.

## High Availability considerations

Azure APIM being critical, it is interesting to deploy a primary instance in a region (West Europe for example bellow) and then [deploy additional locations](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-deploy-multi-region#-deploy-api-management-service-to-an-additional-location) (with [multiple AZ](https://docs.microsoft.com/en-us/azure/api-management/zone-redundancy) to increase overall SLA per region).

Another solution can be to deploy multiple Azure APIM in several regions but the deployment process become complicated because it is necessary to deploy APIs on each Azure APIM.

Here is a comparison table of two Azure APIM deployment options:
  ![azure-apim-deploymentoptions]({{ site.baseurl }}/assets/images/azure-apim-deploymentoptions.png)

For the scenarios described below, I chose to use the Azure APIM Premium with multiple locations solution.

## Scenario \#1 - External

On this Scenario #1, I decided to deploy Azure APIM in External mode (Gateway & other Azure APIM endpoints are accessible from the Internet).

Azure APIM can access APIs exposed on the Internet (Azure Functions or other APIs on the Internet...) but also internally (Azure VNets with peering, "on-premise" with S2S VPN or Express Route).

To fit security constraints, it is interesting to put an Azure Front Door between API consumers and the Azure APIM.

It gives the following network diagram:

[Azure Front Door](https://docs.microsoft.com/en-us/azure/frontdoor/) is a global, scalable entry-point that uses the Microsoft global edge network to create fast, secure, and widely scalable web applications. It supports dynamic site acceleration (DSA), **TLS/SSL offloading and end to end TLS, Web Application Firewall**, cookie-based session affinity, url path-based routing, free certificates and multiple domain management, and others. It provides **built-in DDoS protection and application layer security and caching**.

  ![azure-apim-scenario1]({{ site.baseurl }}/assets/images/azure-apim-scenario1.png)

This scenario fulfills all customer constraints in terms of **security** (DDOS, WAF), **performance** (Azure Front Door: TLS handshake as close as possible to users and load balancing between gateways) and **needs** (Azure APIM in front of APIs).

When using this deployment model, it is **important to**:
- [Restrict call to Azure APIM from Azure Front Door only](https://techcommunity.microsoft.com/t5/azure-paas-blog/integrate-azure-front-door-with-azure-api-management/ba-p/2654925)
- [Restrict call to Azure Functions from Azure APIM only](https://techcommunity.microsoft.com/t5/azure-architecture-blog/accept-only-traffic-from-a-front-end-service-api-management/ba-p/2811502)

A **limitation** of this external deployment model is that if consumers are on the company's network (*on-premise*), they will consume Azure APIM from the Internet, which could appear as **non-optimized**.

This is where Scenario #2 comes in.

## Scenario \#2 - Hybrid: External & Internal exposition

On this Scenario #2, I decided to deploy Azure APIM in Internal mode. It means that all Azure APIM endpoints are accessible only from within the VNet via an internal load balancer. As in the previous scenario, Azure APIM can access APIs exposed on the Internet but also exposed internally.

I called this scenario "Hybrid" because it supports two types of API consumers:
1. **Internal API Consumers**: coming from **company's network**, they will consume Azure APIM without going on to the Internet
2. **Internet API Consumers**: they will consume Azure APIM from the **Internet**

Because Azure APIM is deployed in Internal mode, I decided to use an **Azure Application Gateway to expose it on to the internet** (for the *Internal API Consumers*).

Application Gateway provides users the ability to protect the API Management service from OWASP vulnerabilities. Application gateway is a reverse proxy service which has a 7-layer load balancer and provides Web Application Firewall (WAF) as one of the services in this use case.

As Application Gateway is a **regional service**, I also decided to use **[Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)** to load balance Internet consumers between regional Application Gateways.

For Internet or Internal API Consumer, we have the following access paths:
1. Internet API Consumers:  will access to Azure APIM (Internal) through Traffic Manager => Application Gateways (that are exposed on to the Internet)
2. Internal API Consumers: will access to Azure APIM (Internal) directly or through the private path of Application Gateways

It gives the following network diagram:

  ![azure-apim-scenario2]({{ site.baseurl }}/assets/images/azure-apim-scenario2.png)

This scenario also fulfills all customer constraints in terms of **security** (DDOS on VNet & WAF Policy on Application Gateway), **performance** (Traffic Manager for giving closest Application Gateway to Internet consumers) and **needs** (Azure APIM in front of APIs)

## Scenario #1 vs Scenario #2

Here is a comparison table between Scenario #1 and Scenario #2
  ![azure-apim-scenario1]({{ site.baseurl }}/assets/images/azure-apim-scenarios-comparison.png)

In terms of functionality, the two scenarios are quite equivalent.

Scenario #2 has the advantage of allowing private API consumers not to go through the Internet. On the other hand, it requires a lot of effort in terms of deployment and the price of this solution can be excessively expensive. 

The costs of the two scenarios can be compared on the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/).

# Conclusion

There are many other scenarios and it would be impossible to illustrate them all. I preferred in this article to illustrate the two which seemed interesting to me.

A highly expected feature by Azure customers is the availability of **Azure Private Endpoint / Private Link for Azure APIM**. 

Microsoft is working on it and it promises some very interesting new scenarios even if no availability date has yet been communicated.

### References

* [HighSecurityAPIM](https://github.com/microsoft/HighSecurityAPIM)
* [Front Door APIM](https://github.com/paolosalvatori/front-door-apim)
* [How to manage, monitor and monetize API using Azure APIM](https://evan-wong.medium.com/how-to-3m-manage-monitor-and-monetize-api-using-azure-api-management-99c865e43ad8)
* [Integrating API Management with App Gateway V2](https://techcommunity.microsoft.com/t5/azure-paas-blog/integrating-api-management-with-app-gateway-v2/ba-p/1241650)
