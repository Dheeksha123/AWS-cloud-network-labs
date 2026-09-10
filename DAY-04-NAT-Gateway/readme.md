Day 04 — NAT Gateway, Private Subnets & High Availability
Project Summary

Designed and deployed a highly available AWS VPC architecture across two Availability Zones using public/private subnet segmentation, an Internet Gateway, zonal NAT Gateways, Elastic IPs, route tables, Bastion-based administration, and Security Groups.

Verified private-subnet outbound Internet connectivity through NAT and internal cross-subnet SSH connectivity using VPC local routing.

Skills Demonstrated
AWS VPC architecture
IPv4 subnet design
Public/private subnet segmentation
Route table engineering
Internet Gateway configuration
NAT Gateway deployment
Elastic IP management
Availability Zone redundancy
Security Group design
Bastion Host administration
SSH agent forwarding
VPC local routing
NAT/SNAT traffic-flow analysis
AWS CLI verification
High Availability design
Private subnet administration
Network troubleshooting
1. Project Objective

The objective of this lab was to design and deploy a production-style AWS VPC containing:

Two Availability Zones
Two public subnets
Two private subnets
Internet Gateway
Two NAT Gateways
Two Elastic IP addresses
Dedicated route tables
Bastion Host
Private EC2 instances
Security Groups
Private-to-Internet outbound connectivity
Cross-AZ internal connectivity

The main networking requirement was:

Private EC2 instances must be able to access the Internet without having public IP addresses.

The architecture was also designed for Availability Zone-level NAT redundancy, so that each private subnet uses a NAT Gateway in its own Availability Zone.

2. Architecture

                              INTERNET
                                  |
                                  |
                         +----------------+
                         |      IGW       |
                         | Internet       |
                         | Gateway        |
                         +----------------+
                           /            \
                          /              \
                         /                \
              Public-A Subnet          Public-B Subnet
              10.40.1.0/24             10.40.2.0/24
                    |                       |
                    |                       |
                 NAT-A                   NAT-B
                 EIP-A                   EIP-B
                    |                       |
                    |                       |
              Private-A Subnet         Private-B Subnet
              10.40.11.0/24            10.40.12.0/24
                    |                       |
                    |                       |
                 EC2-A                    EC2-B
              No Public IP             No Public IP

Administrative access:

My Laptop
    |
    | SSH
    |
    v
Bastion Host
Public Subnet
    |
    | SSH through VPC
    |
    +----------------------+
    |                      |
    v                      v
Private EC2-A          Private EC2-B
10.40.11.x             10.40.12.x
3. Network Design
Component	Configuration
VPC	CloudNet-Day4-NAT
VPC CIDR	10.40.0.0/16
Region	ap-south-1
AZ-A	ap-south-1a
AZ-B	ap-south-1b
Public-A	10.40.1.0/24
Public-B	10.40.2.0/24
Private-A	10.40.11.0/24
Private-B	10.40.12.0/24
Internet Gateway	IGW-Day4
NAT Gateway A	NAT-A
NAT Gateway B	NAT-B
Public Route Table	RT-Day4-Public
Private Route Table A	RT-Day4-Private-A
Private Route Table B	RT-Day4-Private-B
Bastion	Day4-Bastion
Private instances	Day4-Private-A, Day4-Private-B
4. VPC Design

The VPC uses:

10.40.0.0/16

This provides the overall private IPv4 address space for the lab.

The VPC is divided into four subnets:

10.40.0.0/16
     |
     +-- Public-A   10.40.1.0/24
     |
     +-- Public-B   10.40.2.0/24
     |
     +-- Private-A  10.40.11.0/24
     |
     +-- Private-B  10.40.12.0/24

The subnets are distributed across two Availability Zones.

5. Public and Private Subnet Design

A key concept demonstrated in this lab is:

A subnet is not called public because its CIDR address is a "public IP range."

For example:

10.40.1.0/24

is an RFC1918 private IPv4 address range.

Yet Public-A is still a public subnet.

Why?

Because its route table contains:

0.0.0.0/0 → Internet Gateway

Therefore:

Public subnet
    +
Route to IGW
    =
Public subnet

Similarly:

Private subnet
    +
No direct route to IGW
    =
Private subnet

The terms public subnet and private subnet describe routing behavior, not whether the subnet CIDR itself is a public IP range.

6. Route Table Architecture
Public Route Table

The public subnets use:

RT-Day4-Public

Routes:

Destination	Target
10.40.0.0/16	local
0.0.0.0/0	Internet Gateway

Therefore:

Public EC2
    |
    v
Public Route Table
    |
    +---- 10.40.0.0/16 → local
    |
    +---- 0.0.0.0/0 → IGW
7. Private Route Table A

Private-A uses:

RT-Day4-Private-A

Routes:

Destination	Target
10.40.0.0/16	local
0.0.0.0/0	NAT-A

Therefore:

Private EC2-A
      |
      v
RT-Day4-Private-A
      |
      +---- 10.40.0.0/16 → local
      |
      +---- 0.0.0.0/0 → NAT-A
8. Private Route Table B

Private-B uses:

RT-Day4-Private-B

Routes:

Destination	Target
10.40.0.0/16	local
0.0.0.0/0	NAT-B

Therefore:

Private EC2-B
      |
      v
RT-Day4-Private-B
      |
      +---- 10.40.0.0/16 → local
      |
      +---- 0.0.0.0/0 → NAT-B
9. Why NAT Gateway Is Required

Private EC2 instances do not have public IPv4 addresses.

For example:

EC2-A
Private IP: 10.40.11.x
Public IP: NONE

The instance can communicate inside the VPC using its private address.

However, if it wants to access:

Internet

the Internet cannot directly route traffic back to:

10.40.11.x

because this is an RFC1918 private address.

NAT Gateway solves this problem.

10. NAT Gateway Architecture

The NAT Gateway is deployed inside a public subnet.

For example:

Private-A
10.40.11.0/24
       |
       | 0.0.0.0/0
       v
     NAT-A
       |
       | EIP
       v
 Public-A
10.40.1.0/24
       |
       v
      IGW
       |
       v
   INTERNET

NAT-A has an Elastic IP associated with it.

This allows outbound Internet traffic from private instances to appear to the Internet as coming from the NAT Gateway's public IP.

11. Why NAT Gateway Needs an Elastic IP

This was an important part of the lab.

The private EC2 has:

10.40.11.x

That address cannot be used as an Internet-facing source address.

Therefore the NAT Gateway performs source NAT.

Conceptually:

Before NAT:

Source = 10.40.11.25
Destination = Internet server

NAT Gateway translates the source:

Source = NAT Elastic IP
Destination = Internet server

For example:

10.40.11.25
      |
      v
    NAT-A
      |
      v
3.x.x.x  ← NAT EIP
      |
      v
  INTERNET

The Internet therefore sees the NAT Gateway's public Elastic IP rather than the private EC2 address.

12. NAT Traffic Flow

The complete outbound flow is:

Private EC2-A
10.40.11.x
     |
     v
Private Route Table A
     |
     | 0.0.0.0/0 → NAT-A
     v
NAT-A
     |
     | Source translated to EIP-A
     v
Public-A
     |
     v
Internet Gateway
     |
     v
INTERNET

The return traffic follows the NAT state back to the private instance.

Conceptually:

INTERNET
   |
   v
IGW
   |
   v
NAT-A
   |
   | NAT translation
   v
10.40.11.x
   |
   v
Private EC2-A
13. Why Two NAT Gateways?

A single NAT Gateway could technically provide Internet access to both private subnets.

For example:

Private-A ----\
               \
                NAT-A
               /
Private-B ----/

However, this introduces a dependency on one Availability Zone.

Instead, this lab uses:

Private-A → NAT-A
Private-B → NAT-B

Architecture:

AZ-A                              AZ-B

Public-A                          Public-B
   |                                 |
 NAT-A                              NAT-B
   |                                 |
Private-A                        Private-B
   |                                 |
