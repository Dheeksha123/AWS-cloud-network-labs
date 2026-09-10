Day 03 — ENI & IP Address Management
Objective

Design and deploy an AWS network environment to understand how Elastic Network Interfaces (ENIs) manage private IP addresses for EC2 instances.

This lab focuses on:

Elastic Network Interfaces
Primary private IPv4 addresses
Secondary private IPv4 addresses
Public IPv4 address mapping
Elastic IP addresses
EC2-to-ENI relationships
Private IP management
AWS VPC and subnet relationships
Verifying network configuration using the AWS Console and CLI

The main goal is to understand how AWS represents and manages network identity at the ENI level, rather than treating an EC2 instance as simply having "an IP address."

1. Scenario

An application server requires multiple private IP addresses.

For example, an organization may want one EC2 instance to use:

Primary IP       → Application traffic
Secondary IP 1   → Additional service
Secondary IP 2   → Additional service

The server also requires a stable public IP address for administrative access.

Instead of thinking:

EC2 → IP address

this lab demonstrates the more accurate AWS networking relationship:

EC2
 |
 ENI
 |
 +---- Primary Private IP
 |
 +---- Secondary Private IP
 |
 +---- Secondary Private IP
 |
 +---- Public IP / Elastic IP mapping
2. Architecture

                         INTERNET
                             |
                             |
                       Internet Gateway
                             |
                             |
                  VPC: 10.30.0.0/16
                             |
                    Public Subnet
                     10.30.1.0/24
                             |
                             |
                           EC2
                             |
                            ENI
                    +--------+--------+
                    |        |        |
                    |        |        |
                 Primary   Secondary Secondary
                 Private   Private   Private
                 IP        IP        IP
                 10.30.1.10 10.30.1.20 10.30.1.30
                    |
                    |
              Public IP / EIP
                    |
                    v
                 Internet
3. AWS Resource Design
Resource	Configuration
VPC	CloudNet-Day3-ENI
VPC CIDR	10.30.0.0/16
Subnet	Day3-Public-Subnet
Subnet CIDR	10.30.1.0/24
Internet Gateway	IGW-Day3
Route Table	RT-Day3-Public
EC2	Day3 EC2 instance
ENI	Primary network interface
Primary Private IP	10.30.1.10
Secondary Private IP	10.30.1.20
Additional Secondary IP	10.30.1.30
Public IP	AWS public IPv4 / EIP
Security Group	Day 3 security group

The secondary IP addresses shown above represent the lab design. The actual IPs and EIP values should be verified from the AWS Console/CLI output included with this project.

4. VPC Design

The VPC was created with:

VPC CIDR: 10.30.0.0/16

This provides the VPC with its private IPv4 address space.

10.30.0.0/16

contains:

65,536 IPv4 addresses

The subnet used for the EC2 instance is:

10.30.1.0/24
5. Subnet Design

The EC2 instance and its ENI are located in:

Day3-Public-Subnet
10.30.1.0/24

The subnet is connected to the VPC's routing infrastructure through its associated route table.

The route table contains:

10.30.0.0/16 → local
0.0.0.0/0    → Internet Gateway

Therefore, Internet-bound traffic can leave through the Internet Gateway.

6. Understanding the ENI

An Elastic Network Interface (ENI) is the network interface through which an EC2 instance communicates with the VPC.

A simplified relationship is:

VPC
 |
Subnet
 |
ENI
 |
Private IP addresses
 |
EC2

The ENI contains networking information such as:

Private IPv4 address
Secondary private IPv4 addresses
MAC address
Security Groups
Subnet association
VPC association

Therefore, the ENI is an important networking object in AWS.

7. Primary Private IP

The EC2 instance receives a primary private IPv4 address through its primary ENI.

Example:

Primary Private IP
10.30.1.10

This address belongs to the subnet:

10.30.1.0/24

The primary private IP is associated with the primary ENI and normally remains associated with that ENI for its lifetime.

