---
layout: post
title:  "Azure Private Endpoints implementation at scale // DNS deep-dive"
author: davidsantiago
categories: [ azure, network ]
image: assets/images/azure-pe-dns-deepdive-5.png
---

> This post was originally published on [Michelin BlogIT](https://blogit.michelin.io/azure-private-endpoints-implementation-at-scale-dns-deep-dive/).

The intent of this article is to explain how we deployed widely Azure Private Link on PaaS Services and **how we tackled the DNS part** of this technology, both for **on-premise and Azure machines/workstations**.

## Azure Private Link

[Azure Private Link](https://docs.microsoft.com/en-US/azure/private-link/private-link-overview) is a technology designed to provide private connectivity to selected PaaS services, customer-owned and partner offered services.

Private Link allows you to access services over a private endpoint deployed into your virtual network. All traffic privately goes through the Microsoft global backbone, eliminating any exposure to the public internet and increasing security.

![private-link]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-1.png)

There are two key components of _Azure Private Link_:
* **Private Endpoint** – a network interface connected to your virtual network, assigned with a private IP address. It is used to connect privately and securely to a service powered by Azure Private Link or a Private Link Service that you or a partner might own.
* **Private Link Service** – your own service, powered by Azure Private Link that runs behind an Azure Standard Load Balancer, enabled for Private Link access. This service can be privately connected with and consumed using Private Endpoints deployed in the consumer’s virtual network.

**The main challenge** when we use Azure Private Endpoints on a PaaS service is the **DNS part**. We will explain below how we tackle this challenge both for on-premise and Azure infrastructure with a Storage Account as an example but first, we need to deep dive on how DNS works in a traditional hub and spoke vDC.

## Azure DNS operation in a traditional hub and spoke topology

When we built our first virtual Datacenter in Azure in 2015, we followed Microsoft recommendations regarding the hub and spoke topology implementation:
* **hub VNet** acts as a central point of connectivity (ExpressRoute, VPN, ...) to many spoke virtual networks, but also hosts infrastructure management services (Domain Controller, Patching Services, Security Scans services, Proxies, ...).
* **spokes VNets** host usual workload (VMs...). Workloads are isolated between spokes and/or subnets.
* the **DNS Servers configuration** on all virtual network is IP addresses of the **two Domain Controllers** hosted in hub VNet.
* DNS servers on Domain Controllers are configured as following:
  * **Conditional forwarders** to on-premise DNS appliances. It is used for **private DNS resolution**.
  * **Traditional/Conventional forwarders** to DNS & HTTP Proxies solution hosted in hub VNet. It is used for **public DNS resolution**. The DNS & HTTP Proxies (non-authoritative) forwards DNS requests to a public DNS services that Michelin Group trusts.

**Example of resolution path**:

* Virtual Machine A resolves _www.microsoft.com_: `VM A <=> DC1 or DC2 <=> DNS Proxies <=> Public DNS solution trusted by Michelin Group`
* VM A resolves _somethingprivate.michelin.com_: `VM A <=> DC1 or DC2 <=> Private DNS solution (appliances) on-premise`

This model, widely deployed on the market, is ultimately quite simple.
_DNS Proxies_ are not mandatory, we could forward public request directly from Domain Controllers. We decided to use a DNS Proxy solution to get security features (in addition to traceability).

## vDC network architecture with Private Endpoint

At Michelin, we decided to have **dedicated VNets to host Private Endpoints NICs** (_Network Interface Cards_). These dedicated PaaS VNets are peered to hub and peered to _eligible_ spokes:

![vdc-network-arch]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-2.png)

* Peering between `PaaS VNet` and `hub VNet` ensure that the **Private Endpoint** is **accessible from on-premise** (this part will be described later)
* Peering between `PaaS VNet` and `spoke2 Vnet` ensure that **VMs** in `spoke2 VNet` **will be able to access to the Private Endpoint**

Taking the above diagram, VM B can access the storage account using the private IP `10.7.0.4`. In fact, it cannot work because the certificate issued by Microsoft is given for `*.blob.core.windows.net`:

![certificate]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-3.png)

It will result in an insecure connection error. At the end, we need to consume the Storage Account by using the public FQDN.

> If the private IP address 10.7.0.4 was put in certificate's Alternate Name field (as we can see sometimes on other mechanisms), it could works but hopefully, this is not the method pushed by Microsoft: client must come using the storage account FQDN (Full Qualified Domain Name). ↩

Accessing https://storageaccountblog.blob.core.windows.net/containerExample from VM B will maybe work ... or not! 😅

Storage Account DNS resolution from VM B gives the following result:

```bash
$ nslookup  storageaccountblog.blob.core.windows.net
Server:		DC1
Address:	10.x.x.x#53

Non-authoritative answer:
storageaccountblog.blob.core.windows.net	canonical name = storageaccountblog.privatelink.blob.core.windows.net.
storageaccountblog.privatelink.blob.core.windows.net	canonical name = blob.ams23prdstr02a.store.core.windows.net.
Name:	blob.ams23prdstr02a.store.core.windows.net
Address: 52.239.213.100
```

Even after having deployed Private Endpoint, the DNS result from Storage Account FQDN is a Public IP. If VM B has access to the Internet and the Storage Account Firewall allows public access, it will work but this is not what we want here. **We want VM B to access the Storage Account using the Private Endpoint**.

Do get this behavior, two actions are required:
* Leverage Azure Private DNS zone
* Configure DNS Proxy to use the Azure magic IP address to perform DNS resolution on Azure Private DNS zone

## Azure Private DNS Zone & Azure Magic IP !

