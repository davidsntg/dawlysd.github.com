---
layout: post
title:  "pfSense & Azure - Set up S2S VPN with static or dynamic routing"
author: davidsantiago
categories: [ azure, network ]
image: assets/images/pfsense-azure-s2s-00.png
---

In this article, I will describe how to configure pfSense to create a S2S VPN to Azure with static or dynamic routing.

# pfSense
[pfSense](https://www.pfsense.org/) is a free firewall/router computer software distribution based on FreeBSD. The open source pfSense Community Edition (CE) and pfSense Plus is installed on a physical computer or a virtual machine to make a dedicated firewall/router for a network. It can be configured and upgraded through a web-based interface, and requires no knowledge of the underlying FreeBSD system to manage.

# Azure & pfSense - S2S VPN static souting

The pfSense configuration described in this article is based on the following architecture:

![BigPicture]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-00.png)

If you have not installed pfSense already, follow [this tutorial](https://docs.ovh.com/au/en/dedicated/pfSense-bridging/) to do it.

Before configuring pfSense, let's deploy basic Azure infrastructure.

## Azure infrastructure

In a resource group, create :
* A virtual Network (VNet)
  * Name: vnet
  * Address Space: 10.1.0.0/16
  * Subnets
    * default: 10.1.0.0/24
    * GatewaySubnet: 10.1.1.0/25
* A virtual network gateway:
  * Name: vnetgw01
  * Region: same VNet location 
  * Gateway type: VPN
  * VPN type: Route-based
  * SKU: Basic
  * Generation: Generation2
  * Virtual network: VNet
  * Public IP address name: vnetgwpip01
  * Enable active-active mode: Disabled
  * Configure BGP: disabled
* A local network gateway:
  * Name: vnetgwlng01
  * Region: same VNet region
  * IP address: pfSense machine public IP (WAN interface in the schema)
  * Address space: 192.168.1.0/24 (or all address space you want to route through the tunnel) 
  * Configure BGP settings: No
* A VNet gateway connection:
  * Name: azure-to-onprem
  * Connection type: Site-to-site (IPsec)
  * First virtual network gateway: vnetgw01
  * Local network gateway: vnetgwlng01
  * Shared key (PSK): david123
  * IKE protocol: IKEv2

Let's now configure pfSense.

## pfSense configuration

### Create a new IPsec VPN

Go to `VPN -> IPsec` and create a new IPsec VPN:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-01.png)

Save and apply changes.

Configure now Phase 2:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-02.png)

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-03.png)

Update Phase 2 SA/Key Exchange parameters:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-04.png)

Do not change other parameters:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-05.png)

### Configure IPsec Firewall rules

Go to `Firewall -> Rules -> IPsec` and create a new rule that will allow everything:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-06.png)

Save the rule and apply changes:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-07.png)

Go to `Status -> IPsec`:

![AzureAPIManagementServicesOverview]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-08.png)

Check the VPN is connected on Azure side also:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-09.png)

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-10.png)

### Run final connectivity checks

Provision a VM on default subnet.

Check effective routes on the network interface card:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-11.png)

Go to `Diagnostics -> Ping` and ping this VM (*10.1.0.4*) from pfSense using *LAN* source address:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-12.png)

The S2S VPN between Azure & pfSense is up using static routing.


# Azure & pfSense - S2S VPN dynamic routing

## Azure infrastructure

I will keep the same infrastructure used previously.
It is just required to update the virtual network gateway, the local network gateway and the connection.

Update virtual network gateway, enable BGP and provide Azure BGP ASN:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-13.png)

Update local network gateway, enable BGP and provide custom BGP ASN and BGP peer IP address:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-14.png)

Finally, update virtual network gateway and enable BGP:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-15.png)

## pfSense configuration

Let's update IPsec Phase 2 configuration, install and configure FRR.

### Update IPsec Phase 2 configuration

Go to `VPN -> IPsec` and edit IPsec Phase 2 configuration:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-16.png)

### Install FRR package

Go to `System -> Package Manager` and install [FRR package](https://docs.netgate.com/pfsense/en/latest/packages/frr/index.html):

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-17.png)

FRR package will bring BGP feature to pfSense.

### Update FRR configuration

Go to `Services -> FRR Global/Zebra -> Raw Config` :

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-18.png)

Deploy this configuration:
```bash
##################### DO NOT EDIT THIS FILE! ######################
###################################################################
# This file was created by an automatic configuration generator.  #
# The contents of this file will be overwritten without warning!  #
###################################################################
!
frr defaults traditional
hostname pfSense.home.arpa
password david123
service integrated-vtysh-config
!
ip router-id 192.168.1.7
!
router bgp 65505
  bgp router-id 192.168.1.7
 neighbor 10.1.1.126 remote-as 65515
 neighbor 10.1.1.126 description Azure Bgp Private Ip
 !
 address-family ipv4 unicast
  redistribute connect 
  redistribute kernel
  neighbor 10.1.1.126 activate
  no neighbor 10.1.1.126 send-community
  neighbor 10.1.1.126 prefix-list bc-any in
  neighbor 10.1.1.126 prefix-list bc-any out
 exit-address-family
 !
!
ip prefix-list bc-any seq 9 deny 0.0.0.0/0
ip prefix-list bc-any seq 10 permit any 
!
line vty
!
```
*Note: pfSense won't announce 0.0.0.0/0 route with this configuration.*

Save, go to `Services -> FRR Gobal/Zebra -> Global Settings`  and Force FRR Service Restart.

### Run checks

From Azure Portal, display VmTest NIC effective routes:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-19.png)

From Azure Portal, display virtual network gateway BGP peers :

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-20.png)

From pfSense, go to `Diagnostics -> Routes`: 

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-21.png)

From pfSense, go to `Status -> FRR -> BGP`:

![pfSenseConfig]({{ site.baseurl }}/assets/images/pfsense-azure-s2s-22.png)

The S2S VPN between Azure & pfSense is up using dynamic routing.

# References

Below articles were used to write this article (thanks authors!):
* [Configuring pfSense network bridge](https://docs.ovh.com/au/en/dedicated/pfSense-bridging/)
* [Route-Based VPN vs Policy-Based VPN (aka dynamic vs static): a more complete picture](https://www.linkedin.com/pulse/route-based-vpn-vs-policy-based-aka-dynamic-static-more-shawn-travers)
* [Set up Dynamic Routing with FRR (Free Range Routing) in pfSense – OpenBGPD now Depricated](https://blog.matrixpost.net/set-up-dynamic-routing-with-frr-free-range-routing-in-pfsense-openbgpd-now-depricated/)