8. Secondary Private IP

A secondary private IPv4 address was assigned to the ENI.

Example:

Primary:
10.30.1.10

Secondary:
10.30.1.20

This allows one ENI to have multiple private IPv4 addresses.

The relationship is:

EC2
 |
ENI
 |
 +--- 10.30.1.10  Primary
 |
 +--- 10.30.1.20  Secondary
 |
 +--- 10.30.1.30  Secondary

This is useful when multiple services or network identities need to exist on the same EC2 instance.

9. Why Secondary Private IPs Are Useful

Secondary private IPs can be useful for scenarios such as:

Hosting multiple services
Application migration
Failover scenarios
Service-specific IP addresses
Reassigning an IP between ENIs
Network appliance designs
Multi-service EC2 architectures

For example:

10.30.1.10 → Main application

10.30.1.20 → API service

10.30.1.30 → Internal service

The exact application mapping depends on the workload.

10. Public IPv4 Address vs Private IPv4 Address

A major concept demonstrated in this lab is that the public and private addresses are different identities.

Example:

EC2 / ENI

Private IP:
10.30.1.10

Public IP:
X.X.X.X

The EC2 instance communicates inside the VPC using its private address.

The public IPv4 address provides external Internet reachability when the appropriate routing and security configuration are present.

11. Elastic IP

An Elastic IP (EIP) is a persistent public IPv4 address allocated to the AWS account.

It can be associated with a network interface/private IP.

Conceptually:

Internet
   |
Public IPv4 / EIP
   |
Private IPv4
   |
ENI
   |
EC2

The EIP provides a stable public IPv4 identity compared with a dynamically assigned public IPv4 address.

12. Important AWS IP Mapping Concept

The Internet does not directly route to:

10.30.1.10

because this is a private IPv4 address.

Instead, the public address is mapped to the private address associated with the ENI.

Conceptually:

Public Internet
      |
      | Public IPv4 / EIP
      ↓
AWS networking
      |
      | Mapping
      ↓
Private IP
10.30.1.10
      |
      ↓
ENI
      |
      ↓
EC2

This demonstrates why public and private IP addresses should not be treated as the same address.

13. Route Table

The public subnet was associated with:

RT-Day3-Public

The important routes are:

Destination        Target
--------------------------------
10.30.0.0/16       local
0.0.0.0/0          Internet Gateway
Local route
10.30.0.0/16 → local

allows communication between resources within the VPC.

Default route
0.0.0.0/0 → IGW

sends traffic destined outside the VPC toward the Internet Gateway.

14. Security Group

The EC2 instance was associated with a Security Group controlling permitted traffic.

For administrative access, SSH was configured using:

Protocol: TCP
Port: 22
Source: Administrator public IP /32

Outbound traffic was allowed according to the lab security design.

Security Groups operate as traffic filters associated with the EC2 network interface.

15. Traffic Flow
Internal VPC Traffic

When an AWS resource communicates with the EC2 using its private IP:

Source
   |
   ↓
VPC Router
   |
   | 10.30.0.0/16 → local
   |
   ↓
ENI
   |
   ↓
10.30.1.10

The traffic remains inside the VPC.

16. Internet Traffic

For Internet-bound traffic:

EC2
 |
ENI
 |
Private IP
 |
Route Table
 |
0.0.0.0/0
 |
Internet Gateway
 |
Internet

If the instance has an appropriate public IPv4/EIP association, AWS provides the required public/private address mapping for Internet communication.

17. EC2 → ENI → IP Relationship

The most important relationship learned during this lab is:

                    VPC
                     |
                   Subnet
                     |
                    ENI
                     |
          +----------+----------+
          |          |          |
       Primary   Secondary   Secondary
       Private    Private     Private
          IP         IP          IP
          |          |           |
          +----------+-----------+
                     |
                    EC2

More precisely, the EC2 instance uses its network interface for VPC networking, and the ENI owns the private IP configuration.

