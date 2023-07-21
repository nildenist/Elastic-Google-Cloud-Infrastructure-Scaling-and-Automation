# Configuring Google Cloud HA VPN

## Overview

HA VPN is a high-availability (HA) Cloud VPN solution that lets you securely connect your **on-premises** network to your VPC network through an IPsec VPN connection in a single region. HA VPN provides an SLA of 99.99% service availability.

HA VPN is a **regional** per VPC, VPN solution. HA VPN gateways have two interfaces, each with **its own public IP address**. When you create an HA VPN gateway, two public IP addresses are ***automatically*** chosen from different address pools. When HA VPN is configured with two tunnels, Cloud VPN offers a 99.99% service availability uptime.

In this workshop you create a global VPC called vpc-demo, with two custom subnets in **us-east1** and **us-central1**. In this VPC, you add a Compute Engine instance in each region. You then create a second VPC called on-prem to simulate a customer's on-premises data center. In this second VPC, you add a subnet in region us-central1 and a Compute Engine instance running in this region. Finally, you add an HA VPN and a cloud router in each VPC and run two tunnels from each HA VPN gateway before testing the configuration to verify the 99.99% SLA.

In this workshop, you create HA VPN as in this architecture that you see in the following:

![architecture-of-HA-VPN](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/339b5c96-42fd-42de-823c-0c981850d306)

You will see that this architecture is splitted for each task with the related shell code. This will help your understanding for codes in which part you dealed with. 

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

![vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f6960399-eb0e-43b4-b06b-3447e364c804)


2. In Cloud Shell, create **subnet** ```vpc-demo-subnet1``` in the region **us-central1**:

![vpc-demo-subnet1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/5b40f760-bb1e-4c21-8bca-038a3bb19b4a)

```console
gcloud compute networks subnets create vpc-demo-subnet1 \
--network vpc-demo --range 10.1.1.0/24 --region us-central1
```
The output should look similar to this:

![vpc-demo-subnet1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/71bc811f-fd0f-4aad-bd23-08cdedd9f974)


3. Create subnet ```vpc-demo-subnet2``` in the region **us-east1**:

![vpc-demo-subnet2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a99c9f23-af23-448c-9833-25f2ff53d55e)

```console
gcloud compute networks subnets create vpc-demo-subnet2 \
--network vpc-demo --range 10.2.1.0/24 --region us-east1
```

The output should look similar to this:

![vpc-demo-subnet2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/eb69edd0-f24c-41f2-965b-510c77165502)


4. Create a firewall rule to allow all custom traffic within the network:

![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/15be99cc-a947-42cf-a825-7e404d99fe4e)

```console
gcloud compute firewall-rules create vpc-demo-allow-custom \
  --network vpc-demo \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8
```
The output should look similar to this:

![vpc-firewall1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/ad820fb0-5840-4707-8c0b-b3aa16fd9156)


5. Create a firewall rule to allow SSH, ICMP traffic from anywhere:

![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/7b9dac42-634a-422b-b9bd-23525de158be)

```console
gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp \
    --network vpc-demo \
    --allow tcp:22,icmp
```
The output should look similar to this:

![vpc-firewall2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/98cf287c-eed2-4254-be39-dde8eea7d70d)

6. Create a VM instance **vpc-demo-instance1** in zone **us-central1-b**:

![vpc-demo-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6af0da31-437d-4bc9-8e6e-6bdfb099fe13)

```console
gcloud compute instances create vpc-demo-instance1 --zone us-central1-b --subnet vpc-demo-subnet1
```
The output should look similar to this:

![vpc-demo-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/9fdc3396-2580-4df7-8872-e21282bd79ad)

7. Create a VM instance **vpc-demo-instance2** in zone **us-east1-b**:

![vpc-demo-instance2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/faf0a641-4d77-4314-bec2-c5008e1416c1)

```console
gcloud compute instances create vpc-demo-instance2 --zone us-east1-b --subnet vpc-demo-subnet2
```
The output should look similar to this:

![vpc-demo-instance2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/b99dcb89-1837-4b28-94f9-13f90d7e9b74)

