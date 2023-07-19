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

Now, in order to connect to your on-premises network and its resources, you need to configure your ***Cloud VPN gateway***, ***on-premises VPN gateway***, and ***two VPN tunnels***.

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

![HAPNVtoPeerVPN](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0b65fd41-23a0-4ebd-9599-ff8ae106b707)



