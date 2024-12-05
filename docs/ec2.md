# Amazon Elastic Compute Cloud (EC2)
It allows us to deploy virtual servers within the AWS environment.

## EC2 Components

- **Amazon Machine Images (AMI):** Pre configured EC2 instances that allow to quickly launch a new EC2 instance based on the config defined in the AMI. An AMI can help in auto scaling where multiple instances of the AMI can be quickly created.
- **Instance Types:** Defines the size of the instance based on different parameters - vCPUs, Architecture, Memory, Storage, Storage type, Network performance
- **Instance purchasing options:** On demand instances [can be launched anytime, can be used for as long as needed, flat rate based on instance types, typically used for short-term workloads], Spot Instances [on demand instances that are not taken currently, these are based on supply and demand], Reserved Instances [purchase a discounted on-demand instance for a set period], On-Demand Capacity Reservations [reserve capacity based on attribute]
- **Tenancy:** Shared Tenancy [by default, instances run on available hosts with selected resources], Dedicated Tenancy [runs on Dedicated Instances and Dedicated Hosts]
- **Uesr Data:** Allows to enter commands that will run during the first boot cycle of that instance. 
- **Storage Options:** Persistent Storage [EBS Volumes, also can be taken a snapshot of and store in an Amazon S3 Bucket], Ephemeral Storage [Created by EC2 instances using local storage]
- **Security:** Security group is an instance level firewall that protects traffic from ingress and egress requests. Restrict communications using source ports and protocols. Key-Pair Login: Private key and Public key system

## Virtual Private Cloud (VPC)
A VPC (Virtual Private Cloud) is a logically isolated section of a cloud provider’s network.

- **Isolation:** Resources inside a VPC are isolated.
- **Customization:** We can customize the IP address range, subnets, routing and security settings like network access control list (ACL) and security groups.
- **Security:** Firewalls, Private Subnets and VPNs
- **Network Config:** We can configure routing tables, internet gateways and even peering connections.

#### VPC Endpoint
A VPC Endpoint allows you to privately connect your VPC to support AWS services (like S3, DynamoDB, etc.) without requiring an internet connection, NAT gateway, or VPN connection.

#### CIDR (Classless Inter-Domain Routing)
It is a method for assigning IP addresses and IP routing that allows more efficient and flexible allocation of IP addresses. It provides a way to define network addresses with variable-length subnet masks, enabling networks of different sizes to be created.

CIDR notations are typically as - ```IP address/prefix length```

#### Subnet
A subnet (short for subnetwork) is a logical division of an IP network into smaller, manageable parts. It allows you to break a larger network into smaller segments to improve performance, security, and manageability.

#### Subnet Mask
A subnet mask is used to separate the network portion of an IP address from the host portion. In CIDR notation, the subnet mask is directly related to the prefix length. For ```192.168.0.0/24```, the subnet mask is written as ```255.255.255.0```.

#### IPV4 and IPV6 CIDR
IPv4 addresses are 32 bits long, written in four octets (e.g., 192.168.0.1), which gives around 4.3 billion unique addresses.

For example - ```192.168.0.0/24```, This represents all IP addresses from ```192.168.0.0``` to ```192.168.0.255```

IPv6 addresses are 128 bits long, written in hexadecimal and separated by colons (e.g., 2001:0db8:85a3::8a2e:0370:7334). IPv6 can support a vastly larger number of addresses—about 340 undecillion addresses.

#### Routing Table
In a routing table, the destination and target (sometimes called "next hop") columns are key components that determine how network traffic is routed. 

- **Destination Column:** The destination column in the routing table specifies the destination network or destination IP address that the packet is trying to reach.
- **Target Column:** The target (or "next hop") column specifies where the packet should go next to reach the destination.

The CIDR block must not be the same or larger than a destination CIDR range in a route in any of the VPC route tables.

#### Internet Gateway (IGW)
IGW is a component that helps VPCs connect with the internet or other external networks. It serves as the bridge between the private network (your VPC) and the public Internet.

- **Bidirectional Traffic:**  The Internet Gateway allows instances within the VPC to send and receive traffic from the Internet. 
- **Public and Private IP Handling:** For the Internet Gateway to work, instances in the VPC need either: Public IP address, Elastic IP address, NAT Gateways

#### Network Address Translation (NAT) Gateway
A NAT gateway enables instances within a private subnet in a VPC to access the internet for outbound traffic. The NAT gateway is a fully managed service provided by AWS, meaning AWS automatically takes care of scaling, fault tolerance, and maintenance. It can scale automatically to accommodate large amounts of traffic.

#### Virtual Private Gateway (VGW)
A Virtual Private Gateway (VGW) enables communication between an on-premises network and a cloud-based Virtual Private Cloud (VPC). 