By the way, we have completed this part of the architecture with that only compute engine, firewall and subnet side that is shown above:
![architecture1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f5f7dbf3-32d7-44bc-972c-a7f75f0e2f36)


## Task 2. Set up a simulated on-premises environment

In this task you create a VPC called **on-prem** that <ins>simulates</ins> an on-premises environment from where a customer connects to the Google cloud environment.

1. In Cloud Shell, create a VPC network called **on-prem**:

![vpc-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/1b04eb8d-cc87-4b43-94f5-a6b7f4db881f)

```console
gcloud compute networks create on-prem --subnet-mode custom
```

The output should look similar to this:

![vpc-network-create-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/51437438-4f91-409f-885f-fe829c90bb81)

2. Create a subnet called **on-prem-subnet1**:

![on-prem-subnet1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/1baf6203-39ef-4b98-baba-dffea6a96534)

```console
gcloud compute networks subnets create on-prem-subnet1 \
--network on-prem --range 192.168.1.0/24 --region us-central1
```
The output should look similar to this:

![on-prem-subnet1-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6a0c4b32-3924-434e-b8e8-3491dba12236)

3. Create a firewall rule to allow all custom traffic within the network:

![on-prem-firewallrules](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/86e2beb3-6035-4672-bd37-e78fb703decf)

```console
gcloud compute firewall-rules create on-prem-allow-custom \
  --network on-prem \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 192.168.0.0/16
```
The output should look similar to this:

![on-prem-firewallrule1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a6104456-f29e-4a5e-b3d4-c422b7ea7e0d)

4. Create a firewall rule to allow SSH, RDP, HTTP, and ICMP traffic to the instances:
   
![on-prem-firewallrules](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/e27ebbca-fee4-46bd-8557-ebc8d5e5f2aa)

```console
gcloud compute firewall-rules create on-prem-allow-ssh-icmp \
    --network on-prem \
    --allow tcp:22,icmp
```
The output should look similar to this:

![on-prem-firewall2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/60540d4d-2070-4fae-812a-5d3cc3ba5b77)


5. Create an instance called **on-prem-instance1** in the region **us-central1**:
   
![on=prem-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f5b3f00e-f18c-4c2b-ac7a-024d4a5ac3a9)

```console
gcloud compute instances create on-prem-instance1 --zone us-central1-a --subnet on-prem-subnet1
``` 
The output should look similar to this:

![on-prem-instance1-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0ee45128-208f-4e21-ab96-ef65f243369b)

## Task 3. Set up an HA VPN gateway

In this task you create an HA VPN gateway in each VPC network and then create HA VPN tunnels on each Cloud VPN gateway.

1. In Cloud Shell, create an HA VPN in the **vpc-demo network**:

![HA-VPN-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/74b80df9-a6bb-4130-a34e-1599af4f4378)

```console
gcloud compute vpn-gateways create vpc-demo-vpn-gw1 --network vpc-demo --region us-central1
```
The output should look similar to this:

![HA-VPN-VPC-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/278fb5ac-a481-499e-9637-c71298edf1ba)

Note that ```INTERFACE0``` and ```INTERFACE1``` IP adresses, these are regional external IP adresses. 

2. Create an HA VPN in the **on-prem network**:

![HA-VPN-VPC-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/89bd2f4e-07d4-4e0b-9c14-857a09fae96a)

```console
gcloud compute vpn-gateways create on-prem-vpn-gw1 --network on-prem --region us-central1
```
The output should look similar to this:

![HA-VPN-on-prem-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/3d3a6623-59c2-48b4-b815-d064cce0ce3c)

Note that again ```INTERFACE0``` and ```INTERFACE1``` IP adresses, these are regional external IP adresses.

![INTERFACES](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/9eccbf24-cd2a-425a-a2b2-f453304d9e37)


3. View details of the **vpc-demo-vpn-gw1** gateway to verify its settings:

```console
gcloud compute vpn-gateways describe vpc-demo-vpn-gw1 --region us-central1
```
The output should look similar to this:

![vpc-demo-vpn-gw1-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0f5718fd-0be0-42ad-9177-98d14ffbdc78)

4. View details of the **on-prem-vpn-gw1** vpn-gateway to verify its settings:

