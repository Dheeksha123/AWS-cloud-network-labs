# Day 2 – AWS VPC Route Engineering

## Objective

The objective of Day 2 was to build and understand the routing foundation of an AWS VPC.

## AWS Architecture

* VPC: `CloudNet-Day2`
* VPC CIDR: `10.20.0.0/16`
* Availability Zones: 2

### Subnets

| Subnet    | CIDR            | Type    |
| --------- | --------------- | ------- |
| Public-A  | `10.20.1.0/24`  | Public  |
| Public-B  | `10.20.2.0/24`  | Public  |
| Private-A | `10.20.11.0/24` | Private |
| Private-B | `10.20.12.0/24` | Private |

### Route Tables

#### Public Route Table

```text
10.20.0.0/16 → local
0.0.0.0/0 → Internet Gateway
```

Associated with:

```text
Public-A
Public-B
```

#### Private Route Table

```text
10.20.0.0/16 → local
```

Associated with:

```text
Private-A
Private-B
```

## Internet Gateway

An Internet Gateway was attached to the VPC.

The public route table contains:

```text
0.0.0.0/0 → Internet Gateway
```

This provides the routing path toward the Internet for resources placed in the public subnets.

## Local Routing

AWS automatically provides:

```text
10.20.0.0/16 → local
```

This allows traffic destined for addresses inside the VPC to remain within the VPC routing domain.

For example:

```text
10.20.1.10
      ↓
VPC routing
      ↓
10.20.2.10
```

No Internet Gateway is required for communication between resources inside the VPC.

## AWS CLI

The AWS CLI was used to inspect the network.

### Verify AWS identity

```powershell
aws sts get-caller-identity
```

### Find the VPC

```powershell
aws ec2 describe-vpcs `
  --filters "Name=tag:Name,Values=CloudNet-Day2" `
  --query "Vpcs[*].[VpcId,CidrBlock,State]" `
  --output table
```

### Find subnets

```powershell
aws ec2 describe-subnets `
  --filters "Name=vpc-id,Values=YOUR-VPC-ID" `
  --query "Subnets[*].[SubnetId,AvailabilityZone,CidrBlock,MapPublicIpOnLaunch,Tags[?Key=='Name']|[0].Value]" `
  --output table
```

## What I Learned

* How AWS route tables work.
* What the `local` route means.
* How a subnet is associated with a route table.
* How public and private routing differ.
* How an Internet Gateway becomes reachable through a route table.
* How to inspect AWS networking using the AWS CLI.

## Next Step

Day 3 will focus on:

* ENIs
* Primary private IP addresses
* Secondary private IP addresses
* Public IP addresses
* Elastic IP addresses
* EC2 network interfaces
