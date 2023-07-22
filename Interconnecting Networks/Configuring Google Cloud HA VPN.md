 # Configuring Google Cloud HA VPN

## Overview

HA VPN is a high-availability (HA) Cloud VPN solution that lets you securely connect your **on-premises** network to your VPC network through an IPsec VPN connection in a single region. HA VPN provides an SLA of 99.99% service availability.

HA VPN is a **regional** per VPC, VPN solution. HA VPN gateways have two interfaces, each with **its own public IP address**. When you create an HA VPN gateway, two public IP addresses are ***automatically*** chosen from different address pools. When HA VPN is configured with two tunnels, Cloud VPN offers a 99.99% service availability uptime.

In this workshop you create a global VPC called vpc-demo, with two custom subnets in **us-east1** and **us-central1**. In this VPC, you add a Compute Engine instance in each region. You then create a second VPC called on-prem to simulate a customer's on-premises data center. In this second VPC, you add a subnet in region us-central1 and a Compute Engine instance running in this region. Finally, you add an HA VPN and a cloud router in each VPC and run two tunnels from each HA VPN gateway before testing the configuration to verify the 99.99% SLA.

In this workshop, you create HA VPN as in this architecture that you see in the following:

![HA-VPN-Architecture](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/8a221a65-3dfe-43c5-9faf-65a26cc4d6dc)

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

![vpc_demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/279af7aa-e9f2-4810-bd57-f88df41421a1)


```console
gcloud compute networks create vpc-demo --subnet-mode custom
```
**Note:** If it is needed "Authorize" your Cloud Shell.

The output should look similar to this:


![create_vpc](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6f6ffb2e-8df7-4379-a81e-7ae3d7dbed51)


2. In Cloud Shell, create **subnet** ```vpc-demo-subnet1``` in the region **us-central1**:

![vpc-demo-subnet1-ss](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/0b6ad0db-9a53-4bb7-9765-5375f2542d85)


```console
gcloud compute networks subnets create vpc-demo-subnet1 \
--network vpc-demo --range 10.1.1.0/24 --region us-central1
```
The output should look similar to this:

![vpc-demo-subnet1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/00d15c94-a6bf-41e3-82f5-dbfa12e34728)


3. Create subnet ```vpc-demo-subnet2``` in the region **us-east1**:

![vpc-demo-subnet2-ss](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c3ccb63a-8846-4e16-8378-393c168b5baf)

```console
gcloud compute networks subnets create vpc-demo-subnet2 \
--network vpc-demo --range 10.2.1.0/24 --region us-east1
```

The output should look similar to this:

![vpc-demo-subnet2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/4d21f0a6-b268-4e40-9e57-bfff01ecbc5c)


4. Create a firewall rule to allow all custom traffic within the network:

![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/68f07db2-d986-45b2-b76e-bba71772de92)

```console
gcloud compute firewall-rules create vpc-demo-allow-custom \
  --network vpc-demo \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 10.0.0.0/8
```
The output should look similar to this:

![vpc-firewall1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/4078f433-c678-40dc-95fa-43be2a3aaa8c)

5. Create a firewall rule to allow SSH, ICMP traffic from anywhere:
 ![firewall-rule](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2f1869e3-c38e-4328-ae5a-c6827cacbc38)
  
```console
gcloud compute firewall-rules create vpc-demo-allow-ssh-icmp \
    --network vpc-demo \
    --allow tcp:22,icmp
```
The output should look similar to this:
![vpc-firewall2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c70a5e8b-0ce7-4ea0-a6a9-b5f8ad08da87)


6. Create a VM instance **vpc-demo-instance1** in zone **us-central1-b**:

![vpc-demo-instance1-ss](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/e5b6c47b-fc9b-4a38-890d-6445bc9f487e)

```console
gcloud compute instances create vpc-demo-instance1 --zone us-central1-b --subnet vpc-demo-subnet1
```
The output should look similar to this:

![vpc-demo-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a12f9bde-20cc-44da-962b-884a3fabb0fc)