EC2-A                            EC2-B

This improves resilience and also avoids unnecessary cross-AZ traffic for normal outbound Internet access.

14. Availability Zone Failure Scenario

Suppose:

AZ-A

experiences a failure.

The design keeps the resources in:

AZ-B

independent of NAT-A.

Similarly, normal traffic from Private-B uses:

Private-B
    |
    v
NAT-B

rather than crossing to:

NAT-A

This is an important production networking design principle:

Keep private-subnet outbound traffic on a NAT Gateway in the same Availability Zone whenever possible.

15. Bastion Host

A Bastion Host was deployed in the public subnet.

Example:

Bastion
Public-A
Private IP: 10.40.1.x
Public IP: 3.110.105.144

The Bastion provides an administrative entry point into the private subnet.

The architecture becomes:

Laptop
   |
   | SSH
   v
Bastion
   |
   | VPC local routing
   |
   +------------+
   |            |
   v            v
Private-A    Private-B

The private EC2 instances do not need public IP addresses.

16. Bastion vs NAT Gateway

These two components perform completely different jobs.

Component	Purpose
Bastion	Allows administrators to access private servers
NAT Gateway	Allows private servers to access the Internet
Internet Gateway	Connects VPC to the Internet
Route Table	Determines where packets should go
Security Group	Controls allowed traffic

Important distinction:

Bastion
= INBOUND ADMINISTRATION

while:

NAT Gateway
= OUTBOUND INTERNET ACCESS
17. Security Group Design

Two Security Groups were used.

Bastion Security Group

Example:

SG-Day4-Bastion

Inbound:

TCP 22
Source: My Public IP /32

Outbound:

Allow required outbound traffic

This prevents arbitrary Internet hosts from SSHing into the Bastion.

18. Private EC2 Security Group

Example:

SG-Day4-Private

Inbound:

TCP 22
Source: SG-Day4-Bastion

This is preferable to using:

10.40.1.x/32

for the Bastion.

The Security Group reference expresses the intended relationship:

Bastion SG
     |
     | TCP/22
     v
Private EC2 SG

Therefore, if the Bastion's private IP changes, the security rule does not need to be changed.

19. Why My Laptop's Private IP Was Not Used in AWS

During the SSH troubleshooting process, the Windows machine showed a local address such as:

172.16.x.x

This should not be placed in the Bastion Security Group as the Internet source.

Why?

Because the laptop is behind a local router/NAT.

The path is:

Laptop
172.16.x.x
   |
   v
Home/Local Router
   |
   | NAT
   v
Internet
   |
   v
AWS Bastion

AWS normally sees the connection coming from the public Internet-facing IP.

Therefore the Bastion Security Group should allow:

YOUR CURRENT PUBLIC IP/32

not the laptop's local RFC1918 address.

20. SSH Agent Forwarding

The private SSH key was stored on the Windows machine.

Instead of copying the private key into the Bastion, SSH agent forwarding was used.

Architecture:

Windows Laptop
     |
     | SSH Agent
     |
     v
Bastion
     |
     | forwarded authentication
     v
Private EC2

The private key remains on the local machine.

The Bastion does not need a copy of the .pem private key.

This is a more secure administrative approach than manually copying private keys onto intermediate servers.

21. SSH Agent Verification

On the Windows machine:

ssh-add .\Day3-ENI-Key.pem

Then:

ssh-add -L

The SSH public key was displayed, confirming that the key had been loaded into the local SSH agent.

The Bastion was then accessed using agent forwarding:

ssh -A -i .\Day3-ENI-Key.pem ec2-user@<BASTION_PUBLIC_IP>

Inside the Bastion:

ssh-add -L

returned the forwarded public key.

This confirmed:

Windows SSH Agent
        |
        v
SSH Agent Forwarding
        |
        v
Bastion
        |
        v
Private EC2
22. VPC Local Routing

One important networking concept verified in this lab was the automatically created:

10.40.0.0/16 → local

route.

This allows resources within the VPC to communicate across subnets.

For example:

Bastion
10.40.1.x
     |
     | destination = 10.40.12.x
     v
Route Table
     |
     | 10.40.0.0/16 → local
     v
Private-B
10.40.12.x

No Internet Gateway or NAT Gateway is required for this internal communication.

23. Cross-AZ Connectivity

The Bastion was located in:

ap-south-1a

while Private-B was located in:

ap-south-1b

The two resources can still communicate because both addresses belong to:

10.40.0.0/16

The VPC's local route handles this traffic.

Conceptually:

Bastion
10.40.1.x
   |
   | 10.40.0.0/16 → local
   |
   v
Private-B
10.40.12.x

The Availability Zone boundary does not mean the VPC becomes a separate network.

24. Important Difference: NAT Traffic vs Internal Traffic
Internal traffic
Bastion
   |
   v
10.40.0.0/16 → local
   |
   v
Private EC2

No NAT is required.

Internet-bound traffic
Private EC2
   |
   v
0.0.0.0/0 → NAT
   |
   v
NAT EIP
   |
   v
IGW
   |
   v
Internet

NAT is required because the private instance does not have a public Internet-routable source address.

25. Why the Private EC2 Does Not Need a Public IP

The private EC2 is intentionally configured without a public IPv4 address.

Example:

Private EC2-A

Private IP:
10.40.11.x

Public IPv4:
None

This provides a stronger network boundary.

The server can:

✓ communicate with VPC resources
✓ receive administrative access through Bastion
✓ initiate outbound Internet connections through NAT

while not being directly addressable from the public Internet.

26. Verification

The following checks were used to verify the architecture.

VPC
aws ec2 describe-vpcs

Purpose:

Verify:

VPC exists
VPC CIDR
VPC ID
DNS configuration
Subnets
aws ec2 describe-subnets

Purpose:

Verify:

subnet CIDRs
Availability Zones
VPC association
subnet IDs

Expected structure:

Public-A    10.40.1.0/24
Public-B    10.40.2.0/24
Private-A   10.40.11.0/24
Private-B   10.40.12.0/24
Route Tables
aws ec2 describe-route-tables

Verify:

Public:
0.0.0.0/0 → IGW

Private-A:
0.0.0.0/0 → NAT-A

Private-B:
0.0.0.0/0 → NAT-B
NAT Gateways
aws ec2 describe-nat-gateways

Verify:

NAT-A → Public-A
NAT-B → Public-B

and verify their associated Elastic IPs.

27. Private EC2 Route Verification

Inside Private EC2-A:

ip route

The important route should conceptually look like:

default via 10.40.11.1 dev eth0
10.40.11.0/24 dev eth0 ...

The default route sends Internet-bound traffic toward the subnet's routing infrastructure.

The AWS subnet route table then determines:

0.0.0.0/0 → NAT-A
28. NAT Internet Verification

From Private EC2-A:

curl -4 https://checkip.amazonaws.com

The returned public IP should correspond to the Elastic IP associated with NAT-A.

Conceptually:

Private EC2-A
10.40.11.x
      |
      v
NAT-A
      |
      v
NAT-A EIP
      |
      v
Internet

Therefore:

curl public IP
        =
NAT-A EIP

The same test can be performed from Private EC2-B.

Expected result:

Private EC2-B
      |
      v
NAT-B
      |
      v
NAT-B EIP
29. AWS CLI Credential Lesson

When the AWS CLI was executed from the Bastion:

aws ec2 describe-instances ...

the command returned:

Unable to locate credentials.

This was not a networking failure.

It demonstrated an important distinction:

AWS Console access
        ≠
AWS CLI authentication

The Bastion did not automatically receive AWS IAM credentials merely because it was running inside AWS.

For this lab, AWS credentials were intentionally not installed on the Bastion.

The required private EC2 information could instead be obtained from the AWS Console.

In a production environment, if an EC2 instance genuinely needs AWS API access, an IAM role attached to the instance would generally be preferred over manually storing long-lived credentials.

30. Troubleshooting Experience

