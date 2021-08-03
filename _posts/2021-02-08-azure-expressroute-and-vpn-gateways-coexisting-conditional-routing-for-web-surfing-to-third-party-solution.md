---
layout: post
title:  "Azure ExpressRoute and VPN Gateways coexisting - Conditional routing for web-surfing to third party solution"
author: davidsantiago
categories: [ azure, network ]
image: assets/images/azure-expressroute-vpn-coexisting-9.png
---

> This post was originally published on [Michelin BlogIT](https://blogit.michelin.io/azure-expressroute-and-vpn-gateways-coexisting-conditional-routing-for-web-surfing-to-third-party-solution/).

## Context
Azure implementation at Michelin follows Microsoft recommendations: we have several **v**irtual **D**ata **C**enters (vDC) in different regions connected to our "on-premises" network through **Express Route**.

In terms of network topology, we leverage the traditional hub and spoke model:

![hub-and-spoke]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-1.png)

The vast majority of the workload are Virtual Machines (VMs) with Network Interface Cards (NICs) in Virtual Networks (VNets) of the spokes.

Due to internal security policy, VMs are not allowed to go direct to the Internet. The access is currently denied using Network Security Groups (NSGs) and tomorrow using Azure Firewall.

If a VM needs to go on to the Internet, it must use a web proxy solution that is available in the hub VNet and carry a global allow-list.

## Special Case: Virtual Desktop Infrastructure (VDI) and web-surfing

We have an application (VDI) which has a dedicated VNet peered to the hub network:

![vdi]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-2.png)

_tenant_ subnet hosts several Windows 10 VMs able to go to the Internet. To do this, VMs are configured with a proxy.pac which provides the right proxy configuration depending on the ressource to access.

Everything was working well until ... our Network service line launched a project internally to:
* Switch to another proxy supplier which will be more appropriated to cover all our scope.
* **Remove the proxy.pac from machines: everything must be done at route-level.**

We needed to set up an elegant network architecture that only impacts VNets/subnets with machines that need web-surfing. And for this, we explore different solutions in a try / do / learn approach to select the best one.

## Solution #1 - [Fail] Dedicated VNET to carry IKEv2 IPSec VPN to web-proxy supplier

Our first approach was to build a dedicated VNet that will carry an IKEv2 IPSec Site to Site VPN to the proxy supplier and the peer VNets that are doing web surfing to this _proxy-vnet_:

![solution-1]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-3.png)

IMHO, that was an elegant approach but in practice, **it is not possible to implement this solution**.

There is a limitation in Azure, and it is not possible to have a VNET peered to multiple VNETs with "Use remote Gateway" flag checked: we can check this flag only for one peering.

The error we get is the following:
> Failed to add virtual network peering 'vdi-vnet-to-proxy-vnet'. Error: Peering vdi-vnet/vdi-vnet-to-proxy-vnet **cannot have UseRemoteGateways flag set to true, because another peering** /subscription/.../resourceGroups/.../.../virtualNetworkPeerings/vdi-vnet-to-hub-vnet **already has UseRemoteGateways flag set to true**.

## Solution #2 - [Fail] Build the VPN Tunnel in dedicated application VNets

Step backwards: let us try to build the VPN in the VDI application VNet directly:

![solution-2]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-4.png)

The VPN was configured, and I was able to connect from tenant subnet to go on to the Internet through the tunnel during web-surfing but we had another issue.

It is not possible to re-create the peering from _vdi-vnet_ to _hub-vnet001_ with the option "Use the remote virtual network's gateway" because this option is disabled:

![solution-2-button-disabled]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-5.png)

As sometimes the Azure web portal does not let us do things, but it works using Azure CLI; I tried but it failed with a very explicit error message: `ErrorResponse`...

It looks like Azure stuck the "Use the remote virtual network's gateway" during peering between two VNets when the two VNets have a GatewaySubnet.

It means that we can enable the peering without the option, but we lost the Express Route connectivity, and this is not what we want.

## Solution #3 - [Success but dirty] Express Route VNet Gateway + VPN VNet Gateway in dedicated VNET

After having suffered two failures, I decided to mount an express route VNET gateway in **vdi-vnet/GatewaySubnet** to ensure connectivity with the "rest of the world" which gives the following network architecture:

![solution-3]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-6.png)

Guess what? It works and it is pretty logical but I identified many issues :
* Cost: 2 gateways (1 Express Route & 1 VPN) per VNets for workloads that need to do web-surfing will end up being very expensive
* Operations: multiple VPN VNet gateway implies multiple public IPs on our side and many configuration efforts on proxy supplier side. This is not scalable.
* Filtering: next year we will deploy Azure Firewall in _hub-vnet_. What about the filtering for _vdi-vnet_? Do we need to consider this VNet as a "new hub" plugged to express route circuit ?
* Latency: if VDI machines need to access applications hosted in _hub-vnet_ spokes, that means going back through the express route route each time.

After a call with Alexandre Weiss, network specialist from Microsoft, he confirmed what I wanted to avoid from the beginning: the VPN Gateway must be deployed in the hub network.

## Solution #4 - [Success] VPN VNet Gateway in hub-vnet

By deploying the VPN VNet Gateway in the _hub-vnet_, we get the following network architecture:

![solution-4]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-7.png)

In terms of configuration, we need to configure the Local Network Gateway associated to the VPN VNet Gateway and the _hub-vnet_ VNet as the following:

![solution-4-configuration]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-8.png)

Announcing `0.0.0.0/1` and `128.0.0.0/1` is a trick because we cannot announce a default route `0.0.0.0/0` (neither in the portal nor Azure CLI). Ideally, all possible IPs should be announced minus the company's RFC1918 (which sometimes can be exotic) but it gives a long list of range.

When we are announcing that, it will propagate `0.0.0.0/1` and `128.0.0.0/1` to _hub-vnet001_ and all it is peered network to use _VPNgw001_ as Next Hop. The BGP routes announced in the Express Route circuit being more precise and will be preferred compare to larger route (`0.0.0.0/1` & `127.0.0.0/1`).

By doing that, _vdi-vnet/tenant_ subnet machines are able to do web-surfing using _VPNgw001_ located in _hub-vnet001_ but we still have an issue: the DefaulRoute in Azure ( `0.0.0.0/0`  => `Internet`) become Invalid, replaced by what we announced (the tunnel).

That becomes a problem because:
* We want to send in the tunnel **only subnets with machines doing web-surfing**
* We **do not want to modify the default route of other VNets** because even if at the beginning I said that the machines must use a proxy to go to the Internet, we host applications that must go directly to the Internet (in a derogation way, because the application does not support network proxification...)

The solution is therefore to override the route tables of subnets that do not want to send web traffic in the tunnel:
![solution-4-final]({{ site.baseurl }}/assets/images/azure-expressroute-vpn-coexisting-9.png)

This is the final solution we have chosen except that we built a second hub that hosts both the express route gateway and the VPN gateway: the idea was not to impact all the vnets/subnets that did not need this flow proxy solution. Vnets that need this proxy solution are peered to the second hub only.

## Conclusion

If sometimes, network in Cloud ecosystem looks magic, it is far from that and this is not just pipes!

This design will have to be reviewed soon when we will add the NVA brick (Azure Firewall) to the network design of our vDCs.

I hope this article can help people who will have the same needs as us. Do not hesitate to comment on the article if you have any questions or comments.

## Acknowledgment

A huge thank you to [Vincent Misson](https://twitter.com/vincentmisson) with whom I worked on the subject, we learned a lot about the network in Azure through these different stages.

Additionally, thank you to Alexandre Weiss for the network expertise you provided to us.