7. Create a VM instance **vpc-demo-instance2** in zone **us-east1-b**:

![vpc-demo-instnce2-ss](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/1bbb7bef-3133-48c0-b9f2-5b27fe0043b5)


```console
gcloud compute instances create vpc-demo-instance2 --zone us-east1-b --subnet vpc-demo-subnet2
```
The output should look similar to this:

![vpc-demo-instance2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/793dd2fe-a072-49fd-8b1e-ea4087a84132)



By the way, we have completed this part of the architecture with that only compute engine, firewall and subnet side that is shown above:

![architecture1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/d3300c79-18ef-4535-9197-625f3de74e48)


## Task 2. Set up a simulated on-premises environment

In this task you create a VPC called **on-prem** that <ins>simulates</ins> an on-premises environment from where a customer connects to the Google cloud environment.

1. In Cloud Shell, create a VPC network called **on-prem**:

![vpc-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/21fe6f0b-ae28-4cdc-bf00-ca7676d62309)


```console
gcloud compute networks create on-prem --subnet-mode custom
```

The output should look similar to this:

![vpc-network-create-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f506dfa4-3954-4a82-b12b-e58c08d2efbb)


2. Create a subnet called **on-prem-subnet1**:

![on-prem-subnet1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/042bb11e-d681-4c46-b638-badee386399e)


```console
gcloud compute networks subnets create on-prem-subnet1 \
--network on-prem --range 192.168.1.0/24 --region us-central1
```
The output should look similar to this:

![on-prem-subnet1-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/e78ffc39-085e-4074-8bc0-218eebc66d43)


3. Create a firewall rule to allow all custom traffic within the network:

![on-prem-firewallrules](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/adf33f9c-e954-49df-a762-83de48f8e2c4)

```console
gcloud compute firewall-rules create on-prem-allow-custom \
  --network on-prem \
  --allow tcp:0-65535,udp:0-65535,icmp \
  --source-ranges 192.168.0.0/16
```
The output should look similar to this:

![on-prem-firewallrule1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2fe16977-3fca-4440-b3e0-a76361e4fea7)


4. Create a firewall rule to allow SSH, RDP, HTTP, and ICMP traffic to the instances:
   
![on-prem-firewallrules](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c9fa16a3-2158-4a5c-baad-9e3a597cc091)

```console
gcloud compute firewall-rules create on-prem-allow-ssh-icmp \
    --network on-prem \
    --allow tcp:22,icmp
```
The output should look similar to this:

![on-prem-firewall2](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/3467be53-41ba-45ca-89f7-b2fa10ebe990)

5. Create an instance called **on-prem-instance1** in the region **us-central1**:
   
![on-prem-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/42382df7-3e9c-4929-b838-111e32783b3f)

```console
gcloud compute instances create on-prem-instance1 --zone us-central1-a --subnet on-prem-subnet1
``` 
The output should look similar to this:

![on-prem-instance1-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/ec7dfe93-913d-4360-8b53-7376ef3c0f27)


## Task 3. Set up an HA VPN gateway

In this task you create an HA VPN gateway in each VPC network and then create HA VPN tunnels on each Cloud VPN gateway.

1. In Cloud Shell, create an HA VPN in the **vpc-demo network**:

![HA-VPN-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c4c396e5-0c7d-43f8-9952-5b1e1f9ab146)

```console
gcloud compute vpn-gateways create vpc-demo-vpn-gw1 --network vpc-demo --region us-central1
```
The output should look similar to this:

![HA-VPN-VPC-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/51fd30bc-cf40-4ddb-87b3-48dea1d7de84)


Note that ```INTERFACE0``` and ```INTERFACE1``` IP adresses, these are regional external IP adresses. 

2. Create an HA VPN in the **on-prem network**:

![HA-VPN-VPC-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/4b5dfd8b-39e9-4cd6-91ae-8b3803d77cb4)

