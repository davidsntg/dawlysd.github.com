---
layout: post
title:  "Expose on-premise applications with Application Gateway, and share those applications privately to Azure partners"
author: davidsantiago
categories: [ network, applicationgateway ]
image: assets/images/ag-lb-onprem-4.png
featured: true

---

Load balancing on-premise applications from Azure with Application Gateway is possible.
The intent of this article is to explain how to achieve that, but also detail how to **share these on-premise applications to a partner using Azure Application Gateway + Private Link (*preview*), all privately**.

# Load Balance on-premise applications from Azure

**Architecture**:

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-1.png)

The terraform code to provision above infrasture is available on my [GitHub](https://github.com/dawlysd/lab-ag-onpremiseapplications).

**Description**:
* Two on-premises sites (emulated) are establishing S2S VPN to Azure `hub-vnet`. 
* `site1-vm` and `site2-vm` host an apache2 web server
* `ApplicationGateway` in `hub-vnet` has a Private Listener on port `80`, Load Balancing HTTP traffic to `site1-vm:80` and `site2-vm:80` machines

This architecture is interesting when there is no on-premise load balacing solution available or if it is not possible to rely on it/them.

**Load balancing illustration**:
  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-2.gif)

**Result of** `$ curl http://10.221.1.134` **from** `hub-vm`:

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-3.png)

**Result**: Azure VMs are able to reach Azure Application Gateway which Load Balance traffic to on-premise applications.

**Note**: if backend machines expose websites through HTTPS using a custom (or corporate) Certificate Authority, it must be trusted by Application Gateway. It is required to [upload the root certificate to Application Gateway's HTTP Settings](https://learn.microsoft.com/en-us/azure/application-gateway/self-signed-certificates#upload-the-root-certificate-to-application-gateways-http-settings).

Now that we have managed to load balance traffic to on-premise applications, let's try to share those applications with partners which use Azure.

# Share those on-premise applications to Azure partners, privately!

[Private link support in Azure Application Gateway is in preview since June 2022](https://azure.microsoft.com/en-us/updates/public-preview-private-link-support-for-application-gateway/).

Let's use this preview feature to create a Private Endpoint in Fabrikam tenant that will point to Contoso's Application Gateway 🔥🔥.

**Architecture**:
  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-4.gif)

**Step-by-step configuration**:
* **Contoso tenant**: **[Configure Private Link configuration](https://learn.microsoft.com/en-us/azure/application-gateway/private-link-configure?tabs=portal)** - it defines the infrastructure used by Application Gateway to enable connections from Private Endpoints.

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-5.png)

* **Fabrikam tenant**: **Create the Private Endpoint**

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-6.png)

  * **Indicate the Resource ID of Application Gateway**. Target sub-resource value must be the Private Frontend IP configuration name.

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-7.png)

  * **Provide the vnet/subnet where the Private Endpoint will be created**.

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-8.png)

  * After creation, the Private Endpoint connection status indicates Pending, it means that the connection must be approved on Contoso tenant.
  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-9.png)

* **Contoso tenant**: **Approve Private Endpoint connection request from Fabrikam tenant**

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-10.png)

**Result of** `$ curl http://10.3.0.5` **from** `fabrikam-vm`:

  ![ag-lb-onprem]({{ site.baseurl }}/assets/images/ag-lb-onprem-11.png)

It works 🔥🔥


### References

* [YouTube - Adam Stuart - Azure Private Link support for Azure Application Gateway](https://www.youtube.com/watch?v=jBZSHQRBON4)