During the lab, SSH connectivity to the Bastion initially failed.

The troubleshooting process demonstrated several real-world networking concepts.

The first important check was:

Test-NetConnection <BASTION_PUBLIC_IP> -Port 22

This checks whether TCP port 22 can be reached from the client.

The investigation included:

Local machine IP
Local gateway
AWS Security Group
Current public IP
EC2 public IP
SSH key
Port 22

The Security Group initially contained an outdated public IP.

The current public IP was different.

After updating the SSH inbound rule to the correct current public IP/32, Bastion SSH connectivity succeeded.

This demonstrated an important operational issue:

A user's public IP can change, causing an otherwise correct Security Group rule to stop matching.

31. SSH Authentication vs Network Connectivity

Another important distinction learned during troubleshooting:

Network connectivity problem

Example:

Connection timed out

Possible causes:

Security Group
NACL
route
wrong public IP
Internet connectivity
firewall
SSH authentication problem

Example:

Permission denied (publickey)

Possible causes:

wrong private key
wrong username
wrong key pair
key permissions
SSH configuration

These are different layers.

A successful TCP connection to port 22 does not automatically mean SSH authentication will succeed.

32. Final Traffic Flows
Flow 1 — Administrator → Bastion
Laptop
   |
   | SSH TCP/22
   v
Internet
   |
   v
IGW
   |
   v
Bastion
Public IP

Security Group:

TCP/22
Source = Administrator's public IP/32
Flow 2 — Bastion → Private EC2
Bastion
10.40.1.x
   |
   | VPC local route
   | 10.40.0.0/16 → local
   v
Private EC2
10.40.11.x / 10.40.12.x

Security Group:

TCP/22
Source = SG-Day4-Bastion
Flow 3 — Private EC2 → Internet
Private EC2
     |
     v
Private Route Table
     |
     | 0.0.0.0/0 → NAT
     v
NAT Gateway
     |
     | Source NAT
     v
Elastic IP
     |
     v
IGW
     |
     v
Internet
Flow 4 — Internet → Private EC2 Response
Internet
    |
    v
IGW
    |
    v
NAT Gateway
    |
    | NAT state mapping
    v
Private EC2

The private EC2 does not become directly Internet-addressable merely because it initiated an outbound connection.

33. Why the NAT Gateway Is in a Public Subnet

A NAT Gateway itself needs a path to the Internet.

Therefore:

NAT Gateway
      |
      v
Public Subnet
      |
      v
Public Route Table
      |
      | 0.0.0.0/0 → IGW
      |
      v
Internet Gateway
      |
      v
Internet

If the NAT Gateway were placed in a private subnet without an appropriate Internet path, it could not perform its intended Internet connectivity role.

Therefore:

NAT Gateway belongs in a public subnet.

Private instances use the NAT Gateway but do not become public themselves.

34. Core Networking Relationships

The most important relationships learned in this lab are:

VPC
 |
 +---- Subnets
 |       |
 |       +---- Public Subnets
 |       |
 |       +---- Private Subnets
 |
 +---- Route Tables
 |       |
 |       +---- Public → IGW
 |       |
 |       +---- Private → NAT
 |
 +---- Internet Gateway
 |
 +---- NAT Gateways
 |
 +---- EC2 / ENIs
 |
 +---- Security Groups

More specifically:

Private EC2
    |
    v
Subnet
    |
    v
Associated Route Table
    |
    v
0.0.0.0/0
    |
    v
NAT Gateway
    |
    v
Elastic IP
    |
    v
Internet Gateway
    |
    v
Internet
35. Key Lessons
1. Public subnet does not mean public CIDR
10.40.1.0/24

is still a private RFC1918 address range.

The subnet is considered public because its route table provides a path to the IGW.

2. Route tables determine packet direction

For example:

0.0.0.0/0 → IGW

means Internet-bound traffic uses the Internet Gateway.

While:

0.0.0.0/0 → NAT

means Internet-bound traffic uses the NAT Gateway.

3. NAT Gateway provides outbound Internet access