```console
gcloud compute vpn-gateways create on-prem-vpn-gw1 --network on-prem --region us-central1
```
The output should look similar to this:

![HA-VPN-on-prem-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/b0362632-aeaa-4dd9-bd45-9b1404ccf447)

Note that again ```INTERFACE0``` and ```INTERFACE1``` IP adresses, these are regional external IP adresses.

![INTERFACES](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f0a0cc97-3081-41df-824c-179408afe41a)


3. View details of the **vpc-demo-vpn-gw1** gateway to verify its settings:

```console
gcloud compute vpn-gateways describe vpc-demo-vpn-gw1 --region us-central1
```
The output should look similar to this:

![vpc-demo-vpn-gw1-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/cade9db1-508c-4b7d-b278-4530dff9cbf5)


4. View details of the **on-prem-vpn-gw1** vpn-gateway to verify its settings:

```console
gcloud compute vpn-gateways describe on-prem-vpn-gw1 --region us-central1
```
The output should look similar to this:

![on-prem-vpn-gw-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/5f896c33-fe49-4693-94c7-154ad44b3df8)


### Create cloud routers

1. Create a cloud router in the **vpc-demo network**:
   
![vpc-demo-cloud-router-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/3dd0d36c-ecef-459d-b2e8-ed115d82c60f)

```console
gcloud compute routers create vpc-demo-router1 \
    --region us-central1 \
    --network vpc-demo \
    --asn 65001
```
The output should look similar to this:

![vpc-demo-cloud-router-outputt](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/169cbae8-b014-41fe-8616-77a4d2c2c8b4)

2. Create a cloud router in the on-prem network:

![on-prem-cloud-router](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/329f6f19-c7c5-4084-846a-7b2dd21c3720)


```console
gcloud compute routers create on-prem-router1 \
    --region us-central1 \
    --network on-prem \
    --asn 65002
```
The output should look similar to this:

![on-prem-cloud-router-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f672e8c8-674a-4c6c-b966-29bf1975e614)

Note that ```asn``` values different with each other, asn values 65001 on vpn-demo nd 65002 on-prem, relatively.

## Task 4. Create two VPN tunnels

In this task you create VPN tunnels between the two gateways. For HA VPN setup, you add two tunnels from each gateway to the remote setup. You create a tunnel on **interface0** and connect to **interface0** on the remote gateway. Next, you create another tunnel on **interface1** and connect to **interface1** on the remote gateway.

When you run HA VPN tunnels between two Google Cloud VPCs, you need to make sure that the tunnel on **interface0** is connected to **interface0** on the remote VPN gateway. Similarly, the tunnel on **interface1** must be connected to **interface1** on the remote VPN gateway.

In this workhop you are simulating an on-premises setup with both VPN gateways in Google Cloud. You ensure that **interface0** of one gateway connects to **interface0** of the other and **interface1** connects to **interface1** of the remote gateway.

1. Create the <ins>first</ins> VPN tunnel in the **vpc-demo network**:

![HA-VPN-GATEWAY-VPC-DEMO](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f9311d25-5918-4dcc-9b02-e40c857b4b86)

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

![vpn-tunnel-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/bc8866cb-0dc1-4e3a-ab41-ca2c2988973b)


2. Create the <ins>second</ins> VPN tunnel in the vpc-demo network:

![s2n-vpc-demo-vpn-tunnel](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a2fd8d63-af03-4284-8113-e17c1c5dead1)

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

![VPC-DEMO-2nd-VPN-TUNNEL](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/ac245d83-f826-4474-83b6-36f6d84a08c0)

3. Create the <ins>first</ins> VPN tunnel in the **on-prem network**:

![first-on-prem-vpn-tunnel](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/61d5ed2e-381e-40b4-bf64-4f1d0644eb24)

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

![1st-VPN-TUNNEL-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6619ed99-5f11-4e62-8e5e-baeb2d817005)

4. Create the <ins>second</ins> VPN tunnel in the on-prem network:

![second-vpn-tunnel-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2621fc56-6a48-4aa3-aa24-b762f776e354)

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

