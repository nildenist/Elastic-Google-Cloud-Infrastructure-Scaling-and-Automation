# Configuring Google Cloud HA VPN

## Overview

HA VPN is a high-availability (HA) Cloud VPN solution that lets you securely connect your **on-premises** network to your VPC network through an IPsec VPN connection in a single region. HA VPN provides an SLA of 99.99% service availability.

HA VPN is a **regional** per VPC, VPN solution. HA VPN gateways have two interfaces, each with **its own public IP address**. When you create an HA VPN gateway, two public IP addresses are ***automatically*** chosen from different address pools. When HA VPN is configured with two tunnels, Cloud VPN offers a 99.99% service availability uptime.

In this workshop you create a global VPC called vpc-demo, with two custom subnets in **us-east1** and **us-central1**. In this VPC, you add a Compute Engine instance in each region. You then create a second VPC called on-prem to simulate a customer's on-premises data center. In this second VPC, you add a subnet in region us-central1 and a Compute Engine instance running in this region. Finally, you add an HA VPN and a cloud router in each VPC and run two tunnels from each HA VPN gateway before testing the configuration to verify the 99.99% SLA.


## Objectives

In this workshop, you learn how to perform the following tasks:

   - Create two VPC networks and instances.

   - Configure HA VPN gateways.

  - Configure dynamic routing with VPN tunnels.

  - Configure global dynamic routing mode.

  - Verify and test HA VPN gateway configuration.

## Task 1. Set up a Global VPC environment
In this task you set up a Global VPC with **two** custom subnets and **two** VM instances running in each zone.

1. In Cloud Shell, create a VPC network called vpc-demo:
![vpc_demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f06c371e-0ca2-4472-9a29-c0532a4eb7a5)

```console
gcloud compute networks create vpc-demo --subnet-mode custom
```
**Note:** If it is needed "Authorize" your CLoud Shell.

The output should look similar to this:
![create_vpc](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a6608d34-9da6-4789-a179-b586027e6a03)

2. In Cloud Shell, create **subnet** ```vpc-demo-subnet1``` in the region **us-central1**:

![vpc-demo-subnet1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/5b40f760-bb1e-4c21-8bca-038a3bb19b4a)

```console
gcloud compute networks subnets create vpc-demo-subnet1 \
--network vpc-demo --range 10.1.1.0/24 --region us-central1
```
The output should look similar to this:
![vpc-demo-subnet](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/81accd19-25cb-4acb-9ef9-554bf6a2f6af)

3. Create subnet ```vpc-demo-subnet2``` in the region **us-east1**:

![vpc-demo-subnet2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a99c9f23-af23-448c-9833-25f2ff53d55e)

```console
gcloud compute networks subnets create vpc-demo-subnet2 \
--network vpc-demo --range 10.2.1.0/24 --region us-east1
```
4. Create a firewall rule to allow all custom traffic within the network:

![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/15be99cc-a947-42cf-a825-7e404d99fe4e)

```console
gcloud compute firewall-rules create vpc-demo-allow-custom \
  --network vpc-demo \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8
```

![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/63a89cd4-5df2-45ed-9e9c-967d311ce3fb)

5. Create a firewall rule to allow SSH, ICMP traffic from anywhere:

![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/7b9dac42-634a-422b-b9bd-23525de158be)

```console
gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp \
    --network vpc-demo \
    --allow tcp:22,icmp
```