```console
gcloud compute vpn-gateways describe on-prem-vpn-gw1 --region us-central1
```
The output should look similar to this:

![on-prem-vpn-gw-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a383f999-24d8-4f53-ab73-14752f79cd88)

### Create cloud routers

1. Create a cloud router in the **vpc-demo network**:
   
![vpc-demo-cloud-router-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/fef56d88-4afd-464b-8a72-2a25248b917e)

```console
gcloud compute routers create vpc-demo-router1 \
    --region us-central1 \
    --network vpc-demo \
    --asn 65001
```
The output should look similar to this:

![vpc-demo-cloud-router-outputt](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/41d2b861-637a-4ef3-a907-6bd3cc0402c6)

2. Create a cloud router in the on-prem network:

![on-prem-cloud-router](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0d328b4b-8ea8-4e7a-97d9-c192c522f6a7)

```console
gcloud compute routers create on-prem-router1 \
    --region us-central1 \
    --network on-prem \
    --asn 65002
```
The output should look similar to this:

![on-prem-cloud-router-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/77a07219-a52b-48d5-96d5-282afde78ba8)


Note that ```asn``` values different with each other, asn values 65001 on vpn-demo nd 65002 on-prem, relatively.

## Task 4. Create two VPN tunnels

In this task you create VPN tunnels between the two gateways. For HA VPN setup, you add two tunnels from each gateway to the remote setup. You create a tunnel on **interface0** and connect to **interface0** on the remote gateway. Next, you create another tunnel on **interface1** and connect to **interface1** on the remote gateway.

When you run HA VPN tunnels between two Google Cloud VPCs, you need to make sure that the tunnel on **interface0** is connected to **interface0** on the remote VPN gateway. Similarly, the tunnel on **interface1** must be connected to **interface1** on the remote VPN gateway.

In this workhop you are simulating an on-premises setup with both VPN gateways in Google Cloud. You ensure that **interface0** of one gateway connects to **interface0** of the other and **interface1** connects to **interface1** of the remote gateway.

1. Create the <ins>first</ins> VPN tunnel in the **vpc-demo network**:

![HA-VPN-GATEWAY-VPC-DEMO](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/726c21e4-3c7a-4650-8a66-53effd0447c9)

```console
gcloud compute vpn-tunnels create vpc-demo-tunnel0 \
    --peer-gcp-gateway on-prem-vpn-gw1 \
    --region us-central1 \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router vpc-demo-router1 \
    --vpn-gateway vpc-demo-vpn-gw1 \
    --interface 0
```

The output should look similar to this:

![vpn-tunnel-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0989d6c0-d932-4cee-a4a8-6e566e894c1e)

2. Create the <ins>second</ins> VPN tunnel in the vpc-demo network:

![s2n-vpc-demo-vpn-tunnel](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/cf59ef2b-98b2-4a27-870d-32845988d726)


```console
gcloud compute vpn-tunnels create vpc-demo-tunnel1 \
    --peer-gcp-gateway on-prem-vpn-gw1 \
    --region us-central1 \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router vpc-demo-router1 \
    --vpn-gateway vpc-demo-vpn-gw1 \
    --interface 1
```

The output should look similar to this:

![VPC-DEMO-2nd-VPN-TUNNEL](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/3e2d9df0-9959-4328-b762-7345d79fb6b3)

3. Create the <ins>first</ins> VPN tunnel in the **on-prem network**:

![first-on-prem-vpn-tunnel](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/4d75f19c-8d1d-458b-9b65-a35bd9b5b992)

```console
gcloud compute vpn-tunnels create on-prem-tunnel0 \
    --peer-gcp-gateway vpc-demo-vpn-gw1 \
    --region us-central1 \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router on-prem-router1 \
    --vpn-gateway on-prem-vpn-gw1 \
    --interface 0
```

The output should look similar to this:

![1st-VPN-TUNNEL-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/3eb471d0-3787-4d74-9803-7f76de580ad8)


4. Create the <ins>second</ins> VPN tunnel in the on-prem network:

![second-vpn-tunnel-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/25191577-2928-4f8d-b121-4f5759f238c6)

