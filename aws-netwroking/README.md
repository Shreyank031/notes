# AWS Networking

## Common terminologies:

### VPC

Creating a Virtual Private Cloud (VPC) in AWS is like setting up a private network in the cloud, allowing you to control your virtual networking environment. A VPC is a virtual network dedicated to your AWS account. It is logically isolated from other virtual networks in the AWS Cloud.

With VPC, you can:

- **Define a Virtual Network**: Customize the IP address range, create subnets, and configure route tables and network gateways.
- **Launch AWS Resources**: Deploy AWS resources, like Amazon EC2 instances, within your VPC.
- **Control Access**: Manage inbound and outbound access to and from individual subnets.


### Subnets and IP Addressing

Subnets are part of your VPC’s IP address range. They allow you to segment your VPC into smaller networks. Each subnet is associated with a particular availability zone (AZ) within an AWS region.

- Subnetting: A VPC spans all the Availability Zones in the region. When you create a VPC, you must specify a range of IPv4 addresses in the form of a CIDR block (e.g., 10.0.0.0/16). You can divide a VPC’s IP address range into one or more subnets.
- IP Addressing: AWS assigns a default IP address range to each subnet. You can assign a custom IP address range to each subnet within the CIDR block. Each subnet must reside entirely within one Availability Zone and cannot span zones.

### Internet Gateway

Internet Gateway (IGW) allows instances with public IPs to access the internet.

- Internet Gateway (IGW) is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet.

- Internet Gateway enables resources (like EC2 instances) in public subnets to connect to the internet. Similarly, resources on the internet can initiate a connection to resources in your subnet using the public.

- If a VPC does not have an Internet Gateway, then the resources in the VPC cannot be accessed from the Internet (unless the traffic flows via a Corporate Network and VPN/Direct Connect).

### VPC Route Table

A VPC Route Table contains a set of rules, known as routes, which determine where network traffic from your VPC is directed. Think of it as a map for your network traffic, guiding data packets to their destinations based on the rules you define.

**Components of a Route Table**

- Routes: These are the rules that determine the flow of traffic. Each route specifies a destination and a target (e.g., internet gateway, virtual private gateway, another VPC, etc.).

- Subnet Associations: Subnets in your VPC must be explicitly associated with a route table. If a subnet is not associated with any route table, it cannot send or receive traffic.

#### Types of Route Tables

- **Main Route Table**`: Every VPC comes with a main route table that can control the default routing for all subnets. If you create a subnet and don't associate it with any route table, it automatically associates with the main route table.

- **Custom Route Tables**: You can create custom route tables to define different routing rules for different subnets. This allows you to tailor the flow of traffic within your VPC according to your needs.

#### How to Create and Configure a Route Table

- **Create a Route Table**: In the AWS Management Console, under the VPC dashboard, you can easily create a new route table for your VPC.

- **Add Routes**: Specify the destination and target for your traffic. For example, to allow internet access, you might add a route where the destination is 0.0.0.0/0 (representing all IP addresses) and the target is an Internet Gateway.

- **Associate Subnets**: Decide which subnets should follow the rules defined in your route table and associate them accordingly.
