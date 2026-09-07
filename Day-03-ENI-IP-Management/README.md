# Day 3 — ENI & IP Address Management

## AWS Cloud Network Engineering Lab

### Objective

The objective of this lab is to understand how networking is attached to an
Amazon EC2 instance through Elastic Network Interfaces (ENIs), and how
primary private IPv4 addresses, secondary private IPv4 addresses, public
IPv4 addresses, and Elastic IP addresses work together.

This lab focuses on understanding the actual relationships between:

- VPC
- Subnet
- Route Table
- Internet Gateway
- EC2
- Elastic Network Interface (ENI)
- Primary private IPv4 address
- Secondary private IPv4 addresses
- Public IPv4 address
- Elastic IP

The goal is not simply to create these resources, but to understand how
AWS networking is constructed around an EC2 instance.

---

# 1. Scenario

A cloud application server requires multiple IP addresses while remaining
within the same EC2 instance.

The server will have:

- One primary ENI
- One secondary ENI
- One primary private IPv4 address
- Two secondary private IPv4 addresses on the primary ENI
- One additional private IPv4 address on the secondary ENI
- A public IPv4 address
- An Elastic IP associated with the primary ENI

This represents a simplified version of scenarios where a server may need
multiple network identities, multiple services, or multiple interfaces.

---

# 2. Architecture

The architecture consists of a VPC containing a public subnet.

The subnet is associated with a route table containing:

    10.30.0.0/16 → local
    0.0.0.0/0    → Internet Gateway

The EC2 instance is connected to the subnet through its primary ENI.

A second ENI is also attached to the EC2 instance.

### Logical architecture

    Internet
        |
        |
    Internet Gateway
       IGW-Day3
        |
        |
    +--------------------------------------+
    | VPC                                  |
    | CloudNet-Day3-ENI                    |
    | 10.30.0.0/16                         |
    |                                      |
    |  +--------------------------------+  |
    |  | Public Subnet                  |  |
    |  | Day3-Public-Subnet             |  |
    |  | 10.30.1.0/24                   |  |
    |  |                                |  |
    |  |      Route Table               |  |
    |  |      RT-Day3-Public            |  |
    |  |                                |  |
    |  |      EC2 Instance              |  |
    |  |      Day3-ENI-Server            |  |
    |  |             |                  |  |
    |  |       +-----+------+           |  |
    |  |       |            |           |  |
    |  |     ENI-1        ENI-2         |  |
    |  |       |            |           |  |
    |  |    3 IPs          1 IP         |  |
    |  |                                |  |
    |  +--------------------------------+  |
    +--------------------------------------+

---

# 3. Environment

## AWS Region

Mumbai Region:

    ap-south-1

## Availability Zone

    ap-south-1a

## VPC

Name:

    CloudNet-Day3-ENI

CIDR:

    10.30.0.0/16

The VPC provides the overall private IPv4 address space for this lab.

---

# 4. Subnet

Name:

    Day3-Public-Subnet

CIDR:

    10.30.1.0/24

Availability Zone:

    ap-south-1a

The subnet is carved from the VPC CIDR.

Relationship:

    VPC
      |
      └── 10.30.0.0/16
              |
              └── Subnet
                    |
                    └── 10.30.1.0/24

The subnet provides the address space from which the ENIs obtain their
private IPv4 addresses.

---

# 5. Internet Gateway

Name:

    IGW-Day3

The Internet Gateway is attached to:

    CloudNet-Day3-ENI

The Internet Gateway provides the VPC with a path to and from the Internet
when the appropriate routing and public addressing are configured.

However, simply attaching an Internet Gateway does not automatically make
a subnet public.

The subnet must have a route pointing Internet-bound traffic toward the
Internet Gateway.

---

# 6. Route Table

Name:

    RT-Day3-Public

The route table is associated with:

    Day3-Public-Subnet

Routes:

    Destination       Target
    --------------------------------
    10.30.0.0/16      local
    0.0.0.0/0         IGW-Day3

## Local Route

The local route:

    10.30.0.0/16 → local

allows resources inside the VPC to communicate with other destinations
within the VPC address space.

## Internet Route

The route:

    0.0.0.0/0 → IGW-Day3

means traffic whose destination does not match the VPC's local CIDR can
use the Internet Gateway.

---

# 7. EC2 Instance

Name:

    Day3-ENI-Server

Instance type:

    t3.micro

The EC2 instance is deployed into:

    Day3-Public-Subnet

The EC2 instance does not directly obtain networking independently of the
VPC architecture.

Its network connectivity is provided through one or more Elastic Network
Interfaces.

---

# 8. Elastic Network Interface — ENI

An Elastic Network Interface is the networking interface through which the
EC2 instance connects to the VPC.

The ENI contains networking attributes such as:

- Private IPv4 addresses
- Public IPv4 association
- Elastic IP association
- Security groups
- MAC address
- Subnet association
- VPC association
- Source/destination check

The important relationship is:

    EC2
      |
      └── ENI
            |
            ├── Private IP
            ├── Security Groups
            ├── MAC Address
            └── Subnet

Therefore, the ENI is the important networking object connecting the EC2
instance to the VPC.

---

# 9. Primary ENI

The EC2 instance's primary network interface is ENI-1.

Example ENI:

    eni-083096992511724b5

Status:

    In-use

Subnet:

    Day3-Public-Subnet