```console
gcloud compute vpn-tunnels create on-prem-tunnel1 \
    --peer-gcp-gateway vpc-demo-vpn-gw1 \
    --region us-central1 \
    --ike-version 2 \
    --shared-secret [SHARED_SECRET] \
    --router on-prem-router1 \
    --vpn-gateway on-prem-vpn-gw1 \
    --interface 1
```

The output should look similar to this:

![second-vpn-tunnel-on-prem-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/01e00afa-91fb-4d69-a7c3-f3c6fdf457df)

## Task 5. Create Border Gateway Protocol (BGP) peering for each tunnel

In this task you configure BGP peering for each VPN tunnel between **vpc-demo** and **VPC on-prem**. HA VPN requires dynamic routing to enable 99.99% availability.

1. Create the router interface for **tunnel0** in network **vpc-demo**:

![interface-bgp-peer-tunnel0-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0942b6b0-2c90-486c-8e2e-e9ca65619a05)


```console
gcloud compute routers add-interface vpc-demo-router1 \
    --interface-name if-tunnel0-to-on-prem \
    --ip-address 169.254.0.1 \
    --mask-length 30 \
    --vpn-tunnel vpc-demo-tunnel0 \
    --region us-central1
```
The output should look similar to this:

![tunnel0-vpc-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/fd02eb8b-3cd7-4f80-aa7a-f20c91413b7b)

2. Create the BGP peer for **tunnel0** in network **vpc-demo**:

![interface-bgp-peer-tunnel0-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/66f523a1-a667-4198-b5ba-15119197025c)


```console
gcloud compute routers add-bgp-peer vpc-demo-router1 \
    --peer-name bgp-on-prem-tunnel0 \
    --interface if-tunnel0-to-on-prem \
    --peer-ip-address 169.254.0.2 \
    --peer-asn 65002 \
    --region us-central1
```

The output should look similar to this:

![tunnel0-BGP-peer-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f10450e8-7b33-4520-8b37-cb937b853020)

3. Create a router interface for **tunnel1** in network **vpc-demo**:
   
![interface-bgp-peer-tunnel1-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/bd685d44-d599-4243-9484-14c4932d5fda)

```console
gcloud compute routers add-interface vpc-demo-router1 \
    --interface-name if-tunnel1-to-on-prem \
    --ip-address 169.254.1.1 \
    --mask-length 30 \
    --vpn-tunnel vpc-demo-tunnel1 \
    --region us-central1
```

The output should look similar to this:

![3tunnel1-vpc-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/714a796f-5fc8-4e2b-9a68-576782a69633)

4. Create the BGP peer for **tunnel1** in network **vpc-demo**:

![interface-bgp-peer-tunnel1-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6acc3f77-0cc7-44c0-a87c-a2636d02ba5f)

```console
gcloud compute routers add-bgp-peer vpc-demo-router1 \
    --peer-name bgp-on-prem-tunnel1 \
    --interface if-tunnel1-to-on-prem \
    --peer-ip-address 169.254.1.2 \
    --peer-asn 65002 \
    --region us-central1
```
The output should look similar to this:

![tunnel1-bgp-peer-vpc-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/1d662392-7666-436c-843e-491c41fc55a0)

5. Create a router interface for **tunnel0** in network **on-prem**:
6. 
![tunnel0-on-prem-intrface-bgp-peer](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6a8b5cb0-e999-4926-9aff-1b9665196757)

```console
gcloud compute routers add-interface on-prem-router1 \
    --interface-name if-tunnel0-to-vpc-demo \
    --ip-address 169.254.0.2 \
    --mask-length 30 \
    --vpn-tunnel on-prem-tunnel0 \
    --region us-central1
```

The output should look similar to this:

![tunnel-0-on-prem-interface-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c29c3f4f-efd6-47fa-a62b-e5a4495e51ea)

6. Create the BGP peer for **tunnel0** in network **on-prem**:
   
![tunnel0-on-prem-intrface-bgp-peer](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c4882264-fc2b-4990-9397-e3611dbd19c6)

```console
gcloud compute routers add-bgp-peer on-prem-router1 \
    --peer-name bgp-vpc-demo-tunnel0 \
    --interface if-tunnel0-to-vpc-demo \
    --peer-ip-address 169.254.0.1 \
    --peer-asn 65001 \
    --region us-central1
```

7. Create a router interface for **tunnel1** in network **on-prem**:

![tunnel1-interface-bgp-peer-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/350ccb9a-3f54-4b35-9b71-3b9f467d3177)

```console
gcloud compute routers add-interface  on-prem-router1 \
    --interface-name if-tunnel1-to-vpc-demo \
    --ip-address 169.254.1.2 \
    --mask-length 30 \
    --vpn-tunnel on-prem-tunnel1 \
    --region us-central1
```

The output should look similar to this:

![tunnel1-on-prem-interface-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c589aaab-6bb2-44cd-948f-9c5f12325717)

8. Create the BGP peer for **tunnel1** in network **on-prem**:
   
![tunnel1-interface-bgp-peer-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/98bc7bfd-22df-4cbd-808a-ceeeac39b9fb)

```console
gcloud compute routers add-bgp-peer  on-prem-router1 \
    --peer-name bgp-vpc-demo-tunnel1 \
    --interface if-tunnel1-to-vpc-demo \
    --peer-ip-address 169.254.1.1 \
    --peer-asn 65001 \
    --region us-central1
```
The output should look similar to this:

![tunnel1-bgp-peer-on-prem-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/aedd8063-b5cc-4cfc-aa6c-afc2787ac51a)

## Task 6. Verify router configurations

In this task you verify the router configurations in both VPCs. You ***configure firewall rules to allow traffic between each VPC*** and verify the status of the tunnels. You also verify private connectivity over VPN between each VPC and enable global routing mode for the VPC.

1. View details of Cloud Router **vpc-demo-router1** to verify its settings:

```console
gcloud compute routers describe vpc-demo-router1 \
    --region us-central1
```

The output should look similar to this:

![vpc-demo-router1-verify-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/ad0154a9-8bac-4468-adb1-b29557fd5479)

2. View details of Cloud Router **on-prem-router1** to verify its settings:

```console
gcloud compute routers describe on-prem-router1 \
    --region us-central1
```

The output should look similar to this:

![on-prem-router1-verify-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/bc0126f2-de86-4965-928d-590c58dda797)

### Configure firewall rules to allow traffic from the remote VPC

Configure firewall rules to allow traffic from the private IP ranges of peer VPN.

1. Allow traffic from network VPC **on-prem** to **vpc-demo**:

```console
gcloud compute firewall-rules create vpc-demo-allow-subnets-from-on-prem \
    --network vpc-demo \
    --allow tcp,udp,icmp \
    --source-ranges 192.168.1.0/24
```

The output should look similar to this:

![remotevpc-firewall](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0a1a3383-0431-4c66-827a-f52d3241e5b9)


2. Allow traffic from **vpc-demo** to network VPC **on-prem**:

```console
gcloud compute firewall-rules create on-prem-allow-subnets-from-vpc-demo \
    --network on-prem \
    --allow tcp,udp,icmp \
    --source-ranges 10.1.1.0/24,10.2.1.0/24
```

The output should look similar to this:

![firewallfomvpcdemotoonprem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/08178af7-1677-4cba-bd35-85b8fe7a75b0)

### Verify the status of the tunnels

1. List the VPN tunnels you just created:

```console
gcloud compute vpn-tunnels list
```

There should be four VPN tunnels (two tunnels for each VPN gateway). The output should look similar to this:

![list-of-vpn-tunnels](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/7ca0f77b-b86f-476f-a8ff-ecea16e323e6)

2. Verify that **vpc-demo-tunnel0** tunnel is up:

```console
gcloud compute vpn-tunnels describe vpc-demo-tunnel0 \
      --region us-central1
```

The tunnel output should show detailed status as ***Tunnel is up and running***.

![vpc-demo-tunnel0](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/60f37ee3-dc91-4344-bbad-ce2005b526d5)

3. Verify that ***vpc-demo-tunnel1** tunnel is up:

```console
gcloud compute vpn-tunnels describe vpc-demo-tunnel1 \
      --region us-central1
```

The tunnel output should show detailed status as Tunnel is up and running.

![vcp-demo-tunnel1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/25b0c3f4-36ef-4dc1-a399-d8063132642e)

4. Verify that **on-prem-tunnel0** tunnel is up:

```console
gcloud compute vpn-tunnels describe on-prem-tunnel0 \
      --region us-central1
```

The tunnel output should show detailed status as Tunnel is up and running.

![on-prem-tunnel0](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/06000cdf-9e8c-402d-9980-7228c46e3889)

5. Verify that **on-prem-tunnel1** tunnel is up:

```console
gcloud compute vpn-tunnels describe on-prem-tunnel1 \
      --region us-central1
```

The tunnel output should show detailed status as Tunnel is up and running.

![on-prem-tunnel1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/b239ca36-8f44-4e95-8c84-5c69c743fe65)


### Verify private connectivity over VPN

1. Open a new Cloud Shell tab and type the following to connect via SSH to the instance **on-prem-instance1**:

```console
gcloud compute ssh on-prem-instance1 --zone us-central1-a
```

The output should look similar to this:

![ssh-on-prem-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2435c6fd-9354-4dcb-91af-021bbdd7c751)


2.Type "y" to confirm that you want to continue.
3. Press **Enter** twice to skip creating a password.
4. From the instance **on-prem-instance1** in network **on-prem**, to reach instances in network **vpc-demo**, ping 10.1.1.2:

```console
ping -c 4 10.1.1.2
```

Pings are successful. The output should look similar to this:

![ping](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2439e809-2096-470d-8672-4b2823957a57)

### Global routing with VPN

HA VPN is a ***regional*** resource and cloud router that by default <ins>only sees</ins> the routes in the ***region*** in which it is deployed. To reach instances in a ***different region*** than the cloud router, you <ins>need to enable global routing mode for the VPC</ins>. This allows the cloud router to see and advertise routes from other regions.

1. Open a new Cloud Shell tab and update the **bgp-routing mode** from **vpc-demo** to **GLOBAL**:

```console
gcloud compute networks update vpc-demo --bgp-routing-mode GLOBAL
```

The output should look similar to this:

![global](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0b4bccf7-e0f1-4ec2-a54f-61c5039e967b)

2. Verify the change:

```console
gcloud compute networks describe vpc-demo
```

The output should look similar to this:

![verify-global](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a77a5b57-5df5-4cdb-af03-1a80e58c75eb)

3. From the Cloud Shell tab that is currently connected to the instance in network **on-prem** via **ssh**, ping the instance **vpc-demo-instance2** in region us-east1:

```console
ping -c 2 10.2.1.2
```
Pings are successful. The output should look similar to this:

![ping-to-another-region](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/dc965938-da2b-45d6-979e-33ef042e9e25)

## Task 7. Verify and test the configuration of HA VPN tunnels

In this task you will test and verify that the high availability configuration of each HA VPN tunnel is successful.

1. Open a new Cloud Shell tab.
2. Bring **tunnel0** in network **vpc-demo** down:

```console
gcloud compute vpn-tunnels delete vpc-demo-tunnel0  --region us-central1
```

Respond "y" when asked to verify the deletion. The respective **tunnel0** in network **on-prem** will go down.

The output should look similar to this:

![tunnel-deleted](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/dc5cd458-2c6c-4c68-a419-3668a89220b9)

3. Verify that the tunnel is down:

```console
gcloud compute vpn-tunnels describe on-prem-tunnel0  --region us-central1
```

The detailed status should show as ***Handshake_with_peer_broken***.

![verify-tunnel-is-down](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a2b6918b-8fa5-493d-91a1-394e1dcaf235)


4. Switch to the previous Cloud Shell tab that has the open **ssh** session running, and verify the pings between the instances in network **vpc-demo** and network **on-prem**:

```console
ping -c 3 10.1.1.2
```
Pings are still successful because the traffic is now sent over the second tunnel. You have successfully configured HA VPN tunnels.

## Task 9: Review

In this workshop you configured HA VPN gateways. You also configured dynamic routing with VPN tunnels and configured global dynamic routing mode. Finally you verified that HA VPN is configured and functioning correctly.