![second-vpn-tunnel-on-prem-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/18b0f6d4-3cea-430c-8009-810e8a005af9)

## Task 5. Create Border Gateway Protocol (BGP) peering for each tunnel

In this task you configure BGP peering for each VPN tunnel between **vpc-demo** and **VPC on-prem**. HA VPN requires dynamic routing to enable 99.99% availability.

1. Create the router interface for **tunnel0** in network **vpc-demo**:

![interface-bgp-peer-tunnel0-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/7528d4de-72bf-4592-986e-e0e892072997)

```console
gcloud compute routers add-interface vpc-demo-router1 \
    --interface-name if-tunnel0-to-on-prem \
    --ip-address 169.254.0.1 \
    --mask-length 30 \
    --vpn-tunnel vpc-demo-tunnel0 \
    --region us-central1
```
The output should look similar to this:

![tunnel0-vpc-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/fb77c8c0-2c13-47a5-9529-676fbdf99834)

2. Create the BGP peer for **tunnel0** in network **vpc-demo**:

![interface-bgp-peer-tunnel0-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/a42f972a-f3b1-4da7-9ffd-597260ddae0a)

```console
gcloud compute routers add-bgp-peer vpc-demo-router1 \
    --peer-name bgp-on-prem-tunnel0 \
    --interface if-tunnel0-to-on-prem \
    --peer-ip-address 169.254.0.2 \
    --peer-asn 65002 \
    --region us-central1
```

The output should look similar to this:

![tunnel0-BGP-pper-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/b3363dcb-f634-4b70-9f63-44e3df0538ae)

3. Create a router interface for **tunnel1** in network **vpc-demo**:
   
![interface-bgp-peer-tunnel1-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/5de3655f-44ca-4f5f-9a8c-6a43cdb7cc37)

```console
gcloud compute routers add-interface vpc-demo-router1 \
    --interface-name if-tunnel1-to-on-prem \
    --ip-address 169.254.1.1 \
    --mask-length 30 \
    --vpn-tunnel vpc-demo-tunnel1 \
    --region us-central1
```

The output should look similar to this:

![tunnel1-vpc-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/ca0e95c3-ab22-4189-921b-62bcee476f37)


4. Create the BGP peer for **tunnel1** in network **vpc-demo**:

![interface-bgp-peer-tunnel1-vpc-demo](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/95830172-d49a-4057-a854-0be28e76803c)

```console
gcloud compute routers add-bgp-peer vpc-demo-router1 \
    --peer-name bgp-on-prem-tunnel1 \
    --interface if-tunnel1-to-on-prem \
    --peer-ip-address 169.254.1.2 \
    --peer-asn 65002 \
    --region us-central1
```
The output should look similar to this:

![tunnel1-bgp-peer-vpc-demo-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/eb0a43c4-b188-42b5-b282-538bfb886ff9)


5. Create a router interface for **tunnel0** in network **on-prem**:
   
![tunnel0-on-prem-intrface-bgp-peer](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2993807e-ecc0-42fd-841f-6ad5c35cd6a8)


```console
gcloud compute routers add-interface on-prem-router1 \
    --interface-name if-tunnel0-to-vpc-demo \
    --ip-address 169.254.0.2 \
    --mask-length 30 \
    --vpn-tunnel on-prem-tunnel0 \
    --region us-central1
```

The output should look similar to this:

![tunnel-0-on-prem-interface-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2b8b6e7b-cecc-4e76-a1d7-b953dc9de0fa)


6. Create the BGP peer for **tunnel0** in network **on-prem**:
   
![tunnel0-on-prem-intrface-bgp-peer](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/f260421d-7b0d-4fd5-9b06-52ef120893c4)


```console
gcloud compute routers add-bgp-peer on-prem-router1 \
    --peer-name bgp-vpc-demo-tunnel0 \
    --interface if-tunnel0-to-vpc-demo \
    --peer-ip-address 169.254.0.1 \
    --peer-asn 65001 \
    --region us-central1
```