18. Verification

The configuration was verified through the AWS Console and AWS CLI.

Important resources verified:

VPC
Subnet
Route Table
Internet Gateway
EC2
ENI
Primary Private IP
Secondary Private IP
Public IPv4 / EIP
Security Group
19. AWS CLI Verification
VPC
aws ec2 describe-vpcs

Used to verify:

VPC ID
CIDR block
VPC state

Detailed output:

View VPC CLI Output

Subnet
aws ec2 describe-subnets

Used to verify:

Subnet ID
CIDR block
Availability Zone
VPC association

Detailed output:

View Subnet CLI Output

ENI
aws ec2 describe-network-interfaces

Used to verify:

ENI ID
Private IP addresses
Subnet
VPC
Security Groups
Attachment information

Detailed output:

View ENI CLI Output

EC2
aws ec2 describe-instances

Used to verify:

Instance ID
Private IP
Public IP
Subnet
ENI
Security Group

Detailed output:

View EC2 CLI Output

20. Evidence

The following evidence was captured during the implementation:

AWS Console
VPC configuration
Subnet configuration
Route table
Internet Gateway
EC2 instance
Network Interface
Private IP configuration
Public IP / EIP configuration
CLI

CLI outputs were stored separately under:

cli-outputs/
Architecture

The complete network design is documented in:

architecture.png
21. Key Lessons Learned
1. EC2 does not directly "own" an IP

The networking relationship is:

EC2
 ↓
ENI
 ↓
Private IP
2. ENI is a fundamental AWS networking object

The ENI connects the EC2 instance to the VPC network.

It contains the networking configuration used by the instance.

3. One ENI can have multiple private IP addresses
ENI
 ├── Primary Private IP
 ├── Secondary Private IP
 └── Secondary Private IP
4. Private and public IPs have different roles
Private IP
→ VPC/internal identity

Public IP/EIP
→ Internet-facing identity
5. Route tables determine the path

The ENI has an IP, but the route table determines where packets destined for other networks should go.

6. Security Groups filter traffic

The Security Group determines whether permitted traffic can reach the ENI/instance.

It does not create the network path.

22. Troubleshooting Experience

During the lab, SSH connectivity was tested from the administrator's Windows machine.

An initial connection problem was investigated by checking:

SSH key location
Security Group source IP
Public IP
Route table
Internet Gateway
TCP port 22 connectivity

The administrator's local private address was identified as:

172.16.x.x

This address was not used as the AWS Security Group source because it was the private address assigned by the local Wi-Fi network.

The AWS Security Group instead required the administrator's current public IPv4 address.

After correcting the Security Group source, SSH connectivity to the Bastion/EC2 instance was successfully established.

23. Final Architecture Summary
                         INTERNET
                             |
                             |
                    Internet Gateway
                             |
                             |
                    VPC 10.30.0.0/16
                             |
                    Public Subnet
                     10.30.1.0/24
                             |
                             |
                           EC2
                             |
                            ENI
                             |
              +--------------+--------------+
              |              |              |
              ↓              ↓              ↓
         Primary IP     Secondary IP   Secondary IP
         10.30.1.10      10.30.1.20     10.30.1.30
                             |
                             |
                       Public IP / EIP
24. Final Result

The lab successfully demonstrated how AWS manages network identity using Elastic Network Interfaces and multiple private IPv4 addresses.

The key relationship established was:

VPC
 ↓
Subnet
 ↓
ENI
 ↓
Private IP addresses
 ↓
EC2

and for Internet connectivity:

EC2
 ↓
ENI
 ↓
Private IP
 ↓
Public IP / EIP mapping
 ↓
Internet Gateway
 ↓
Internet

This provides a foundation for more advanced AWS networking concepts such as:

Multiple ENIs
ENI reassignment
Network appliances
High-availability architectures
NAT Gateways
Load balancers
VPC routing
Network security
Advanced IP management
