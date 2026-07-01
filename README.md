These are my personal technical notes, maintained as part of my ongoing professional development in Google Cloud Platform. Based on Google Cloud documentation, Labs, training courses, community resources, and my own experimentation these notes reflect my current understanding and serve as a reference for my labs and projects. 
Please note that this documentation is a living, evolving work in progress and may contain incomplete and missing sections or stylistic inconsistencies. It is not intended as formal instructional material or a replacement for the official documentation.

For those interested I try to provide links and resources that helped me understand IT infrastructure and Cloud Computing, even though these notes explains cloud concepts in a simplified way and sometimes in greater detail, intermediate IT knowledge is required as a foundation to effectively follow these notes.

You do not need to be an expert on everything but you do need a fundamental understanding of the listed prerequisites.
Linux
Networking
Basic API understanding
Programming
Security
Basic Cloud Computing understanding
Problem solving
Documentation reading and writing


## Anything you create in the cloud belongs to one of two resources

Resource Hierarchy (Account-level resources) 
Domain - Identity, org association, universal policies, linked to Workspaces or Cloud Identity
Organization - Root of resource hierarchy, settings, permissions, all resource types
Folders - logical grouping of project and folders
Projects - groups Service-level resources
Tags - IAM Conditions and Organization Policy constraints

--------Resources above this line are managed in IAM & Admin > Manage Resources--------

Resources (Service-level resources) 
    Compute Instance VMs
    Cloud Storage buckets
    Cloud SQL database
    Anything that processes your workloads.

Metadata
    Labels - Filter resources with key/value pairs (great for tracking billing)

Architectural Hierarchies

Environment Oriented Hierarchy 
1 organization 1 folder per environment (Production, Staging, Development)
Pros
Simple implementation
Cons
Challenging to deploy service shared by multiple environments

Function Oriented Hierarchy 
1 organization per business function (Apps, Management, IT)
Pros
Increased flexibility (deployment of shared services)
Cons
Increased complexity

Granular-Access Oriented Hierarchy 
1 organization 1 folder per business unit (Retail, Risk, Finance, Commerce)
Pros
Most flexible and extensible
Cons
Management complexity in multiple areas (Structure, Roles, Permission, Network)

Some updates were made in early 2026 to the hierarchies but the concepts remain the same https://docs.cloud.google.com/architecture/landing-zones/decide-resource-hierarchy

These structures are not enforced in anyway and you are not locked into these when building your system, this is just an acknowledgment that these suggestions exist. When starting a project no matter the scale you can start with labels and build your structure from the bottom-up only creating structure when it makes sense to your specific project.

Projects and Folders
Projects are logical groupings of resources that consist of settings, permissions and metadata. A resource must always belong to a project. By default resources are project based and not accessible by other projects unless Shared VPC or VPC Network Peering is used. Each project ID is globally unique and permanently reserved so if you delete YourNextBigSaaS no one can ever use that name again. By default you can have 50 projects in your organization, increase to this amount can be requested from google. One billing account is associated per project.