7. Create a router interface for **tunnel1** in network **on-prem**:

![tunnel1-interface-bgp-peer-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/c1ba1e4f-eff8-4252-9214-9b96b983a78a)


```console
gcloud compute routers add-interface  on-prem-router1 \
    --interface-name if-tunnel1-to-vpc-demo \
    --ip-address 169.254.1.2 \
    --mask-length 30 \
    --vpn-tunnel on-prem-tunnel1 \
    --region us-central1
```

The output should look similar to this:

![tunnel1-on-prem-interface-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/85bd06af-0eb8-4f38-9ac7-c23c34d4b803)


8. Create the BGP peer for **tunnel1** in network **on-prem**:

![tunnel1-interface-bgp-peer-on-prem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/8ab55b39-d463-43f8-a172-ef65e15d3b58)

```console
gcloud compute routers add-bgp-peer  on-prem-router1 \
    --peer-name bgp-vpc-demo-tunnel1 \
    --interface if-tunnel1-to-vpc-demo \
    --peer-ip-address 169.254.1.1 \
    --peer-asn 65001 \
    --region us-central1
```
The output should look similar to this:

![tunnel1-bgp-peer-on-prem-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/fd7a8d63-2d90-46d0-9370-0ce27e748570)


## Task 6. Verify router configurations

In this task you verify the router configurations in both VPCs. You ***configure firewall rules to allow traffic between each VPC*** and verify the status of the tunnels. You also verify private connectivity over VPN between each VPC and enable global routing mode for the VPC.

1. View details of Cloud Router **vpc-demo-router1** to verify its settings:

```console
gcloud compute routers describe vpc-demo-router1 \
    --region us-central1
```

The output should look similar to this:

![vpc-demo-router1-verify-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/1d1bd48b-a0ca-4451-87f7-3936e15c8efa)


2. View details of Cloud Router **on-prem-router1** to verify its settings:

```console
gcloud compute routers describe on-prem-router1 \
    --region us-central1
```

The output should look similar to this:

![on-prem-router1-verify-output](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/725c4397-8c23-470d-a7ac-f4d4662a0d29)


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


![remotevpc-firewall](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/b1bbb5e5-9ae1-49b2-887c-eaccfb748d9c)


2. Allow traffic from **vpc-demo** to network VPC **on-prem**:

```console
gcloud compute firewall-rules create on-prem-allow-subnets-from-vpc-demo \
    --network on-prem \
    --allow tcp,udp,icmp \
    --source-ranges 10.1.1.0/24,10.2.1.0/24
```

The output should look similar to this:

![firewallfomvpcdemotoonprem](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/3b43c2ec-c252-421a-b766-74cec3fa89b3)


### Verify the status of the tunnels

1. List the VPN tunnels you just created:

```console
gcloud compute vpn-tunnels list
```

There should be four VPN tunnels (two tunnels for each VPN gateway). The output should look similar to this:

![list-of-vpn-tunnels](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/26afba13-1316-41dc-b0d2-a65c191306c5)


2. Verify that **vpc-demo-tunnel0** tunnel is up:

```console
gcloud compute vpn-tunnels describe vpc-demo-tunnel0 \
      --region us-central1
```

The tunnel output should show detailed status as ***Tunnel is up and running***.

![vpc-demo-tunnel0](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/e888dbd9-d9d9-46e2-90f7-35c94ebb7eea)


3. Verify that ***vpc-demo-tunnel1** tunnel is up:

```console
gcloud compute vpn-tunnels describe vpc-demo-tunnel1 \
      --region us-central1
```

The tunnel output should show detailed status as Tunnel is up and running.

![vcp-demo-tunnel1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/483748f5-f74f-4ca9-9585-ea37de39ffdc)


4. Verify that **on-prem-tunnel0** tunnel is up:

```console
gcloud compute vpn-tunnels describe on-prem-tunnel0 \
      --region us-central1
```

