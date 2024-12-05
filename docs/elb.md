# Load Balancers

## Elastic Load Balancer (ELB)
The main function of an ELB is to help manage and control the flow of inbound requests destined to a group of targets by distributing these requests evenly accross the targeted resource group. These targets could be a group of EC2s, lambda functions, a range of IP addresses or even Containers. 

#### Application Load Balancer
A load balancer serves as the single point of contact for clients. The load balancer distributes incoming application traffic across multiple targets, such as EC2 instances, in multiple Availability Zones. This increases the availability of your application. You add one or more listeners to your load balancer. 

You can set port for HTTP or HTTPS requests in your ALB. It operates at the request level. You also have Advanced routing, TLS termination and visibility features targeted at application architectures.

#### Network Load Balancer

- Ultra-high performance while maintaining very low latencies 
- Operates at the connection level, routing traffic to targets within your VPC
- Handles millions of requests per second

#### Route-Based Load Balancing (Path-Based Routing): 
Route-based load balancing (also called path-based routing) distributes traffic to different target groups based on the request URL path. 
The load balancer inspects the URL path of the incoming HTTP(S) request. Based on the path, it routes the traffic to a specific target group or service.

Example:

- Requests to example.com/frontend are routed to the frontend servers.
- Requests to example.com/api are routed to the API servers.

In AWS, you can configure path-based routing with an Application Load Balancer (ALB). You would define rules in the load balancer's listener based on URL paths (e.g., /images/*, /api/*), which then forwards the request to different target groups.

#### Host-Based Load Balancing
Host-based load balancing (also called host header-based routing) distributes traffic based on the hostname in the HTTP request. This allows you to direct traffic to different services or applications based on the domain or subdomain.

The load balancer inspects the Host header of the incoming HTTP(S) request.

Example:

A company has multiple domains or subdomains -

- www.example.com for the main website.
- api.example.com for their API.
- Requests to www.example.com go to the web servers.
- Requests to api.example.com go to the API servers.

#### Classic Load Balancer
- Used for applications that were build in the existing EC2 Classic Environment
- Operates at both the connection and request level

#### ELB Components
**Listeners:** The rules that you define for a listener determine how the load balancer routes requests to its registered targets. Each rule consists of a priority, one or more actions, and one or more conditions.

**Target Groups:** Each target group routes requests to one or more registered targets, such as EC2 instances, using the protocol and port number that you specify. You can register a target with multiple target groups. You can configure health checks on a per target group basis. 

**Rules:** Rules are associated to each listener that you have configured withing your ELB. They help to define how an incoming request gets routed to which target group.

**Health Checks:** A health check is performed against the resources defined within the target group. These health checks allow the ELB to contact each target using a specific protcol to receive a response.

**Internet-facing ELB:** The nodes of the ELB are accessible via the internt and so have a public DNS name that can be resolved to its Public IP address, in addition to an internal IP address.

**Internal ELB:** An internal ELB only has an internal IP address, this means that it can only serve requests that originate from within your VPC itself.

**ELB Nodes:** For each AZ selected an ELB node will be placed within that AZ.

## SSL Server Certificates
IF we set HTTPS as our listener, then it will allow us an encrypted communication channnel to be set up between clients initiating the request and your ALB. But to allow ALB to receive encrypted traffic over HTTPS it will need a server certificate and an associated security policy. 

SSL (Secure Sockets Layer) is a cryptography protocol, much like TLS (Transport Layer Security). Both SSL and TSL are used interchangebly when discussing certificates on your ALB.

The server certificate used by ALB is an X.509 certificate, which is a digital ID provisioned by AWS Certificate Manager (ACM). 

## Network Load Balancer
The network load balancer operates at layer 4 of the OSI model enabling you to balance requests purely based upon the TCP protocol. The NLB can process requests in TCP, TLS, and UDP. It is able to process millions of requests per second which makes it ideal for high performance applications.

If your application logic requires a static IP address, then NLB will need to be your choice of ELB.

## EC2 Auto Scaling
Auto scaling is the mechanism that allows you to increase or decrease your EC2 resources to meet the demand based off of custom defined metrics and thresholds.

## Auto Scaling in AWS

- Amazon EC2 Auto Scaling
- AWS Auto Scaling - Can automatically scale - Amazon ECS, DynamoDB, Amazon Aurora

## Components of EC2 Auto Scaling

- Create a Launch Configuration or Launch Template - These define how an Auto Scaling group builds new EC2 instances.
- Create an Auto Scaling group

#### Launch Configuration

- Which AMI to use
- What Instance type
- If you would like to use Spot Instances
- If and when Public IP Addresses should be used
- If any user data is on first boot
- What storage volume configuration should be used
- What security groups should be applied

#### Launch Template
A launch template is essentially a newer and more advanced version of the launch configuration. Being a template you can build a standard configuration allowing you to simplify how you launch instances for your auto scaling groups. This is the **preferred** method.

#### Auto Scaling Groups
The auto scaling group defines:

- The required capacity and other limitations of the group using scaling policies
- Where the group should scale resources, such as which AZ
