 Day 2 – AWS VPC Route Engineering

 1. Objective

The objective of Day 2 was to understand and implement routing inside an AWS VPC.

The lab focused on:

* VPC routing
* Route tables
* Local routes
* Public and private subnet routing
* Internet Gateway
* Route table associations
* Multi-AZ network design
* AWS CLI network inspection

The main goal was to understand **how AWS decides where a packet should go**, rather than simply memorizing AWS networking definitions.

---

# 2. Architecture

![Day 2 Architecture](architecture.png)

## Network Design

| Component           | Configuration   |
| ------------------- | --------------- |
| VPC                 | CloudNet-Day2   |
| VPC CIDR            | 10.20.0.0/16    |
| Availability Zones  | 2               |
| Public Subnet A     | 10.20.1.0/24    |
| Public Subnet B     | 10.20.2.0/24    |
| Private Subnet A    | 10.20.11.0/24   |
| Private Subnet B    | 10.20.12.0/24   |
| Internet Gateway    | IGW-Day2        |
| Public Route Table  | RT-Public-Day2  |
| Private Route Table | RT-Private-Day2 |

---

# 3. VPC Configuration

The VPC was created with:

```text
Name: CloudNet-Day2
CIDR: 10.20.0.0/16
```

The `/16` provides the overall address space for the VPC.

The subnet ranges were divided into separate `/24` networks.

```text
10.20.0.0/16
│
├── 10.20.1.0/24   Public-A
├── 10.20.2.0/24   Public-B
├── 10.20.11.0/24  Private-A
└── 10.20.12.0/24  Private-B
```

This provides a structured address plan that can be expanded later.

---

# 4. Availability Zone Design

The VPC was distributed across two Availability Zones.

```text
AWS Region
│
├── Availability Zone A
│   ├── Public-A
│   └── Private-A
│
└── Availability Zone B
    ├── Public-B
    └── Private-B
```

The purpose of using multiple Availability Zones is to avoid placing the entire network architecture inside a single Availability Zone.

This provides the foundation for future highly available architectures.

---

# 5. Route Tables

Two route tables were used.

## Public Route Table

```text
RT-Public-Day2
```

Associated with:

```text
Public-A
Public-B
```

The important routes are:

```text
Destination        Target
10.20.0.0/16       local
0.0.0.0/0          Internet Gateway
```

The `10.20.0.0/16 → local` route allows communication within the VPC.

The `0.0.0.0/0 → Internet Gateway` route provides a path toward the Internet for resources in the public subnets.

---

## Private Route Table

```text
RT-Private-Day2
```

Associated with:

```text
Private-A
Private-B
```

Current route:

```text
Destination        Target
10.20.0.0/16       local
```

There is currently no Internet Gateway route in the private route table.

Therefore, these subnets do not have direct Internet routing.

A NAT Gateway will be introduced in a later lab.

---

# 6. Understanding the Local Route

AWS automatically provides the VPC's local route.

```text
10.20.0.0/16 → local
```

This means traffic destined for an address inside the VPC can be routed internally.

For example:

```text
EC2-A
10.20.1.10
     │
     │
     ▼
VPC Routing
     │
     │ 10.20.0.0/16 → local
     ▼
EC2-B
10.20.2.10
```

The traffic does not need to go through:

* Internet Gateway
* NAT Gateway
* Internet

The destination is inside the VPC, so the traffic remains within the VPC routing domain.

---

# 7. Public Route Flow

For Internet-bound traffic from a resource in a public subnet:

```text
EC2
 │
 ▼
ENI
 │
 ▼
Subnet
 │
 ▼
Public Route Table
 │
 │ 0.0.0.0/0
 ▼
Internet Gateway
 │
 ▼
Internet
```

The important routing decision is:

```text
Destination: 8.8.8.8

Does 8.8.8.8 belong to 10.20.0.0/16?
        │
        └── No

Therefore:
0.0.0.0/0 → Internet Gateway
```

---

# 8. Private Route Flow

For traffic between resources inside the VPC:

```text
Private Resource
      │
      ▼
Private Subnet
      │
      ▼
Private Route Table
      │
      │ 10.20.0.0/16 → local
      ▼
Destination inside VPC
```

The private route table currently has no direct Internet Gateway route.

NAT Gateway functionality will be added in a future lab.

---

# 9. Internet Gateway

An Internet Gateway was created and attached to:

```text
CloudNet-Day2
```

The Internet Gateway itself does not automatically make every subnet public.

The subnet must use a route table containing:

```text
0.0.0.0/0 → Internet Gateway
```

