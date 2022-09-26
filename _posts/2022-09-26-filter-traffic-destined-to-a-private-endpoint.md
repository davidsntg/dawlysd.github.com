---
layout: post
title:  "Filter traffic to Private Endpoints with Azure Firewall"
author: davidsantiago
categories: [ network, azurefirewall ]
image: assets/images/inspect-pe-with-azfw-3.png
featured: true

---

In this article, I present how to filter traffic destined to a Private Endpoint with Azure Firewall.
Long story short: **SNAT is almost mandatory** 😀, as explained in the [documentation](https://learn.microsoft.com/en-us/azure/private-link/inspect-traffic-with-azure-firewall).

# Introduction

## Reminders

* Private Endpoint **only support TCP (not ICMP/UDP)**. [Doc](https://learn.microsoft.com/en-us/answers/questions/379875/private-endpoint-has-listening-restrictions.html).
* Private Endpoint **do not support NSG flow logs**. It means that Traffic Analysis do not work. [Doc](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-nsg-flow-logging-overview).
* Private endpoint **automatically creates a /32 route** propagated to the VNet it resides in and other peered VNets. [Details](https://msandbu.org/private-endpoints-snat-udr-and-azure-firewall/).

## [User-defined routes support for PEs (GA)](https://azure.microsoft.com/en-us/updates/general-availability-of-user-defined-routes-support-for-private-endpoints/)

When [Private Endpoint Network Policies](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy?tabs=network-policy-portal) is enabled on a subnet containing PEs, it 
* **invalid /32 route propagated into peered VNets** only *(highlighted in yellow)*
* **provides the ability to override private endpoints with a larger prefix** *(highlighted in green)*
  
  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-1.png)

Even with Private Endpoint Network Policies enabled, **PEs do not honor UDR attached to their subnet**:
* PEs **always tries to route the packet back directly to the source IP** (VM, Client...)
* UDR are bypassed by traffic coming from PEs. UDR can be used to override traffic destined for the PE only.

The impact (*described later in the article*) is that it is not possible to configure symetric routing in both sides.

# (Best Approach) UDR & Application Rules (SNAT)

As documented, **SNAT** is [recommended](https://learn.microsoft.com/en-us/azure/private-link/inspect-traffic-with-azure-firewall):
* The use of **application rules over network rules is recommended when inspecting traffic destined to private endpoints in order to maintain flow symmetry**.
* If network rules are used, or an NVA is used instead of Azure Firewall, **SNAT must be configured for traffic destined to private endpoints**.

Azure Firewall behavior:
* Azure Firewall doesn't SNAT with Network rules when the destination IP address is in a private IP address range per IANA RFC 1918 or shared address space per IANA RFC 6598.
* **Application rules always SNAT**.

Let's study several scenarios based on the following architecture:

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-2.png)

**Scenario \#1**: `spoke01-vm` to `https://savmfy26.blob.core.windows.net/hello/Hello.txt` using Application Rule (*SNAT*)

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-3.png)

**Result**: ✅

**Explanation**: Works as expected

**Scenario \#2**: `hub-vm` to `https://savmfy26.blob.core.windows.net/hello/Hello.txt` using Application Rule (*SNAT*)

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-4.png)

**Result**: ✅

**Explanation**: Works as expected

## Limitations

There are some limitations in using application rules with Azure Firewall:
* Application rules supports only HTTp, HTTPS and MSSQL protocols
* If other protocols are needed, it is required to
  * Leverage Network Rule to perform filtering
  * Configure Private IP ranges (SNAT) in Azure Firewall Policy to include Private Endpoints vnet in addresses that require SNAT.

# (Supported Approach) UDR & Network Rules with SNAT

Forcing SNAT for Private Endpoints vnet is done using on Private IP ranges (SNAT) in Azure Firewall Policy. SNAT must be configured for all adresses except for RFC1918 minus [pe-vnet-address-space] (*In other words, it forces SNAT for \[pe-vnet-address-space\]*). Using [this tool](https://bluekeyes.github.io/cidrtool/), the operation is quite simple:

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-5.png)