Folders group multiple projects together that share same IAM permissions, commonly used to separate projects for different departments or environments (You can also use labels as pointed out earlier, note that labels are semantic and can't hold IAM permissions)

Organizational Policies
Apply constraints across your entire Resource Hierarchy.
>[!NOTE] Organizational Policies define what is allowed to happen in your resources regardless of who is performing the actions.

To create an Organizational policy you create a constraint and then apply an enforcement.
Enforcement defines

| Enforcement        | Description                                |
| ------------------ | ------------------------------------------ |
| Resource Type      | What do we constrain                       |
| Enforcement Method | When do we apply it (Create, Update)       |
| Condition          | The logical rules of access written in CEL |
| Action             | Allow or Deny                              |
Organizational Policy > Constraint > Enforcement

google provides a high level list of services that support custom constraints
https://docs.cloud.google.com/organization-policy/reference/custom-constraint-supported-services In it's current state creating custom organizational policies is a bit tricky and require investigating the APIs as the docs are not that clear on what can and cannot be applied to different resources.

Steps to setting custom organizational policies

1. Identify the Resource , such as Compute Engine VMs or Cloud Storage buckets.
2. Open the REST API documentation for your Resource and check it's JSON fields.
3. Convert relevant JSON fields into CEL notation, field = resource.field.
4. Use has(resource.field) to confirm a field exists before building logic around it.
5.  Write your CEL expressions using the fields you have verified with has().
6. Use Dry Run mode. This allows you to observe the impact on your environment.
7. Review Cloud Audit Logs for entries where dryRun: true.
8. Refine your CEL condition based on the audit results

Consult the policy library, It contains hundreds of pre-written, policies that can serve as a template or a starting point https://github.com/GoogleCloudPlatform/policy-library

Principle of least privilege
Create custom IAM roles that that provide the exact permissions that you or your users require. or at the very least use the pre-defined roles with fewer permissions. Avoid the primitive owner, editor and viewer roles and only grant access based on actual needs.

Cloud IAM
Identity Access Management who has specific permissions to access Google Cloud resources.
Principal Identifier
Individuals or systems accessing resources (users, groups, service accounts)
Roles
Define what specific actions can be performed by the member. (Viewer, Editor, Custom)
Policies
A collection of bindings that grant roles to principals on a resource.
Binding
Connects one or more principals to a role, optionally with a condition.
```yaml
bindings:
  - role: roles/storage.objectViewer
    members:
      - user:alice@example.com
    condition:
      title: "Limited Access"
      expression: resource.name.startsWith("projects/_/buckets/my-bucket")
```

Prefixes of common Principals (these tell gcp what type of Principal is being referenced)
user:
group:
serviceAccount:
allAuthenticatedUsers:
allUsers:

Account creation (use an existing email address)
This get's added at organization level
Add principals > Assign roles

Service Account Creation
Name: CoolService
Service account ID: cool-service
Based on the ID it get's an email address in the form of
cool-service@YourNewSaaS.iam.gserviceaccount.com
Grant permissions

A service account can grant principals permission to impersonate it. This is useful when you don't want to distribute service account keys or grant a developer direct access to a resource. By granting the Service Account Token Creator role, a developer can generate tokens that act as the service account and inherit its permissions.

You can add the permission through gcloud
```bash
gcloud iam service-accounts add-iam-policy-binding \
  cool-service@YourNewSaaS.iam.gserviceaccount.com \
  --member="user:alice@example.com" \
  --role="roles/iam.serviceAccountTokenCreator"
```

Removing access to the Service Account
```bash
gcloud iam service-accounts remove-iam-policy-binding cool-service@YourNewSaaS.iam.gserviceaccount.com \
    --member="user:alice@example.com" \
    --role="roles/iam.serviceAccountTokenCreator"
```

You can also replace the entire IAM policy attached to the service account with the contents of a policy file.
```bash
gcloud iam service-accounts set-iam-policy \
  cool-service@YourNewSaaS.iam.gserviceaccount.com\
  policy.yaml
```

>[!WARNING] Warning: `set-iam-policy` replaces the entire existing IAM policy. 
>Any bindings not included in `policy.yaml` will be removed. Consider version controlling policy files unless IAM is managed through Terraform.

Service Accounts can generate keys that allow external applications to authenticate as the Service Account. However, service account keys should generally be avoided in favor of service account impersonation or attached service accounts, as keys are long-lived credentials. Service Accounts are project-scoped resources.

Roles
Permission can not be assigned to Principals (users, service accounts) directly, instead permissions are grouped in roles and roles are assigned to Principals.

The 3 main Role types in Google Cloud IAM

| Role Type                       | Description                                                                                                                                | Examples                             |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------ |
| **Basic (legacy, discouraged)** | Simple broad roles that grant wide permissions across a project. Still used but not recommended by Google for fine-grained access control. | Owner, Editor, Viewer                |
| **Predefined**                  | Google-managed roles designed for specific services or job functions. More granular than basic roles.                                      | Compute Admin, Storage Object Viewer |
| **Custom**                      | User-defined roles where you choose exact permissions needed for a specific use case.                                                      | You define permissions yourself      |

Permissions
Specify the actions a user or system can take on a resource.
examples
storage.buckets.get    allows reading bucket information
storage.buckets.list 
storage.objects.create  allows uploading files with post or put
storage.objects.delete
storage.objects.get

Naming convention follows the format of  service.resource.verb

Conditions  
A logical expression that defines when access to a resource is granted.  
Access is only granted when the condition evaluates to true. 
If Condition evaluates to false access is denied,
If there is another binding that grants the same permissions access is granted.
Conditions are written in CEL (Common Expression Language).  
  
For example, viewing access could be granted for an intern that only works in july:  
```yaml  
bindings:  
- role: roles/storage.objectViewer  
members:  
- user:alice@example.com  
condition:  
title: "July access"  
expression: request.time >= timestamp("2026-07-01T00:00:00Z") &&  
request.time <= timestamp("2026-07-31T23:59:59Z")
```

To see what IAM roles you have run this in the cloud shell
```bash
gcloud aim roles list
```

Metadata
In IAM policies provides information for versioning, concurrency control and auditing. 
Etag
Concurrency control for consistent policy updates.
Basically a string that is used to track the state of the policy so that if you are making an update, meanwhile someone else updates the policy now your new update wont silently wipe out changes someone else just made.

Version
Defines schema version to ensure consistency and compatibility across updates.
Indicates which features are supported in the policy structure (such as conditions or binding types) and ensures backward compatibility when the policy format evolves.
Do not modify manually.

Audit Config
Defines which types of actions are recorded in Cloud Audit Logs for a service. It controls what activity is logged, such as read, write, or administrative operations.
It can also exempt certain identities from logging in some configurations, but its main purpose is controlling what gets written into logs.

3 types of audit logs
Admin Activity Logs - Record administrative actions (Policy updates, Resource creations)
Data Access Logs - Tracks access to resource data (read and write operations)
Policy Denied Logs - Captures failed attempts (policy based) to access resources


FreeCodeCamps Google Cloud Associate Cloud Engineer Course
https://www.youtube.com/watch?v=OlAmyf8_4O4
1:26:00
## The Cloud

## The Platform

## Building Apps

## Storage

## APIs

## Security
#remember mention about customer supplied keys to avoid the potential harm of cloud act

## Networking
Let's focus on exploring basic networking, public and private IP addresses, firewalls and routes, hybrid cloud network options and load balancing options as well as how to build Virtual Private Clouds (VPC).

#### VPC Network
The VPC is your own private network on the cloud, built on top of the cloud providers network, allowing many of the same security and access control rules as a physical network and deployment the of infrastructure as a service resources (containers, compute instances).

Virtual Private Clouds are global  and span through all available Google Cloud regions and have no IP address ranges. VPC contains subnets that spans across zones inside a region. If a virtual machine exists inside a separate VPC it will need a public IP address and communication must happen over the internet. By default networks do not communicate with other networks.

> [!INFO] About zones and regions
> A zone is a standalone data centre or a group of data centres europe-north1-a
> A Region is a specific geographic location like europe-north1 with at least 3 zones

#### Subnets
As a regional resource subnets provide IP Addresses to VPC that allow private communication between instances across zones and regions. A virtual machine in us-east1-a can be on the same subnet as a machine in us-east1-c, but to communicate with a virtual machine in europe-north1-a we need to do so by ensuring proper firewall access and utilizing the VPC's internal routing table that automatically connects all the subnets inside itself. Though hierarchy of the subnet is optional given there is no overlap in addresses, conforming to RFC1918 is not. Addresses must follow private IP ranges and CIDR notation.

| Class     | IP Address  | CIDR | Allowed | Not Allowed | Available IP | Managed by  |
| --------- | ----------- | ---- | :-----: | :---------: | ------------ | ----------- |
| Private A | 10.0.0.0    | /8   |    X    |             | 16 777 216   | You/Admin   |
| Private B | 172.16.0.0  | /12  |    X    |             | 1 048 576    | You/Admin   |
| Private C | 192.168.0.0 | /16  |    X    |             | 65 536       | You/Admin   |
| Public    | 8.8.8.8     | /32  |         |      X      | 1            | IANA/ISP    |
| Google    | 142.251.0.0 | /16  |         |      X      | 65 536       | IANA/Google |
| Reserved  | 240.0.0.0   | /4   |         |      X      | 268 435 456  | IETF        |

If you are interested how CIDR addresses work I highly recommend Cisco Networking Academy:
Subnetting Mastery: https://www.netacad.com/courses/subnetting-mastery?courseLang=en-US
NetworkChuck's - You Suck at Subnetting series https://www.youtube.com/watch?v=5WfiTHiU4x8
Test your understanding or find the correct addresses in the CIDR Calculator tool https://cidr.xyz/
Subnetting cheat sheet https://www.geeksforgeeks.org/computer-networks/subnet-cheat-sheet/

But for now it's enough to know that each of the sections of an IP address maps to an 8 bit representation called an octet. So your local IP could be 192.168.1.12 in the binary format it turns into 11000000.10101000.00000001.00001100 This is relevant because the CIDR, also called the Mask defines how many of those bits are used as the network part leaving the rest as host part of the address. So when u see an IP address with /32 it means all those 32 bits are frozen and used as the network address. The 8 bit version of the mask (not the IP) would then look like this 
11111111.11111111.11111111.11111111 To understand what is happening you need to know that each bit has double the value to the bit next to it, because of this every time you increase CIDR the number of available IP addresses is cut in half.

00000000 has a value of 0
00000001 has a value of 1
00000010 has a value of 2
00000100 has a value of 4
00001000 has a value of 8
00010000 has a value of 16
00100000 has a value of 32
01000000 has a value of 64
10000000 has a value of 128
11111111 has a value of 256

> [!NOTE]
> IPv6 follows the same logic, but instead of decimal format, we use hexadecimal format.
> You can think of it as a compressed way to represent binary where each hex digit represents 4 bits. This allows for 340 282 366 920 938 463 463 374 607 431 768 211 456 unique addresses compared to IPv4's 3 706 452 992.
##### Auto subnet mode and 
For experimenting, proof of concepts or extremely low complexity requirements on a small project we can use Auto subnet mode as it creates one subnet for each region with predefined IP ranges and default firewall rules, but in only expandable up to /16 or 65 536 addresses, the addresses them selves will most likely be more than enough but a huge flat network quickly grows to be an administrative nightmare prone to misconfiguration, on-premise integration issues and broadcast storms leading to unnecessary security risks, management overhead, timeouts and network performance issues.
##### Custom subnet mode
For production environment Custom subnet mode should be used as subnets and IP ranges are defined, it's expandable to any size in RFC 1918, no subnets or default firewall rules automatically in place, allowing the design being created from the ground up, requiring attention to the security and permission implementation, but gives you full control over the network. An auto mode network can be turned in the but custom mode can not be turned to auto mode afterwards.

##### Public and Private IP address basics
Internal IP addresses are assigned through Google's DHCP infrastructure. Although leases are periodically renewed, VM internal IPs normally remain unchanged unless the instance or network configuration changes.

External IP addresses can be ephemeral or reserved. They are assigned from a pool of IP addresses associated with the region. If you allocate a reserved IP address but don't attach it to a virtual machine, you will be billed for the IP address. Virtual machines are unaware of their public IP address. If you look at the operating system network configuration the virtual machine will only show the private IP address.

>[!CAUTION] External IPv4 address have a monthly cost of $3.65
>IPv6 do not incur cost for the address but wont work with all legacy systems.
>### Basic Cloud provider fees
>For the absolute cheapest with no discounts 
>(a 3 year commitment reduces these prices for ~50%)
>#### e2-micro with 2 shared vCPUs and 1 GiB of ram $6.73220235 a month
>Easy to throttle not suitable for heavy or continuous loads because of the shared core and limited memory footprint
>#### t2a-standard-1 dedicated vCPU 4 GiB of ram $28.105 a month
>using arm architecture it's suitable for containerized microservices, NGINX and Java as it's well optimized for arm, given that you can make do with the low memory amount.
>#### e2-standard-2 Dedicated vCPUs 8 GiB of ram $53.8576188 a month
>The standard workhorse
>#### e2-custom vCPU and Ram 1 Dedicated vCPU $18.4083735 1GiB ram $2.4667196
>#### n4a-highmem-1 Dedicated vCPU 8 GiB of ram $36.8942 a month
>When memory is more important the cpu additional memory $7.03004089 GiB per month
>
>These are some of the cheapest examples for running a virtual machine so if not strictly nessecery one should look into serverless containerized approach using cloud run with generous free tier and shut down on idle one can start with near 0 costs. 
>##### Some additional fees
>Load Balancer fees if one is created
>Forwarding fee $0.025/hour 
>Data processing fee $0.008 - $0.012/GB 
>Storage fee $0.02 GiB/month
>


#### Routes and Firewall Rules
When you create a VM-instance incoming connections from other VM's are allowed in the same VPC network, as well as SSH, scp, sftp, icmp and rdp. Other ingress traffic is blocked. When using a Custom Mode Subnet these rules do not exist and you need to configure the rules yourself.



To create a Custom Mode VPC Network and VM instance with multiple network interfaces on multiple subnets
https://www.skills.google/focuses/1231?parent=catalog

You can check the maximum number of allowed network interfaces per instance from the docs: https://docs.cloud.google.com/vpc/docs/multiple-interfaces-concepts#max-interfaces
The labs do not require this knowledge but this is good information to know when building your system.

#### Creating a Custom Mode VPC Network With Firewall Rules
In this Network we will allow SSH, ICMP and RDP ingress traffic. You can check you existing networks and create new networks at VPC network > VPC networks. By default you will find the default network created by google cloud per region. Alternatively you can run a command in cloud shell to see your networks.
```bash
gcloud compute networks list
```
To create a VPC network using the command line you can run the command, note that this VPC network is a container that holds routing, DNS and the logical connections.
```bash
gcloud compute networks create [networkName] --subnet-mode=custom
```
We will have to add our subnets inside the network.
```bash
export REGION=["your-region"]
export RANGE=["ip.range.using.CIDR/notation"]
gcloud compute networks subnets create [subnetName] --network=[networkName] --region=$REGION --range=$RANGE
```
To see your subnets run 
```bash
gcloud compute networks subnets list --sort-by=NETWORK
```


To create firewall rules go to VPC Network > Firewall
Give your firewall rule a descriptive name like
[networkName]-allow-icmp-ssh-rdp
specify the target network
specify what instances in the network you want to target
set source filters to IPv4
set the source range to 0.0.0.0/0 so it covers all networks
set protocols and ports check tcp type in 22, 3389
check Other type in icmp

for the command line 
```bash
gcloud compute firewall-rules create [networkName]-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=[networkName] --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
```
Confirm your firewall rules are set in place
```bash
gcloud compute firewall-rules list --sort-by=NETWORK
```
This configuration only allows for IPv4 to allow IPv6 you would add it as a filter and set the source range to ::/0 
```bash 
--source-ranges=0.0.0.0/0,::/0 
```

Creating a low cost virtual machine 
```bash
gcloud compute instances create [instanceName] --zone= --machine-type=e2-micro --subnet=[subnetName] --network-interface="network=[networkName],subnet=[subnetName]"
```


Configuring these firewall rules require specific permissions a Compute Network Admin is only able to list current firewall rules and a Compute Security Admin is able to create and delete them, as well as modify SSL certificates.

Roles can be added and removed in the IAM & Admin tab's edit access section where you can modify existing roles and permissions. Because Service Accounts can be attached to computing resources you can create [firewall rules](https://docs.cloud.google.com/vpc/docs/firewalls) that allow or deny traffic to and from instances based on the service account that you associate with each instance. An instance can only have one Service Account attached.

Relevant skill's lab's
https://www.skills.google/focuses/22772?parent=catalog



>[!NOTE] If you are practicing in skills lab and you SSH connection keeps failing you might need to disable oslogin
gcloud compute instances add-metadata [serverName] \ --zone=[zoneName] \ --metadata enable-oslogin=FALSE
>
>This makes the system to fall back on traditional SSH key authentication that google handles on the background.

#### Building  a Hybrid Cloud
Using cloud VPN we can connect on-premise network into a VPC network with a IPsec VPN tunnel allowing us to send encrypted traffic between VPN gateways using the public internet. This is suitable for low volume data connections.

>[!IMPORTANT] Client-to-Gateway (Point-to-Site) connections are not supported, you can not connect directly from your remote machine using a VPN client, you need to be in a network that is connected to the Cloud VPN tunnel.

A singular tunnel attached to the gateway has monthly cost of $36.50 and if the Cloud VPN tunnel connects to another Cloud VPN Gateway data transfer prices apply (0.08-0.12 per GB). Current prices can be checked at https://cloud.google.com/vpc/network-pricing?hl=fi

To connect to the on-premise network we need to configure Cloud VPN Gateway, on-premise VPN Gateway and 2 VPN tunnels as  each tunnel defines the connection from the perspective of it's gateway. Both gateways require an external IP address.

Dynamic routing with Cloud Router using Broader Gateway Protocol allows routes to be updated without re-configuring the tunnels. When a BGP session is established new subnets are automatically advertised and instantly operational.

##### Cloud Interconnect 
Offers 2 methods for extending your on-premise network to the cloud.
Dedicated provides a direct physical connection to Google Clouds network edge for transferring large amounts of data. 

Partner Interconnect uses service providers to enable the connection to their VPC networks when the organisation does not need +10Gbps connection or can not meet the googles network requirements for googles colocation facility.

Interconnect is an enterprize grade solution and the need for such solution is highly dependent on the organisation.

| Connection             | Provides                                                                  | Capacity                                                      | Requirements                        | Access Type           |
| ---------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------- | --------------------- |
| IPsec VPN Tunnels      | Encrypted tunnel to VPC networks through the public internet              | 1.5–3.0 Gbps per tunnel                                       | On-premises VPN gateway             | Internal IP addresses |
| Dedicated Interconnect | Dedicated, direct connection to VPC networks                              | 8 × 10 Gbps circuits, or 2 × 100 Gbps circuits per connection | Connection in a colocation facility | Internal IP addresses |
| Partner Interconnect   | Dedicated bandwidth connection to VPC networks through a service provider | 50 Mbps–10 Gbps per connection                                | Service provider                    | Internal IP addresses |
#### Direct Peering
Leverages Googles edge network locations reaching all googles services, does not provide SLA and you must satisfy the requirements [https://peering.google.com/#/options/peering](https://peering.google.com/) This is mostly for major Internet Service Providers, Content Delivery Networks or a massive organisation moving petabytes of data.

#### Carrier Peering
When you can't meet googles peer requirements you can opt to use a service provider, most likely you Internet Service Provider/Telecom Carrier. That gives you access to their Direct Peer network infrastructure.

| Connection      | Provides                                                      | Capacity                         | Requirements                    | Access Type         |
| --------------- | ------------------------------------------------------------- | -------------------------------- | ------------------------------- | ------------------- |
| Direct Peering  | Dedicated, direct connection to Google's network              | 10 Gbps per link                 | Connection in Google Cloud PoPs | Public IP addresses |
| Carrier Peering | Peering through a service provider to Google's public network | Varies based on partner offering | Service provider                | Public IP addresses |
#TODO fix: This section is a bit chaotic and blurs the lines of the actual inner workings of different types of load balancers, consider this as a high-level overview. In the mean time read the docs: https://docs.cloud.google.com/load-balancing/docs
### Load Balancing
To avoid exhausting a resource such as a Virtual Machine you distribute the requests across multiple instances.

Through a global HTTP(S) Load Balancer requests are routed to an global anycast IP address then a forwarding rule moves traffic to a target proxy that send request to the backend or use URL maps that send traffic to specific instances based on the URL. The backend service keeps track of the service capacity and instance health and sends traffic to a healthy backend with capacity closest to the user.

When using an SSL proxy the Load Balancer connects to the Virtua Machines with a second TCP connection. The Load Balancer can hold up to 10 SSL certificates. Using SSL proxy is optional but recommended by google.

When using TCP Proxy Load Balancing, the architecture is similar to SSL Proxy Load Balancing, but the load balancer does not terminate or provide encryption. Traffic is forwarded at Layer 4 and is not encrypted by the load balancer itself. By default, traffic between the load balancer and backend is sent in plaintext unless the application protocol enables encryption (TLS-enabled MySQL or PostgreSQL). This setup is typically used when TLS is handled end-to-end or when operating in trusted internal environments.

Network Load Balancing works by distributing traffic based on IP address, protocol and optional port ranges into a target pool that consists of virtual machines.

Internal Load Balancers work within the VPC and allow decoupling applications from detailed configuration settings. As an example, a user may access a web application through a global HTTP(S) Load Balancer, while the backend services communicate internally through an Internal Load Balancer. This allows backend systems such as application servers or databases to scale and be managed independently.

User1  \                             /  Web Tier --- Internal Tier --- Database Tier
User2    - Load Balancer -    Web Tier --- Internal Tier --- Database Tier
User3   /                            \  Web Tier --- Internal Tier --- Database Tier

#### How to create a HTTP Load Balancer
Because we are making a load balancer for unencrypted web traffic we allow traffic to port 80, for encrypted HTTPS traffic we would have opened port 443. We create a Firewall Rule where we choose our network, specify target tag as http-server, filter for IPv4 Ranges and set the source as 0.0.0.0/0 and allow TCP connections on port 80. Now any instance tagged as http-server will inherit this rule.

To decide which load balanced instances can receive new connections we can use Health Checks. The Health Check probes the instances through IP addresses so Firewall Rules need to allow those connections. So while creating a Firewall Rule at the source IPv4 range we need to add 130.211.0.0/22, 35.191.0.0/16 because we are using Application Load Balancing

| Load Balancer Type                          | Health Check IPv4 Source Ranges                                               |
| ------------------------------------------- | ----------------------------------------------------------------------------- |
| Global External Application Load Balancer   | 35.191.0.0/16,        130.211.0.0/22                                          |
| Regional External Application Load Balancer | 35.191.0.0/16,        130.211.0.0/22                                          |
| Internal Application Load Balancer          | 35.191.0.0/16,        130.211.0.0/22                                          |
| Proxy Network Load Balancer                 | 35.191.0.0/16,        130.211.0.0/22                                          |
| External Passthrough Network Load Balancer  | 35.191.0.0/16,        130.211.0.0/22, <br>209.85.152.0/22,    209.85.204.0/22 |
##### Configuring Instance Templates
Think of Instance Templates as blueprints for your web servers that define machine type, boot disk image, subnet, labels and other properties (almost everything you can configure when creating a virtual machine can be defined in this blueprint)

Compute Engine > Instance Templates > Create Instance Template
For example you could create a template that creates a E2 series Virtual Machine with our http-server network tags assigning our predefined firewall rules automatically.

| Property           | Value            |
|--------------------|------------------|
| Name               | Region 1-template |
| Location           | Global           |
| Series             | E2               |
| Machine Type       | e2-micro         |
| Network tags       | http-server      |
| Network            | default          |
| Subnetwork         | default (Region 1) |
In the management tab > +ADD ITEM
We can add a startup script to automatically installs Apache
KEY startup-script-url
VALUE gs://spls/gsp215/gcpnet/httplb/startup.sh
This updates the welcome page with client IP, hostname and server location.

#### Creating a Regional Mirror
Select you Instance Template and click + Create Similiar
Ensure that the information is correct and change the region.

#### Create Managed Instance Groups
This defines the rules we set for scalability. With the below table we are defining that at least 1 instance will always be on and when that instance hit's 80% of it's maximum CPU capacity we scale up to a new instance. For the first 45 seconds autoscaling ignores our new VM's metrics so the autoscaling doesn't confuse the setup process as running out of resources and spawning new VM's This is defined by the initialization period.

Compute Engine > Instance groups > Create Instance Group

| Property                                  | Value             |
| ----------------------------------------- | ----------------- |
| Name                                      | Region 1-mig      |
| Location                                  | Multiple zones    |
| Region                                    | Region 1          |
| Instance template                         | Region 1-template |
| Autoscaling - Minimum number of instances | 1                 |
| Autoscaling - Maximum number of instances | 2                 |
| Autoscaling signal type                   | CPU utilization   |
| Target CPU utilization                    | 80                |
| Initialization period                     | 45                |

If we had multiple Virtual Machines the autoscaling would calculate the average
resource use per instance group and use that average for scaling decisions

So let's say we allow the maximum number of instances to be 4 and current CPU usage is as follows

VM1 83% CPU utilisation
VM2 81% CPU utilisation
VM3 75% CPU utilisation

<details>
<summary>How many new instances would be created?</summary>
<pre>
    0 new instances would be created as the threshold of 80%
    has not yet been surpassed to trigger our scaling rules
    (83 + 81 + 75) / 3 = 79.67%
    
     Even if we temporarily spiked to a utilisation of +80%
     The scaling might not be triggered as the autoscaling 
     is aggregating the CPU utilisation over time
</pre>

</details>

##### Configuring the Application Load Balancer
Navigation Menu → Networking → Network Services → Load balancing → Create Load Balancer → Application Load Balancer (HTTP/HTTPS) → Public → Global → Global external Application Load Balancer → Configure
Name it as http-lb

##### Configure the frontend

Select Frontend Configuration

| Property   | Value     |
| ---------- | --------- |
| Protocol   | HTTP      |
| IP version | IPv4      |
| IP address | Ephemeral |
| Port       | 80        |

Add frontend IP and port

| Property    | Value        |
|-------------|-------------|
| Protocol    | HTTP        |
| IP version  | IPv6        |
| IP address  | Auto-allocate |
| Port        | 80          |
Application Load Balancing supports both IPv4 and IPv6 addresses for client traffic. Client IPv6 requests are terminated at the global load balancing layer, then proxied over IPv4 to your backends.

##### Configure the backend

Backend configuration > Backend services & backend buckets > Create backend service

| Property        | Value        |
|----------------|-------------|
| Name           | http-backend |
| Instance group | Region 1-mig |
| Port numbers   | 80           |
| Balancing mode | Rate         |
| Maximum RPS    | 50           |
| Capacity       | 100          |
This configuration means that the load balancer attempts to keep each instance of **`Region 1`-mig** at or below 50 requests per second (RPS).

Select Add a backend

| Property                    | Value        |
| --------------------------- | ------------ |
| Instance group              | Region 2-mig |
| Port numbers                | 80           |
| Balancing mode              | Utilization  |
| Maximum backend utilization | 80           |
| Capacity                    | 100          |
This configuration means that the load balancer attempts to keep each instance of **`Region 2`-mig** at or below 80% CPU utilization.

Select Create a health check

| Property  | Value         |
|-----------|--------------|
| Name      | http-health-check |
| Protocol  | TCP          |
| Port      | 80           |
Health check determines if a new connection can be made. The health check polls instances every 5 seconds waits up to 5 seconds and consider 2 successful attempt as healthy and 2 unsuccessful ones as unhealthy.

Enable Logging (Client IP, request paths, response status, latency, services)
Set sample ratio to 1 (logs 100% of requests)

Review and finalize, review backend and frontend
Create
Click name of load balancer
Note the IP addresses (LB_IP_v4 and LB_IP_v6) we need these for testing

#### Test the Application Load Balancer
Navigate to http://[LB_IP_v4]
Or if you have a local IPv6 address go to http://[LB_IP_v6]
It can take up to 5 minutes for the Load Balancer to be accessible.

##### Stress test with siege
Create a VM named siege-vm use a different region than your backend we just use Region 3 as a placeholder, set a zone and select series E2.

SSH to siege-VM (button in the UI)
Install siege
```bash
sudo apt-get update && sudo apt-get -y install siege
```
Set your Application Load Balancers IPv4 as environment variable
```bash
export LB_IP=[LB_IP_v4]
```
Simulate 150 users connecting at the same time for the duration of 2 minutes
```bash
siege -c 150 -t120s http://$LB_IP
```

Go to Navigation menu > Network Services > Load balancing > http-lb > Monitoring tab
Observe what happens in the next 2 minutes.

<details>
<summary>What should happen?</summary>
<pre>
    The traffic should be directed to your closest region
    but after the request per second (RPS) increases, traffic
    is directed to another backend in another region.
</pre>

</details>

##### Cloud Armor Denylist
We can use Cloud Armor to create a security policy and deny siege-vm from accessing our Application Load Balancer

>[!CAUTION] Cloud armor is an enterprize tool and costs are per policy and per rule
>1 policy costs $5 per month
>1 rule costs $1 per month

Compute Engine > VM Instances
Note the siege-vm External IP

View all products > Networking > Network Security > Cloud Armor > Create Policy

| Property                          | Value |
|----------------------------------|------|
| Name                             | denylist-siege |
| Default rule action              | Allow |
| Condition > Match                | Enter the SIEGE_IP (e.g. [SIEGE_IP] or [SIEGE_IP]/32) |
| Action                           | Deny |
| Response code                    | 403 (Forbidden) |
| Priority                         | 1000 |
After setting the name and Default rule action Click next step and Add a rule to fill the rest of the information of this table.

Done > Next step > Add target > Type > Backend service (external application load balancer >http-backend > Create policy

Alternatively, you could set the default rule to **Deny** and only  allow traffic from authorized users/IP addresses.

##### Verify the Security Policy

test in siege-vm SSH terminal curl http//:$LB_IP
This should result in 403 forbidden
test in your browser http://[LB_IP_v4]
This should be allowed

In the siege-vm SSH terminal run
```bash
siege -c 150 -t120s http//:$LB_IP
```
Go to Navigation menu > Network security > Cloud armor > denylist-siege > logs > View policy logs Clear all text in the Query preview, from All resources select Application Load Balancer > http-lb-forwarding-rule > http-lb > apply 
Run Query > expand log entry in Query results
Expand http requests you should see siege-vm's IP address
Expand jsonPayload > enforcedSEcurityPolicy
You should see configuredAction Deny denylist-siege in the json.

Relevant Skill's lab
This lab teaches you how to architect cross-regional failover using MIGs.
Implement edge-level security policies with Cloud Armor.
Validate traffic distribution and overflow mechanisms under high-concurrency stress tests.
https://www.skills.google/focuses/1232?parent=catalog

>[!TIP] Before taking this lab you should be comfortable navigating the UI of Google Cloud Platform or you might be stranded for time.
>When creating a region-mig you might see an error: Required subnetwork is only present on [Region 2]. The UI did not update the subnetwork region automatically, under location change the Region to reflect the region of the instance template.

#### How to Create an Internal Passthrough NetworkLoad Balancer
As the name suggests ILB manages and scales your private app infrastructure by distributing TCP and UDP based traffic across you VMs allowing traffic to be send only to the ILBs IP address.

A common deployment pattern for highly available applications is creating 2 managed instance groups (MIG) in different zones, this prevents possible data centre related downtime.

The idea is that clients connect to the Internal Load Balancer. Based on health checks, capacity, and load balancing policy defined in the Regional Backend Service, the ILB selects a healthy backend and forwards the traffic to that instance.

######  Configure HTTP and health check firewall rules
Make sure you have 2 subnets in the same region as ILB is a Regional service. If configuring with Custom Subnet Mode make sure SSH and ICMP is enabled. Create the Firewall Rule in VPC network app-allow-icmp and app-allow-ssh.

| Property | Value |
|----------|-------|
| Name | `app-allow-http` |
| Network | `my-internal-app` |
| Targets | Specified target tags |
| Target tags | `lb-backend` |
| Source filter | IPv4 Ranges |
| Source IPv4 ranges | `10.10.0.0/16` |
| Protocols and ports | Specified protocols and ports |
| Protocol | TCP |
| Port | 80 |
Create Firewall rule for health checks

| Property            | Value                             |
| ------------------- | --------------------------------- |
| Name                | `app-allow-health-check`          |
| Network             | `my-internal-app`                 |
| Targets             | Specified target tags             |
| Target tags         | `lb-backend`                      |
| Source filter       | IPv4 Ranges                       |
| Source IPv4 ranges  | `130.211.0.0/22`, `35.191.0.0/16` |
| Protocols and ports | Specified protocols and ports     |
| Protocol            | TCP                               |
gcloud command
```bash
gcloud compute health-checks create tcp my-ilb-health-check \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --region=asia-east1 \
    --port=80 \
&& \
gcloud compute backend-services create my-ilb \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --load-balancing-scheme=internal \
    --protocol=tcp \
    --region=asia-east1 \
    --health-checks=my-ilb-health-check \
    --health-checks-region=asia-east1 \
&& \
gcloud compute backend-services add-backend my-ilb \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --instance-group=instance-group-1 \
    --instance-group-zone=asia-east1-a \
    --region=asia-east1 \
&& \
gcloud compute backend-services add-backend my-ilb \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --instance-group=instance-group-2 \
    --instance-group-zone=asia-east1-b \
    --region=asia-east1 \
&& \
gcloud compute forwarding-rules create my-ilb-forwarding-rule \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --load-balancing-scheme=internal \
    --region=asia-east1 \
    --network=my-internal-app \
    --subnet=subnet-b \
    --address=10.10.30.5 \
    --ports=80 \
    --backend-service=my-ilb

```

##### Configure Instance Templates and create instance groups
MIGs use the Instance Templates as a blueprint to create identical instances for your backend. To configure an Instance Template

| Setting          | Value                               |
| ---------------- | ----------------------------------- |
| Navigate         | Compute Engine → Instance templates |
| Action           | Click Create instance template      |
| Name             | instance-template-1                 |
| Location         | Global                              |
| Series           | E2                                  |
| Machine type     | Shared-core → e2-micro              |
| Advanced options | Click to expand                     |
| Networking       | Open Networking section             |
| Network tags     | lb-backend                          |
Network tag applies the HTTP and Health Check rules you made previously.

In Network Interfaces dropdown set the following values
Network: [networkName]
Subnetwork: [subnetName]
External IPv4: None

Done

In Management > Automation > Startup script
Paste a script that installs apache and PHP, and crates a page that displays client IP and server metadata.

```bash
#!/bin/bash
apt-get update
apt-get install -y apache2 php libapache2-mod-php
cat <<'EOF' > /var/www/html/index.php
<h1>Internal Load Balancing Lab</h1>
<h2>Client IP</h2>
Your IP address : <?php echo $_SERVER['REMOTE_ADDR']; ?>
<h2>Hostname</h2>
Server Hostname: <?php echo gethostname(); ?>
<h2>Server Location</h2>
Region and Zone: <?php
  $ch = curl_init();
  curl_setopt($ch, CURLOPT_URL, "http://metadata.google.internal/computeMetadata/v1/instance/zone");
  curl_setopt($ch, CURLOPT_HTTPHEADER, array('Metadata-Flavor: Google'));
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
  $zone = curl_exec($ch);
  $parts = explode('/', $zone);
  echo end($parts);
?>
EOF
rm -f /var/www/html/index.html
systemctl restart apache2
```
You can easily modify this script to install nginx, nodejs, python3, golang-go or default-jdk (java) instead of Apache and PHP.

Copy the Instance Template, change it's name to instance-template-2 and change the subnetwork to your other subnet,

The terminal way for configuring instance templates for learning the structure/flags.
```bash
gcloud compute instance-templates create instance-template-2 \  
--project=qwiklabs-gcp-01-b99f03658522 \  
--machine-type=e2-micro \  
--region=asia-east1 \  
--network-interface=subnet=subnet-b,no-address,stack-type=IPV4_ONLY \  
--tags=lb-backend \  
--maintenance-policy=MIGRATE \  
--provisioning-model=STANDARD \  
--service-account=714278148667-compute@developer.gserviceaccount.com \  
--scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \  
--metadata=startup-script='#!/bin/bash  
apt-get update  
apt-get install -y apache2 php libapache2-mod-php  
  
cat <<'\''EOF'\'' > /var/www/html/index.php  
<h1>Internal Load Balancing Lab</h1>  
<h2>Client IP</h2>  
Your IP address : <?php echo $_SERVER["REMOTE_ADDR"]; ?>  
<h2>Hostname</h2>  
Server Hostname: <?php echo gethostname(); ?>  
<h2>Server Location</h2>  
Region and Zone: <?php  
$ch = curl_init();  
curl_setopt($ch, CURLOPT_URL, "http://metadata.google.internal/computeMetadata/v1/instance/zone");  
curl_setopt($ch, CURLOPT_HTTPHEADER, array("Metadata-Flavor: Google"));  
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);  
$zone = curl_exec($ch);  
$parts = explode("/", $zone);  
echo end($parts);  
?>  
EOF  
  
rm -f /var/www/html/index.html  
systemctl restart apache2' \  
--create-disk=boot=yes,auto-delete=yes,image=projects/debian-cloud/global/images/debian-13-trixie-v20260609,size=10,type=pd-balanced \  
--shielded-vtpm \  
--shielded-integrity-monitoring \  
--no-shielded-secure-boot \  
--reservation-affinity=any
```
Note that the script could be sourced from a separate file using
```bash
--metadata-from-file=startup-script=yourscriptname.sh
```

##### Crate the Managed Instance Groups
For Service Level Objectives without manual intervention.
Configure 1 MIG per subnet. Compute Engine > Instance Groups
Location single-zone
Remember to use a different Zones.
Configure Autoscaling as you see fit.

Here is a chained gcloud command for creating instance-group-2 for showcasing different flags that were applied during the creation.
```bash
gcloud beta compute instance-groups managed create instance-group-2 \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --base-instance-name=instance-group-2 \
    --template=projects/qwiklabs-gcp-01-b99f03658522/global/instanceTemplates/instance-template-2 \
    --size=1 \
    --zone=asia-east1-b \
    --default-action-on-vm-failure=repair \
    --action-on-vm-failed-health-check=default-action \
    --force-update-on-repair \
    --standby-policy-mode=manual \
    --list-managed-instances-results=paginated \
    --target-size-policy-mode=individual \
&& \
gcloud beta compute instance-groups managed set-autoscaling instance-group-2 \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --zone=asia-east1-b \
    --mode=on \
    --min-num-replicas=1 \
    --max-num-replicas=1 \
    --target-cpu-utilization=0.8 \
    --cpu-utilization-predictive-method=none \
    --cool-down-period=45 \
    --stabilization-period=600
```

##### Verify Backends
Verify that VM instances are being created in both subnets and create a utility VM to access the backends' HTTP sites directly. This step confirms individual backend functionality before introducing the load balancer, ensuring proper setup of your service tier.

Compuite Engine > VM Instances
Check that instance-groups 1 and 2 are in separate zones and internal IPs match your subnets CIDR blocks.

Create Instance > Machine Configuration

Instance Configuration

| Property | Value |
| :--- | :--- |
| **Name** | utility-vm |
| **Region** | REGION |
| **Zone** | ZONE |
| **Series** | E2 |
| **Machine Type** | e2-micro (1 shared vCPU) |

Networking Configuration

| Property                          | Value              |
| :-------------------------------- | :----------------- |
| **Network**                       | my-internal-app    |
| **Subnetwork**                    | subnet-a           |
| **Primary internal IPv4 address** | Ephemeral (Custom) |
| **Custom ephemeral IP address**   | 10.10.20.50        |

To create the virtual machine with gcloud you can run this command
```bash
gcloud compute instances create utility-vm \\
    --zone=ZONE \\
    --machine-type=e2-micro \\
    --network=[networkName] \\
    --subnet=[subnetName] \\
    --private-network-ip=10.10.20.50
```

Launch SSH for utility-vm
curl the internal IP addresses of the backends
The results should be something like
```html
<h1>Internal Balancing Lab</h1> (IP Hostname, Instance, Region, Zone)
```

Curl commands demonstrate that each VM instance lists the Client IP and its own name and location. This will be useful when verifying that the Internal Load Balancer sends traffic to both backends.

gcloud command for reference
```bash
gcloud compute instances create utility-vm \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --zone=asia-east1-a \
    --machine-type=e2-micro \
    --network-interface=network-tier=PREMIUM,private-network-ip=10.10.20.50,stack-type=IPV4_ONLY,subnet=subnet-a \
    --metadata=enable-osconfig=TRUE,enable-oslogin=true \
    --maintenance-policy=MIGRATE \
    --provisioning-model=STANDARD \
    --service-account=714278148667-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append \
    --create-disk=auto-delete=yes,boot=yes,device-name=utility-vm,image=projects/debian-cloud/global/images/debian-13-trixie-v20260609,mode=rw,size=10,type=pd-balanced \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --labels=goog-ops-agent-policy=v2-template-1-7-0,goog-ec-src=vm_add-gcloud \
    --reservation-affinity=any \
&& \
printf 'agentsRule:\n  packageState: installed\n  version: latest\ninstanceFilter:\n  inclusionLabels:\n  - labels:\n      goog-ops-agent-policy: v2-template-1-7-0\n' > config.yaml \
&& \
gcloud compute instances ops-agents policies create goog-ops-agent-v2-template-1-7-0-asia-east1-a \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --zone=asia-east1-a \
    --file=config.yaml \
&& \
gcloud compute resource-policies create snapshot-schedule default-schedule-1 \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --region=asia-east1 \
    --max-retention-days=14 \
    --on-source-disk-delete=keep-auto-snapshots \
    --daily-schedule \
    --start-time=06:00 \
&& \
gcloud compute disks add-resource-policies utility-vm \
    --project=qwiklabs-gcp-01-b99f03658522 \
    --zone=asia-east1-a \
    --resource-policies=projects/qwiklabs-gcp-01-b99f03658522/regions/asia-east1/resourcePolicies/default-schedule-1

```


#####  Configure the ILB
As a centralized access point for internal traffic ILB ensures intelligent traffic distribution based on health and instance capacity, this is crucial for achieving scalability and high availability.

Go to
Networking > Network Services > Load Balancing
Create load balancer

| Property                      | Value                               |
| :---------------------------- | :---------------------------------- |
| **Type of load balancer**     | Network Load Balancer (TCP/UDP/SSL) |
| **Proxy or passthrough**      | Passthrough load balancer           |
| **Public facing or internal** | Internal                            |
Configurations
name: my-ilb
Region: [yourRegion]
Network [networkName]

##### Configure the Regional Backend Service
This is the brain behind the ILB

Backend Configuration
Instance group: instance-group-1
Add backend
Instance group: instance-group-2
Create a health check
  Name:     my-ilb-health-check
  Protocol: TCP
  Port:        80

Create

Confirm that you see a blue checkmark in the cloud console.

##### Configure the frontend
The frontend is the exposed interface of the Internal Load Balancer (ILB). Assigning a static IP address provides a consistent and predictable endpoint for other internal services to connect to, simplifying service discovery and reducing the need for configuration changes within the VPC.

Simply put the frontend forwards traffic to the backend.
in Frontend Configuration set

| Property          | Value                                       |     |
| ----------------- | ------------------------------------------- | --- |
| Subnetwork        | subnet-b (would work on ether subnet)       |     |
| Internal IP       | Create IP address (under IP address option) |     |
| Name              | my-ilb-ip                                   |     |
| Static IP address | Let me choose                               |     |
| Custom IP address | 10.10.30.5                                  |     |
Reserve
port number: 80
Done

Review and finalize
Create (this can take a few minutes)

##### Test the Internal Load Balancer
SSH into utility-vm
curl the custom IP 10.10.30.5
The result should be similar to your previous curl showing the information of the backend chosen by ILB.

Keep repeating the curl command until you see a different IP or Zone in the terminal. The load balancer is now working correctly.

Docs
https://docs.cloud.google.com/load-balancing/docs
Relevant Skill's Lab
https://www.skills.google/focuses/1250?parent=catalog


## Monitoring
GCP offers a built in operations suite (formerly stackdrive) for monitoring, logging and diagnostics.
We go over them briefly as these tools are usually supplemented with 3rd party tools so implementations vary depending the organizations needs.

#### Cloud Monitoring
Used for automatically discovering and monitoring cloud resources. Offering a dashboard and visualization for identifying rising issues. It also provides reporting for anomalies, pattern detection, predictions about resource exhaustion giving insights to long terms trends that may require your attention.

#### Cloud Logging
Easy navigation between charts, traces, errors and logs created by other tools in this list.
Drill into root cause of system and application issues with centralized locations for all logs.
Logs can be analyzed in realtime and also exported to other systems.

#### Error Reporting
Identify and understand  application errors with realtime exception monitoring and alerting.
Shows top errors on dashboard. Watches your service constantly. Informs you when a new type of error arises.

#### Cloud Trace
Find bottlenecks by inspecting latency information of a request or aggregated latency of your whole application. Continuously gathers and analyzes data from applications to identify changes in performance. Results can be compared in Analysis Report section. If a significant shift in performance is detected Trace will alert you.

#### Cloud Debugger 
⚠️ Discontinued in 2023
Inspecting the state of your app running in production with out a performance dip. Take a snapshot of application state or inject a new logging statement, debugger can be used from Cloud Logging, Error Reporting, dashboards, integrated den environments and gcloud console.

#### Cloud Profiler
Analyzes how CPU and memory hungry functions are executed in the application. Realistic data from production environment instead of local testing. Run over all instances of the application providing a full picture. Application can even run on-prem.

## Data

#### Dataproc
Pricing $0.01 per virtual CPU per cluster per hour. Spark and Hadoop work out of the box, as does Pig and Hive so migrating existing projects and ETL pipelines is straightforward. Cloud Console, Cloud SDK and Dataproc API provide easy management.

Built in integrations with Cloud Storage, BigQuery, Cloud Bigtable, Cloud Logging & Monitoring can be used to create a complete dataplatform instead of just a Spark or Hadoop cluster. Terabytes worth of data are easily Extracted, Transformed and Loaded.
#### Spark & Hadoop Cluster

??
When a query is sent or a specific ETL workload needs to run the following happens:
Server is obtained and configured, OSS is installed, configured, optimized, debugged and data is processed. 
*if using tools that already existing in gcp the image is already optimized

On average it takes 90 seconds to set the infrastructure and start the data processing, this is possible because storage is handled separately allowing independent use of resources and avoiding I/O bottlenecks 
??

#### Hadoop and Spark jobs
Workflow template allows the configuration and execution of jobs

Workflow Templates: [https://cloud.google.com/dataproc/docs/concepts/workflows/overview](https://cloud.google.com/dataproc/docs/concepts/workflows/overview)
Dataproc job:
[https://cloud.google.com/dataproc/docs/concepts/jobs/life-of-a-job](https://cloud.google.com/dataproc/docs/concepts/jobs/life-of-a-job)

##### Creating a Managed Apache Spark Cluster
set the region
```bash
gcloud config set dataproc/region [youRegion]
```
If you encounter issues try to Disable and Enable Dataproc API
```bash
gcloud services disable dataproc.googleapis.com --force && gcloud services enable dataproc.googleapis.com
```
If you don't specify an account with storage bucket permissions for Managed Apache Spark to use, it will default to the Compute Engines service account, that does not have permissions to create staging and temp buckets. Your results are in these buckets.
```bash
PROJECT_ID=$(gcloud config get-value project) && gcloud config set project $PROJECT_ID

PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
```
Now that we have the needed information about the project let's add the Storage Admin and Managed Apache Spark Worker role to the Compute Engines service account
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com --role=roles/storage.admin

gcloud projects add-iam-policy-binding $PROJECT_ID --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com --role=roles/dataproc.worker
```
Enable Private Google Access to your subnet, this allows us to talk to googles APIs using googles infrastructure without an external IP, avoiding hops on the public internet.
```bash
gcloud compute networks subnets update default --region [youRegion] --enable-private-ip-google-access
```
Create a cluster with the resources applicable to your workload needs. This may take a moment.
```bash
gcloud dataproc clusters create [yourAwesomeClusterName] --worker-bootdsk-size 500 --worker-machine-type=e2-standard-4 --master-machine-type=e2-standard-4
``` 

##### Run a simple Spark Job
Here we calculate a rough value of pi 1000 times, the calculation built in to Spark. As you can see as long as --class matches the full path of  your `public static void main` class and --jars points to the Google Cloud Storage path of your JAR (JAVA and Scala things).
Everything after -- is argument to your software / code
```bash
gcloud dataproc jobs submit spark --cluster [yourAwesomeClusterName] --class org.apache.spark.examples.SparkPi --jars file://lib/spark/examples/jars/spark-examples.jar -- 1000
```
For PySpark 
```bash
gcloud dataproc jobs submit pyspark gs://[bucketName]/[pathName]/[coolScriptName] --cluster [youAwesomeCluster] -- --[yourArg1]=[yourValue] --[yourArg2]=[alsoSomeValue]
```

##### Updating the Cluster
Let's add more workers to your awesome cluster, note that this is not additive. Running this same command twice keeps the number of workers at 4. 
```bash
gcloud dataproc clusters update [yourAwesomeClusterName] --num-workers 4
```
Autoscaling policies are also available for clusters. Cluster should be ephemeral in nature but you can configure them to run 24/7. When you are done delete your cluster
```bash
gcloud dataproc clusters delete [yourAwesomeClusterName]
```


#### Data analysis with Dataproc
Dataproc can create clusters that scale for speed and mitigate any single point of failure. Since Dataproc supports Spark, Spark SQL, and PySpark, they could use the web interface, Cloud SDK, or the native Spark Shell via SSH.

Spark Shell:
[https://spark.apache.org/docs/latest/quick-start.html#interactive-analysis-with-](https://spark.apache.org/docs/latest/quick-start.html)[the-spark-shell](https://spark.apache.org/docs/latest/quick-start.html)
Spark Standalone Mode:
[https://spark.apache.org/docs/latest/spark-standalone.html](https://spark.apache.org/docs/latest/spark-standalone.html)

#### Machine Learning with Dataproc
Spark Machine learning library like MLib can be used to run algorithms on large datasets. MLib can be installed on any  Dataproc cluster and customized with init actions. Cluster creation is abstracted and integration with GCP can provide new features.

Spark Machine Learning Libraries (MLlib): 
[http://spark.apache.org/docs/latest/mllib-guide.html](http://spark.apache.org/docs/latest/mllib-guide.html).
## Machine learning





Next we will learn about Containers and Infrastructure as Code (IaC).





## Troubleshooting

SSH connection failures in lab due to OS Login
Fix:
```bash
gcloud compute instances add-metadata [serverName] \  --zone=[zoneName] \  --metadata enable-oslogin=FALSE
```

MIG creation error caused by subnet not existing in selected region
> [!WARNING]  Required subnetwork is only present in [Region X]
Fix: Ensure instance template region matches subnet region

Load Balancer not reachable
Fix: Wait until it is operational can take multiple minutes.