# Cloud VPN

## What is Cloud VPN?

<p align="left">
  <img src="https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/436f3e98-87d1-4d60-aea7-94f2b72cfa39" />
</p> 

**Cloud VPN** securely connects your on-premises network to your Google Cloud VPC network through an **IPsec VPN tunnel**.


Traffic traveling between the two networks is encrypted by **one VPN gateway**, then decrypted by **the other VPN gateway**.
This protects your data as it travels over the public internet, and that’s why Cloud VPN is useful for ***low-volume data*** connections.

As a <ins>managed service</ins>, Cloud VPN provides an SLA of 99.9% service availability and supports site-to-site VPN, static and dynamic routes, and IKEv1 and IKEv2 ciphers.

Cloud VPN <ins>doesn't support</ins> use cases where client computers need to “dial in” to a VPN using client VPN software.

Also, dynamic routes are configured with **Cloud Router**.

![cloudVPN1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/bcb84884-7101-4bf2-ac8c-30e71b29b016)


Cloud VPN overview: https://cloud.google.com/vpn/docs/concepts/overview

## Classic VPN Topology

![cloudVPN2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/fdb757b3-a847-43d6-a4ae-7cf5131af2fa)

This diagram shows a Classic VPN connection between your VPC and on-premises network.

Your VPC network has subnets in **us-east1** and **us-west1**, with Google Cloud resources in each of those regions.

These resources are able to communicate using their **internal IP** addresses because routing within a network is automatically configured (<ins>assuming that firewall rules allow the communication</ins>).

In order to connect to your on-premises network and its resources, you need to configure your ***Cloud VPN gateway***, ***on-premises VPN gateway***, and ***two VPN tunnels***.

The Cloud VPN gateway is a **regional** resource that uses a **regional external IP address**.

Your on-premises VPN gateway can be a <ins>physical device</ins> in your data center or a physical or <ins>software-based</ins> VPN offering in another cloud provider's network.

This VPN gateway also has an **external IP address**.

A VPN tunnel then **connects** your VPN gateways and serves as the virtual medium through which encrypted traffic is passed.

In order to create a connection between **two** VPN gateways, you must establish **two** VPN tunnels.

Each tunnel defines the connection from the perspective of its gateway, and traffic can **only** pass when the pair of tunnels is established.

Now, one thing to remember when using Cloud VPN is that the **maximum transmission unit**, or **MTU**, for your on-premises VPN gateway <ins>cannot be greater than</ins> **1460 bytes*. This is because of the ***encryption*** and ***encapsulation of packets***.

MTU considerations: https://cloud.google.com/vpn/docs/concepts/mtu-considerations

# HA VPN

In addition to Classic VPN, Google Cloud also offers a second type of Cloud VPN gateway, HA VPN.

![HAVPN1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/d2345baf-7962-4fb3-bc4c-cd162bdcff32)

HA VPN is a **h**igh **a**vailability Cloud VPN solution that lets you securely connect your on-premises network to your Virtual Private Cloud (VPC) network through an **IPsec VPN connection** in a ***single region***.

HA VPN provides an SLA of 99.99% service availability. To guarantee a 99.99% availability SLA for HA VPN connections, you must properly configure two or four tunnels from your HA VPN gateway to your peer VPN gateway or to another HA VPN gateway.



When you create an HA VPN gateway, Google Cloud automatically chooses two external IP addresses, one for each of its fixed number of two interfaces.

Each IP address is automatically chosen from a unique address pool to support high availability.

Each of the HA VPN gateway interfaces supports **multiple tunnels**.

You can also create multiple HA VPN gateways.

When you delete the HA VPN gateway, Google Cloud <ins>releases the IP addresses for reuse</ins>.

You can configure an HA VPN gateway with only one active interface and one external IP address; however, this configuration <ins>does not</ins> provide a 99.99% service availability SLA.

VPN tunnels connected to HA VPN gateways must use dynamic **(BGP)** routing. Depending on the way that you configure route priorities for HA VPN tunnels, you can create an ***active/active*** or ***active/passive*** routing configuration.

HA VPN supports site-to-site VPN in one of the following recommended topologies or configuration scenarios: 
- An HA VPN gateway to peer VPN devices
- An HA VPN gateway to an Amazon Web Services (AWS) virtual private gateway
- Two HA VPN gateways connected to each other

## HA VPN gateway to peer VPN devices

There are three typical peer gateway configurations for HA VPN.
  - An HA VPN gateway to two separate peer VPN devices, each with its own IP address,
  - An HA VPN gateway to one peer VPN device that uses two separate IP addresses and
  - An HA VPN gateway to one peer VPN device that uses one IP address.

Let's walk through an example.

![HAPNVtoPeerVPN](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0b65fd41-23a0-4ebd-9599-ff8ae106b707)

In this topology, **one** HA VPN gateway connects to **two** peer devices. <br/>
Each peer device has **one** interface and **one** external IP address.<br/>
The HA VPN gateway uses **two** tunnels, **one** tunnel to **each** peer device.<br/>
If your peer-side gateway is ***hardware-based***, having a **second peer-side gateway** provides **redundancy** and **failover** on that side of the connection.<br/>
A second physical gateway lets you take one of the gateways offline for **software upgrades** or **other scheduled maintenance**. It also **protects** you if there is a failure in one of the devices.<br/>
In Google Cloud, the ```REDUNDANCY_TYPE``` for this configuration takes the value ```TWO_IPS_REDUNDANCY```. The example shown here provides 99.99% availability.

## HA VPN external VPN gateway to Amazon Web Services (AWS)

When configuring an HA VPN external VPN gateway to Amazon Web Services (AWS), you can use either a **transit gateway** or a **virtual private gateway**. <ins>Only</ins> the transit gateway supports **e**qual-**c**ost **m**ulti**p**ath (**ECMP**) routing.

When enabled, ECMP ***equally*** distributes traffic across active tunnels.

![HAVPNAWS](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/65d0e4fa-227c-419f-b30f-b420e8099179)

In this topology, there are **three** major gateway components to set up for this configuration.
  - An **HA VPN gateway** in Google Cloud with two interfaces,
  - **two AWS virtual private gateways**, which connect to your HA VPN gateway, and 
  - **an external VPN gateway** resource in Google Cloud that represents your AWS virtual private gateway.

This resource provides information to Google Cloud about your AWS gateway.

The <ins>supported</ins> AWS configuration uses a **total of four tunnels**.Two tunnels from one AWS virtual private gateway to one interface of the HA VPN gateway, and two tunnels from the other AWS virtual private gateway to the other interface of the HA VPN gateway. 

## Two HA VPN gateways connected to each other

You can connect two Google Cloud VPC networks together by using an HA VPN gateway in each network.

![HAVPNbetweenGCnetworks](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c83575e4-1319-43a5-864f-29509cfbfdb2)

The configuration shown provides 99.99% availability.

From the perspective of each HA VPN gateway you create **two tunnels**. You connect interface 0 on one HA VPN gateway to interface 0 on the other HA VPN, and interface 1 on one HA VPN gateway to interface 1 on the other HA VPN.

Cloud VPN topologies: https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies <br/>
Moving to HA VPN: https://cloud.google.com/network-connectivity/docs/vpn/how-to/moving-to-ha-vpn

# Cloud VPN: Static and Dynamic routes

Cloud VPN supports both static and dynamic routes.

## Dynamic routin with Cloud Router

In order to use dynamic routes, you need to configure Cloud Routers.

<p align="left">
  <img src="https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/7ddeeb48-58d2-4611-9df2-62f3e327a216" />
</p> 


Cloud Router can manage routes for a Cloud VPN tunnel using **B**order **G**ateway **P**rotocol, or BGP.  This routing method **allows** for routes to be updated and exchanged <ins>without changing</ins> the tunnel configuration.

![CloudRouter](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/398051e2-4e62-4f44-ae98-0a317a1756b0)


This diagram shows two different regional subnets in a VPC network, namely Test and Prod. The on-premises network has ***29*** subnets, and the two networks are connected through **Cloud VPN tunnels**.

### How would you add a new “Staging” subnet in the Google Cloud network and a new on-premises 10.0.30.0/24 subnet to handle growing traffic in your data center?

To automatically propagate network configuration changes, the VPN tunnel uses Cloud Router to establish a BGP session between the VPC and the on-premises VPN gateway, which must support BGP. The new subnets are then seamlessly advertised between networks. This means that **instances in the new subnets can start sending and receiving traffic immediately**.

To set up **BGP**, an additional IP address has to be assigned to each end of the VPN tunnel.These two IP addresses must be link-local IP addresses, belonging to the IP address range **169.254.0.0/16**. These addresses <ins>are not</ins> part of IP address space of either network and are used exclusively for establishing a BGP session.