> [Azure Private DNS](https://docs.microsoft.com/en-US/azure/dns/private-dns-overview) provides a reliable and secure DNS service to manage and resolve domain names in a virtual network. The Private DNS Zone must be linked to the virtual network so that VMs in the Virtual Network can resolve all DNS records published in the private zone.

The idea here is to deploy an Azure Private DNS Zone which manages the `privatelink.blob.core.windows.net` _suffix_ zone, linked to the hub virtual Network.

When we create a Private Endpoint, the NIC will be under `PaaS VNet/pe-storage0001` VNet/subnet and we need to **enable the Private DNS integration on the Private DNS zone** linked to the `hub` VNet:

![private-dns-zone]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-4.png)

The last thing to do is to configure public DNS server used by the "DNS Proxies" infrastructure. As the "DNS servers" configured on the VNets are not the "Azure-provided" but the IPs of the two Domain Controllers and as our Domain Controllers forward requests for public resolution the "DNS Proxies" infrastructure, this is where we need to configure a **Magic IP**!

**The Magic IP DNS Proxies must use** to perform DNS resolution is **the IP** `168.63.129.16`. This public IP address is used by Microsoft Azure in all regions and all national clouds.

`168.63.129.16` is in charge of doing DNS resolution according to the private DNS Zone linked to the VNet the request was submitted from, but also does [many other things](https://docs.microsoft.com/en-US/azure/virtual-network/what-is-ip-address-168-63-129-16).

When DNS Private Zone is linked to hub VNet and `168.63.129.16` configured in DNS Proxy infrastructure, the `nslookup` request from VM B is much better:
```bash
Server:		DC1
Address:	10.x.x.x#53

Non-authoritative answer:
storageaccountblog.blob.core.windows.net	canonical name = storageaccountblog.privatelink.blob.core.windows.net.
storageaccountblog.privatelink.blob.core.windows.net	canonical name = blob.ams23prdstr02a.store.core.windows.net.
Name:	blob.ams23prdstr02a.store.core.windows.net
Address: 10.7.0.4
```

Here is a summary of the **whole DNS resolution path**:

![dns-resolution-path]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-5.png)

* `VM B` request `storageaccountblog.blob.core.windows.net` to `DC 1 or DC2`
* `DC1 or DC2` to `DNS Proxies`
* `DNS Proxies` to `168.63.129.16` that answer `10.7.0.4`
* `VM B` connects to the Storage Account through `storageaccountblog-pe`

DNS Proxies can be a [nginx-based solution as proposed in the Microsoft documentation](https://github.com/microsoft/PL-DNS-Proxy), or other solutions available on the market.

To use Private Endpoints with other PaaS Services (Azure Database for PostgreSQL, Azure EventGrid ...), it is required to [create an Azure Private DNS Zone for each PaaS Services corresponding to DNS zone used by Azure](https://docs.microsoft.com/en-US/azure/private-link/private-endpoint-dns#azure-services-dns-zone-configuration).

## Security

When public access is not required, it is a good practice to **disable the public path of a PaaS service**, especially since, nowadays, all Azure PaaS services have "Networking" tab to configure network access filtering.

Regarding the filtering on `PaaS VNet`, NSG (applied to subnets) are currently ignored by Private Endpoints. The filtering must be done on the source-side.

Even if NSG are ignored, it can be a good practice to have one applied to `PaaS Vnet` subnets to get insights on network flows, with NSG Flow Logs feature enabled.

## On-premise integration of Private Endpoint

Private Endpoints must be also accessible from on-premise for several use cases like:
* Manage Azure Database for PostgreSQL servers from DevOps workstation
* Accessing Azure Event Hub from on-premise servers

As for cloud machines, the complexity of this access is highly related to DNS resolution.

## Context

Our on-premise DNS architecture is based on a different design depending on the client type (servers or workstations).
* For security reasons, our on premise servers did not have any DNS public resolution.
* For workstations, the situation was a little bit easier because of a native DNS public resolution.

The diagram below explains how it works in detail:

![dns-resolution-path-onprem]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-6.png)

## Evolution 

To resolve DNS requests of Private Endpoint's entries for everyone including servers, we implemented new configurations in two steps :
* conditional forwarding of Private Link DNS zones from local recursive server to top level recursive server
* conditional forwarding of Private Link DNS zones from top level recursive server to newly created DNS proxy in Azure

> What is always important to consider in a recursive mode is that the original subdomain like `blob.core.windows.net` must be forwarded and not only the sub-subdomain `privatelink.blob.core.windows.net`.

Here is how clients can reach Private Endpoints from internal network as well as VPN:

![dns-resolution-path-all]({{ site.baseurl }}/assets/images/azure-pe-dns-deepdive-7.png)

This implementation requires to update forwarding rules each time a new Private Link zones comes, but helps controlling their integration.

## Conclusion

Private Endpoints sound magic and we have been waiting for this feature from Microsoft Azure for years to stop Internet exposition of our PaaS Services.

DNS was the complicated part on the deployment, especially when you need to explain to security teams that we have public FQDN that will answer private IP address.

At the end, we found several solutions and deployed the one explained here. We have now many challenges regarding vDC evolution, specially on DNS tomorrow with Azure Firewall and Azure AD Domain Services.

Thank you to Thierry Verger & Xavier Barros.

### References

* [What is azure private link](https://www.techkb.onl/what-is-azure-private-link/)
* [Azure Private Link DNS MicroHack](https://github.com/adstuart/azure-privatelink-dns-microhack)
* [Private Endpoint DNS Integration Scenarios](https://github.com/dmauser/PrivateLink/tree/master/DNS-Integration-Scenarios)