Availability Zone:

    ap-south-1a

---

# 10. Primary Private IPv4 Address

The primary private IPv4 address observed on ENI-1 is:

    10.30.1.187

This address belongs to the subnet:

    10.30.1.0/24

The address is used as the primary private identity of the ENI inside
the VPC.

Relationship:

    VPC
      |
      └── Subnet
            |
            └── ENI
                  |
                  └── Primary Private IP
                        10.30.1.187

---

# 11. Secondary Private IPv4 Addresses

Two secondary private IPv4 addresses are assigned to the primary ENI.

They are:

    10.30.1.20
    10.30.1.30

Therefore, ENI-1 contains:

    ENI-1
      |
      ├── Primary
      |     10.30.1.187
      |
      ├── Secondary
      |     10.30.1.20
      |
      └── Secondary
            10.30.1.30

This demonstrates that one ENI can have multiple private IPv4 addresses.

These addresses can be useful for scenarios such as:

- Multiple applications
- Multiple services
- IP-based service identification
- Failover addresses
- Application migration
- Network appliances
- Multi-address workloads

---

# 12. Secondary ENI

A second Elastic Network Interface was created:

    Day3-Secondary-ENI

The secondary ENI has a private IPv4 address:

    10.30.1.40

The ENI is attached to the same EC2 instance.

The resulting architecture is:

    EC2
     |
     +-------------------+
     |                   |
    ENI-1              ENI-2
     |                   |
     +-- 10.30.1.187     +-- 10.30.1.40
     |
     +-- 10.30.1.20
     |
     +-- 10.30.1.30

This demonstrates that an EC2 instance can have multiple ENIs, subject to
the networking limits of the selected EC2 instance type.

---

# 13. Public IPv4 Address

The primary ENI initially received the public IPv4 address:

    52.66.199.25

This public IPv4 address provides public-facing IPv4 addressing for the
instance.

The public address should not be confused with the private IPv4 address.

The private address exists within the VPC:

    10.30.1.187

while the public address is used for Internet-facing communication:

    52.66.199.25

Conceptually:

    Internet
       |
       | Public IPv4
       |
       ↓
    Private IPv4
       |
       ↓
      ENI
       |
       ↓
      EC2

---

# 14. Elastic IP

An Elastic IP address was allocated and associated with the primary ENI.

An Elastic IP provides a persistent public IPv4 address that can be
associated with an AWS resource.

The important concept is that the Elastic IP is associated with a private
IPv4 address on the ENI.

Conceptually:

    Elastic IP
        |
        ↓
    Private IPv4
        |
        ↓
       ENI
        |
        ↓
       EC2

This is different from simply thinking:

    EIP → EC2

The ENI and its private IP are the important networking layer between
the public address and the EC2 instance.

---

# 15. Complete IP Architecture

The resulting IP structure is:

    EC2: Day3-ENI-Server
    |
    +── ENI-1
    |     |
    |     ├── Primary Private IP
    |     |      10.30.1.187
    |     |
    |     ├── Secondary Private IP
    |     |      10.30.1.20
    |     |
    |     ├── Secondary Private IP
    |     |      10.30.1.30
    |     |
    |     └── Public IPv4
    |            52.66.199.25
    |
    └── ENI-2
          |
          └── Private IP
                 10.30.1.40

---

# 16. Key Networking Relationships

The most important relationships learned in this lab are:

    VPC
      ↓
    Subnet
      ↓
    ENI
      ↓
    Private IP
      ↓
    EC2

For Internet-facing communication:

    Internet
      ↓
    Internet Gateway
      ↓
    Route Table
      ↓
    Subnet
      ↓
    ENI
      ↓
    Private IP
      ↓
    EC2

For Elastic IP addressing:

    Elastic IP
      ↓
    Private IP
      ↓
    ENI
      ↓
    EC2

---

# 17. Packet Flow — Outbound Traffic

Suppose the EC2 instance sends traffic to:

    8.8.8.8

The conceptual flow is:

    EC2
      ↓
    ENI
      ↓
    Private IPv4
      ↓
    Subnet
      ↓
    Route Table
      ↓
    0.0.0.0/0
      ↓
    Internet Gateway
      ↓
    Internet

The route table determines the next destination based on the packet's
destination IP.

The local route handles VPC traffic, while the default route handles
Internet-bound traffic.

---

# 18. Why the ENI Is Important

The ENI is one of the most important concepts in AWS EC2 networking.

Instead of thinking:

    EC2 = IP address

the correct mental model is:

    EC2
      |
      └── ENI
            |
            ├── Private IP
            ├── Secondary IPs
            ├── Security Groups
            ├── MAC Address
            └── Network connectivity

This becomes particularly important when working with:

- Multiple network interfaces
- Network appliances
- Firewalls
- Routing appliances
- High-availability architectures
- IP failover
- Secondary IP addresses
- Advanced EC2 networking

---

# 19. Security Group

Security group:

    SG-Day3-ENI

SSH access was restricted to:

    My IP

The security group controls traffic allowed to the ENI.

Therefore, another important relationship is:

    Security Group
          |
          ↓
         ENI
          |
          ↓
         EC2

---

# 20. AWS CLI Verification

The configuration was verified using the AWS CLI.

## VPC

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=CloudNet-Day3-ENI" \
  --query 'Vpcs[*].[VpcId,CidrBlock,State]' \
  --output table