Private instances can initiate Internet connections without having public IP addresses.

4. NAT Gateway does not make a private EC2 public

The private EC2 still has only its private IP.

The NAT Gateway performs source translation for outbound traffic.

5. Elastic IP provides stable public identity

The NAT Gateway's EIP is the public source identity visible to external Internet services.

6. One NAT per AZ improves resilience
Private-A → NAT-A
Private-B → NAT-B

is preferable to forcing both AZs through one NAT Gateway for a production-oriented architecture.

7. Bastion and NAT are different
Bastion = inbound administrative access
NAT     = outbound Internet access
8. Security Groups filter traffic

Security Groups do not determine the routing path.

The route table determines:

WHERE traffic goes

while the Security Group determines:

WHETHER the traffic is allowed
9. VPC local routing enables internal communication

Resources in different subnets and even different Availability Zones can communicate through the VPC's automatic local route when security controls permit it.

10. SSH agent forwarding avoids copying private keys

The private key can remain on the administrator's machine while the Bastion forwards authentication to the private EC2.

36. Verification Checklist

The completed architecture should satisfy the following:

[✓] VPC created
[✓] VPC CIDR = 10.40.0.0/16
[✓] Two Availability Zones used
[✓] Public-A created
[✓] Public-B created
[✓] Private-A created
[✓] Private-B created
[✓] Internet Gateway attached
[✓] Public route table configured
[✓] Public subnets associated
[✓] NAT-A deployed
[✓] NAT-B deployed
[✓] EIP assigned to NAT-A
[✓] EIP assigned to NAT-B
[✓] Private-A routes Internet traffic to NAT-A
[✓] Private-B routes Internet traffic to NAT-B
[✓] Bastion deployed
[✓] Bastion SSH verified
[✓] SSH agent forwarding verified
[✓] Private EC2s have no public IP
[✓] Bastion → Private EC2 connectivity verified
[✓] Private EC2 → Internet tested through NAT
[✓] NAT EIP verified as external source IP
[✓] Cross-AZ VPC connectivity understood
37. Production Design Principles Demonstrated

This lab goes beyond simply creating a NAT Gateway.

It demonstrates several production networking principles:

Segmentation
Public Tier
Private Application Tier

are separated into different subnets.

Least Exposure

Private EC2 instances do not receive public IP addresses.

Controlled Administration

Administrative access enters through a Bastion.

Centralized Egress

Private workloads use NAT for Internet-bound traffic.

AZ Resilience

Each Availability Zone has its own NAT Gateway.

Route Isolation

Different subnet groups use different route tables.

Security Group Referencing

The private servers trust the Bastion Security Group instead of a hardcoded IP.

38. Final Architecture
                                      INTERNET
                                          |
                                          |
                               +--------------------+
                               |   Internet Gateway  |
                               +--------------------+
                                  /              \
                                 /                \
                                /                  \
                         Public-A                  Public-B
                      10.40.1.0/24              10.40.2.0/24
                           |                         |
                           |                         |
                         NAT-A                     NAT-B
                         EIP-A                     EIP-B
                           |                         |
                           |                         |
                     Private-A                   Private-B
                   10.40.11.0/24              10.40.12.0/24
                           |                         |
                           |                         |
                         EC2-A                     EC2-B
                      No Public IP              No Public IP
                           ^
                           |
                           |
                     +-----------+
                     |  Bastion  |
                     | Public-A  |
                     +-----------+
                           ^
                           |
                           |
                       ADMIN PC
39. Final Result

Successfully designed and implemented a two-Availability-Zone AWS VPC architecture with public and private subnet segmentation, Internet Gateway connectivity, redundant NAT Gateways, Elastic IPs, route-table engineering, Bastion-based administration, Security Group relationships, and private EC2 outbound Internet access.

The lab demonstrated the complete packet path from a private workload to the Internet:

Private EC2
    ↓
Private Route Table
    ↓
NAT Gateway
    ↓
Elastic IP
    ↓
Internet Gateway
    ↓
Internet

It also demonstrated secure administrative access:

