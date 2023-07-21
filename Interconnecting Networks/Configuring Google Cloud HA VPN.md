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

1. Create the first VPN tunnel in the **vpc-demo network**:

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

2. Create the second VPN tunnel in the vpc-demo network:

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


4. Create the second VPN tunnel in the on-prem network:

