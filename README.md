# Transit Gateway, a walkthrough

This walkthrough shows how to setup Transit Gateway with multiple VPC and Routing domains as well as connect the Transit Gateway to the Datacenter via VPN.

![Speficy Details Screenshot](./images/HybridDiagram-TGW.png)

## Introduction
When building a multi-VPC and/or multi-account architecture there are several services that we need to consider to proivde seamless integration between our AWS environment and the existing infrastrucutre in our datacenter.
Foundationally, we need to provide robust connectivity and routing between the datacenter and all of the VPCs. But we also need to provide and control routing between those VPC. For example we may have a 'Shared Services' VPC that every other VPC needs access to where we place common resources that everyone needs, such as a NAT Gateway Service to access the internet. At the same time, we dont want just any VPC talking to any other VPC. In this case, we don't want our 'Non-Production' VPCs talking to our 'Production' VPCs.

In the past, customers used thrid-party solutions and/or transit VPCs that they build and managed. In order to remove much of that undifferentiated heavy lifting, we will use **AWS Transit Gateway** Service to provide this connectivty and routing. **AWS Transit Gateway** is a service that enables custoemrs to connect their Amazon Virtual Private Clouds(VPCs) and their on-premise networks to a single highly-available gateway. **AWS Transit Gateway** provides easier connectivity, better visibility, more control, and on-demand bandwidth.

After we have connectivity and routing, we need to provide seamless DNS resolution between our Datacenter the VPCs. Our on-prem devices will want to reach out to our resources in the cloud using DNS names, not IP addresses and the resources in the cloud will want to do the same for servers back in our datacenter. Avoiding hard-coding IP addresses in our applications is best practice. To do this we will use **Amazon Route53 Resolver**. **Amazon Route53 Resolver** for hybrid clouds allows us to create highly-available endpoints in our VPCs to integrate with the Amazon Provided DNS (sometmes referred to as the .2 resolver, since it is always 2 addresses up from the VPC CIDR block. i.e. 172.16.0.2 for VPC CIDR 172.16.0.0/24)

## Planning
### VPC layout
There are lots of choices for VPC and Account Architectures, and this is mostly out-of-scope for this workshop. Take a look at what Androski Spicer presented at re:invent 2018 in his [From One to Many: Evolving VPC Design](https://www.youtube.com/watch?v=8K7GZFff_V0 "youtube video") session.
In our case, we are going to provide three types of VPCs:
1. **Non-production VPCs**: We might create several of these to house our training environments, development, and QA resources.
1. **Prodcution VPCs**: This is for our live production systems.
1. **Shared Resources**: For resources and services that we want shared across all VPCs.
1. **Datacenter**: In this workshop we need to simulate a datacenter. In the real world, this would be our existing datacenter or colo and the hardware it contains. But we are going to make our own version in the cloud!

### IP addressing
Carving up and assigning private IP address(RFC 1918 addresses) space is big subject and can be daunting of you have a large enterprise today, espeically with mergers. Even when you have a centralized IP address management system (IPAM), you will find undocumented address space being used and sometimes finding useable space is difficult. However we want to find large non-fragmented spaces so we can create a well-summerized network. Don't laugh, we all like a challenge, right?
In our case we found that the 10.0.0.0/11 space was available (I know fiction, right?). So, we are going to carve up /13's for our production and non-production and we will grab a /16's for our shared service and a /16 for our simulated datacenter.
What does that mean? 
1. Non-Production CIDR: 10.16.0.0/13 is 10.16.0.0 - 10.23.255.255. which we can carve into eight /16 subnets, one for each of our VPCs.
1. Production CIDR: 10.8.0.0/13 is 10.8.0.0 - 10.15.255.255. which we can also carve into eight /16 subnets.
1. Shared Service CIDR: 10.0.0.0/16 - 10.0.0.0 - 10.0.255.255. which we will use for 1 Datacenter Services VPC.
1. Simulated Datacenter CIDR: 10.4.0.0/16 - 10.4.0.0 - 10.4.255.255. which will be our Datacenter VPC.

### Connectivity
For connectivity between VPCs, AWS Transit Gateway make life easy. We 


## Getting Started
First, we need to get our infrastructure in place. The following Cloudformation Template will build out *five* VPCs. In ourder to do that we will first remove the default VPC. *note: if you ever remove you default VPC in your own account, you can recreate it via the console or the CLI see the [documentation](https://docs.aws.amazon.com/vpc/latest/userguide/default-vpc.html#create-default-vpc "AWS Default VPC Documentation").

### Pick Region
Since we will be deploying Cloud9 into our Datacenter VPC, we need to pick one of the following Regions:
- N. Virginia (us-east-1)
- Ohio (us-east-2)
- Oregon (us-west-2)
- Ireland (eu-west-1)
- Singapore (ap-southeast-1)

### 1. Delete Default VPC
<details>
<summary>HOW TO Delete Default VPC</summary><p>

1. In the AWS Management Console change to the region you plan to work in and change. This is in the upper right hand drop down menu.

1. In the AWS Management Console choose **Services** then select **VPC**.

1. From the left-hand menu select **Your VPCs**.

1.  In the main panel, the checkbox next to only VPC (the default VPC) should be highlighted. You can verify this is the Default VPC by scrolling to the right. The *Default VPC* column will be maked with **Yes**.

1. With our Default VPC checked select the **Actions** dropdown above it and select **Delete VPC**.

1. In the *Delete VPC* Panel, check the box 'I Acknowledge that I want to delete my default VPC.' and click the **Delete VPC** button in the bottomm right.

1. You should gett a green highlighted Dialog stating 'The VPC was deleted' and you can click **Close**. *If it is Red, then likely something is deployed into this VPC and you will have to remove those resources (could be EC2 instances, NAT Gateway, VPC endpoints, etc)*. 

</p>
</details>

### 1. Deploy Our Five VPCs
Run Cloudformation template 1.tgw-vpcs.yaml to deploy the VPCs.
<details>
<summary>HOW TO Deploy the VPCs</summary><p>

1. In the AWS Management Console change to the region you are working in. This is in the upper right hand drop down menu.

1. In the AWS Management Console choose **Services** then select **Cloudformation**.

1. In the main panel select **Create Stack** in the upper right hand corner.

1. Make sure **Template is ready** is selected from Prepare teplate options.

1. At the **Prerequisite - Prepare template** screen, for **template source** select **Upload a template file** and click **Choose file** from **Upload a Template file**. from your local files select **1.tgw-vpcs.yaml** and click **Open**.

1. Back at the **Prerequisite - Prepare template** screen, clcik **Next** in the lower right.

1. For the **Specify stack details** give the stack a name (be sure to record this, as you will need it later) and Select two Availability Zones (AZs) to deploy to. *We will be deploying all of the VPCs in the same AZs, but that is not required.  Click **Next.

1. For **Configuration stack options** we dont need to change anything, so just click **Next** in the bottom right.

1. Scroll down to the bottom of the **Review name_of_youstack** and check the **I acknowledge the AWS CloudFormation might create IAM resourcfes with custom names.**.

</p>
</details>

2. Run Cloudformation template 2.tgw-csr.yaml. Be sure to use the stack name used in step one for the 'Parent Stack'.
3. Create TGW attachment for VPN.

- use 169.254.10.0/30 and 169.254.11.0/30 for CIDR.
- use awsamazon for custom settings

4. using the two VPN tunnel endpoint address generated from step 3, run the bash script, createcsr.sh. Be sure to check the console VPC Service. Under Site-to-Site VPN tunnel detail find the addresses. Be sure to put the address that lines up with Inside IP CIDR address 169.254.10.0/30 for ip1.

```
./createcsr.sh ip1 ip2 outputfile
./createcsr.sh 1.1.1.1 2.2.2.2 mycsrconfig.txt
```

5. ssh to the CSR (Cloudformation output privdes a sample ssh command. be sure to verify your key and key path)
6. enter configuration mode

```
config t
```

7. paste in the text from the outputfile created in step 4.
8. Launch a Bind DNS server into the Datacenter (DC1 VPC) using the cloudformation template 3.tgw-dns.yaml

# To Do:

1. add AWS Resolver from geseib/awsresolver to this environment with Bind server in Datacenter Services VPC which connects to on Prem Bind Server.
2. add External routing through NAT Gateway/IGW in Datacenter services.
3. add Second prod VPC from another account instructions
4. add Shared VPC subnets in the non-prod VPC and give access from the account used in step 3.
5. add a AD in on prem DC connected to AD in Datacenter Services and SSO