**Scenario \#3**: `spoke01-vm` to `https://savmfy26.blob.core.windows.net/hello/Hello.txt` using Network Rule and SNAT for all IP addresses except (RFC1918 minus *10.221.9.0/24*)

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-6.png)

**Result**: ✅

**Explanation**: Works as expected

**Scenario \#4**: `hub-vm` to `https://savmfy26.blob.core.windows.net/hello/Hello.txt` using Network Rule and SNAT for all IP addresses except (RFC1918 minus *10.221.9.0/24*)

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-7.png)

**Result**: ✅

**Explanation**: Works as expected

**Scenario \#5**: `spoke01-vm` to `pgsql-vmfy26.postgres.database.azure.com:5432` using Network Rule and SNAT for all IP addresses except (RFC1918 minus *10.221.9.0/24*)

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-8.png)

**Result**: ✅

**Explanation**: Works as expected


**Scenario \#6**: `hub-vm` to `pgsql-vmfy26.postgres.database.azure.com:5432` using Network Rule and SNAT for all IP addresses except (RFC1918 minus *10.221.9.0/24*)

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-9.png)

**Result**: ✅

**Explanation**: Works as expected

# (Wrong Approach) UDR & Network Rules without SNAT

Long story short: we may have random behavior!

**Scenario \#7**: `spoke01-vm` to `https://savmfy26.blob.core.windows.net/hello/Hello.txt` using Network Rule without SNAT

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-10.png)

**Result**: ✅

**Explanations**: 
* [SDN layer for storage behavior](https://github.com/MicrosoftDocs/azure-docs/issues/69403)
* [Traffic is symmetric for storage](https://blog.cloudtrooper.net/2020/05/23/filtering-traffic-to-private-endpoints-with-azure-firewall/)

**Note**: Even if it works, I do not recommend going into production with this configuration.

**Scenario \#8**: `hub-vm` to `https://savmfy26.blob.core.windows.net/hello/Hello.txt` using Network Rule without SNAT

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-11.png)

**Result**: ✅

**Explanations**: 
* [SDN layer for storage behavior](https://github.com/MicrosoftDocs/azure-docs/issues/69403)
* [Traffic is symmetric for storage](https://blog.cloudtrooper.net/2020/05/23/filtering-traffic-to-private-endpoints-with-azure-firewall/)

**Note**: Even if it works, I do not recommend going into production with this configuration.

**Scenario \#9**: `spoke01-vm` to `pgsql-vmfy26.postgres.database.azure.com:5432` using Network Rule without SNAT

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-12.png)

**Result**: ✅

**Explanation**: I don't know why it is working 😅

**Scenario \#10**: `hub-vm` to `pgsql-vmfy26.postgres.database.azure.com:5432` using Network Rule without SNAT

  ![filter-pe-awfw]({{ site.baseurl }}/assets/images/inspect-pe-with-azfw-13.png)

**Result**: ❌

**Explanation**: Normal because asymmetric routing.

# Conclusion

* SNAT is mandatory right now until PEs honor UDRs attached to their subnets
* Leverage UDRs with either **Application Rules** or **Network Rules with SNAT configured**
* Calculate RFC1918 minus \[pe-vnet-address-space\] is [easy](https://bluekeyes.github.io/cidrtool) to configure Azure Firewall

### References

* [Filtering traffic to Private Endpoints with Azure Firewall](https://blog.cloudtrooper.net/2020/05/23/filtering-traffic-to-private-endpoints-with-azure-firewall/)
* [Private Endpoints – SNAT – UDR and Azure Firewall](https://msandbu.org/private-endpoints-snat-udr-and-azure-firewall/)
* [Interesting behaviors with Private Endpoints](https://journeyofthegeek.com/2020/09/10/interesting-behaviors-with-private-endpoints-new/)
  