Therefore:

```text
Internet Gateway
        +
Public Route Table
        +
Public Subnet
```

together establish the routing foundation for a public subnet.

---

# 10. Route Table Associations

The route tables were associated explicitly with the subnets.

```text
RT-Public-Day2
│
├── Public-A
└── Public-B
```

and:

```text
RT-Private-Day2
│
├── Private-A
└── Private-B
```

This creates a clear separation between public and private routing behavior.

---

# 11. AWS CLI Verification

The AWS CLI was used to inspect the network instead of relying only on the AWS Console.

## Verify AWS identity

```powershell
aws sts get-caller-identity
```

This confirms which AWS identity/account the CLI is currently using.

---

## Find the VPC

```powershell
aws ec2 describe-vpcs `
  --filters "Name=tag:Name,Values=CloudNet-Day2" `
  --query "Vpcs[*].[VpcId,CidrBlock,State]" `
  --output table
```

This was used to identify:

* VPC ID
* VPC CIDR
* VPC state

---

## Find the subnets

Replace the VPC ID with the actual VPC ID.

```powershell
aws ec2 describe-subnets `
  --filters "Name=vpc-id,Values=YOUR-VPC-ID" `
  --query "Subnets[*].[SubnetId,AvailabilityZone,CidrBlock,MapPublicIpOnLaunch,Tags[?Key=='Name']|[0].Value]" `
  --output table
```

This was used to map:

* Subnet ID
* Availability Zone
* CIDR
* Public IP assignment setting
* Subnet name

---

# 12. What I Verified

The AWS CLI was used to verify that the following network components existed:

```text
VPC
│
├── Public-A
├── Public-B
├── Private-A
└── Private-B
```

The next verification is the route-table configuration.

---

# 13. Key Networking Concepts Learned

### VPC

Provides the overall private network boundary.

```text
10.20.0.0/16
```

### Subnet

Divides the VPC address space into smaller networks.

### Route Table

Determines where traffic should be sent based on destination IP.

Example:

```text
10.20.0.0/16 → local
0.0.0.0/0    → Internet Gateway
```

### Local Route

Allows routing to destinations inside the VPC.

### Internet Gateway

Provides the VPC's connection to the Internet when appropriate routing and public addressing are configured.

### Route Table Association

Determines which route table a subnet uses.

---

# 14. Important Design Principle

A subnet is not public simply because it is named:

```text
Public-A
```

The important factor is its routing.

Conceptually:

```text
Subnet
   │
   ▼
Route Table
   │
   ├── 10.20.0.0/16 → local
   │
   └── 0.0.0.0/0 → Internet Gateway
```

That routing configuration is what gives the subnet a path toward the Internet.

---

# 15. Current Architecture

```text
                    INTERNET
                        │
                        ▼
              INTERNET GATEWAY
                        │
                        ▼
        ┌─────────────────────────────┐
        │       VPC 10.20.0.0/16      │
        │                             │
        │   Availability Zone A       │
        │   ┌─────────────────────┐   │
        │   │ Public-A             │   │
        │   │ 10.20.1.0/24        │   │
        │   └─────────────────────┘   │
        │                             │
        │   ┌─────────────────────┐   │
        │   │ Private-A            │   │
        │   │ 10.20.11.0/24       │   │
        │   └─────────────────────┘   │
        │                             │
        │   Availability Zone B       │
        │   ┌─────────────────────┐   │
        │   │ Public-B             │   │
        │   │ 10.20.2.0/24        │   │
        │   └─────────────────────┘   │
        │                             │
        │   ┌─────────────────────┐   │
        │   │ Private-B            │   │
        │   │ 10.20.12.0/24       │   │
        │   └─────────────────────┘   │
        │                             │
        └─────────────────────────────┘
```

---

# 16. Day 2 Outcome

At the end of Day 2, I built and documented the routing foundation of a multi-AZ AWS VPC.

I learned how:

* Subnets use route tables.
* Route tables contain destination-to-target routes.
* AWS automatically provides the VPC local route.
* Public routing requires a route toward an Internet Gateway.
* Private subnets can be isolated from direct Internet routing.
* Multiple subnets can share the same route table.
* AWS CLI can be used to inspect and map the network.
* Network architecture should be designed before deploying workloads.

---

# 17. Next Step

Day 3 will focus on:

* Elastic Network Interfaces (ENIs)
* Primary private IP addresses
* Secondary private IP addresses
* Public IPv4 addresses
* Elastic IP addresses
* How AWS maps IP addresses to ENIs
* How EC2 networking actually connects to the VPC