The tunnel output should show detailed status as Tunnel is up and running.

![on-prem-tunnel0](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/44528221-140b-4cb7-9e0f-11d658317130)


5. Verify that **on-prem-tunnel1** tunnel is up:

```console
gcloud compute vpn-tunnels describe on-prem-tunnel1 \
      --region us-central1
```

The tunnel output should show detailed status as Tunnel is up and running.

![on-prem-tunnel1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/9281f4eb-d2a3-40de-80a3-d3ce1c793ee0)

### Verify private connectivity over VPN

1. Open a new Cloud Shell tab and type the following to connect via SSH to the instance **on-prem-instance1**:

```console
gcloud compute ssh on-prem-instance1 --zone us-central1-a
```

The output should look similar to this:

![ssh-on-prem-instance1](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/9bdad406-7e50-47c1-93fc-aedffdfa2710)



2.Type "y" to confirm that you want to continue.
3. Press **Enter** twice to skip creating a password.
4. From the instance **on-prem-instance1** in network **on-prem**, to reach instances in network **vpc-demo**, ping 10.1.1.2:

```console
ping -c 4 10.1.1.2
```

Pings are successful. The output should look similar to this:

![ping](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/fc00085f-5673-4e3d-a2dd-d1bd74274bca)


### Global routing with VPN

HA VPN is a ***regional*** resource and cloud router that by default <ins>only sees</ins> the routes in the ***region*** in which it is deployed. To reach instances in a ***different region*** than the cloud router, you <ins>need to enable global routing mode for the VPC</ins>. This allows the cloud router to see and advertise routes from other regions.

1. Open a new Cloud Shell tab and update the **bgp-routing mode** from **vpc-demo** to **GLOBAL**:

```console
gcloud compute networks update vpc-demo --bgp-routing-mode GLOBAL
```

The output should look similar to this:

![global](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/5200cb17-ab1d-4849-a484-761495964457)


2. Verify the change:

```console
gcloud compute networks describe vpc-demo
```

The output should look similar to this:

![verify-global](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/2dd3593d-4b0a-420c-91a2-48d63a06932a)


3. From the Cloud Shell tab that is currently connected to the instance in network **on-prem** via **ssh**, ping the instance **vpc-demo-instance2** in region us-east1:

```console
ping -c 2 10.2.1.2
```
Pings are successful. The output should look similar to this:

![ping-to-another-region](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/ede67a76-dbb5-417f-a908-e9b02b418446)


## Task 7. Verify and test the configuration of HA VPN tunnels

In this task you will test and verify that the high availability configuration of each HA VPN tunnel is successful.

1. Open a new Cloud Shell tab.
2. Bring **tunnel0** in network **vpc-demo** down:

```console
gcloud compute vpn-tunnels delete vpc-demo-tunnel0  --region us-central1
```

Respond "y" when asked to verify the deletion. The respective **tunnel0** in network **on-prem** will go down.

The output should look similar to this:

![tunnel-deleted](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/6485cfdc-8b2f-4a4c-b037-8d939e637edb)


3. Verify that the tunnel is down:

```console
gcloud compute vpn-tunnels describe on-prem-tunnel0  --region us-central1
```

The detailed status should show as ***Handshake_with_peer_broken***.

![verify-tunnel-is-down](https://github.com/nildenist/Elastic-Google-Cloud-Infrastructure-Scaling-and-Automation/assets/28653377/9d47d1ec-265a-4386-b75c-293007ce21c7)

4. Switch to the previous Cloud Shell tab that has the open **ssh** session running, and verify the pings between the instances in network **vpc-demo** and network **on-prem**:

```console
ping -c 3 10.1.1.2
```
Pings are still successful because the traffic is now sent over the second tunnel. You have successfully configured HA VPN tunnels.

## Task 9: Review

In this workshop you configured HA VPN gateways. You also configured dynamic routing with VPN tunnels and configured global dynamic routing mode. Finally you verified that HA VPN is configured and functioning correctly.




