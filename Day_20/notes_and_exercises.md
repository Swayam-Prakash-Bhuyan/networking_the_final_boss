# Day 20: Cloud Networking Basics (VPC, Subnets, Gateways, Peering)

## Learning Objectives
By the end of Day 20, you will:
- Understand the core components of cloud networking (VPC, subnets, IGW, NGW)
- Design multi-tier VPC architectures with public and private subnets
- Configure routing, internet access, and inter-VPC connectivity
- Apply cloud networking concepts across AWS, GCP, and Azure

**Estimated Time:** 3-4 hours

---

## Notes

### What is a VPC?
A **Virtual Private Cloud (VPC)** is an isolated, logically defined network within a cloud provider's infrastructure. It gives you full control over:
- IP address range (CIDR block)
- Subnets (public and private)
- Route tables
- Gateways (internet, NAT, VPN)
- Security (Security Groups, NACLs)

### VPC Core Components

#### CIDR Block
- The IP address range for your VPC
- AWS: /16 to /28 (cannot change after creation)
- Example: `10.0.0.0/16` gives 65,536 IPs

#### Subnets
- Subdivisions of the VPC CIDR, tied to a **single Availability Zone**
- **Public subnet:** Has route to Internet Gateway — instances can have public IPs
- **Private subnet:** No direct internet route — uses NAT Gateway for outbound only
- **Isolated/DB subnet:** No internet route at all — only accessible internally

#### Internet Gateway (IGW)
- Horizontally scaled, redundant, highly available
- Enables **bidirectional** internet access for instances with public IPs
- One IGW per VPC
- Must be attached to the VPC and referenced in route tables

#### NAT Gateway (NGW)
- Allows **outbound** internet access for private subnet instances
- Fully managed by the cloud provider
- Placed in a **public subnet**; private subnets route `0.0.0.0/0` through it
- Charged per hour + per GB transferred
- One per AZ recommended for high availability

#### Route Tables
- Each subnet is associated with exactly one route table
- Routes: destination CIDR → target (IGW, NGW, VPC peering, TGW, local)
- "Local" route is automatically added for within-VPC traffic

### Reference VPC Architecture

```
VPC: 10.0.0.0/16
│
├── AZ-a
│   ├── Public Subnet: 10.0.1.0/24 ──→ IGW (internet)
│   │       Route table: 0.0.0.0/0 → igw-xxx
│   │                    10.0.0.0/16 → local
│   │
│   ├── Private Subnet: 10.0.11.0/24 ──→ NAT GW (outbound only)
│   │       Route table: 0.0.0.0/0 → nat-xxx
│   │                    10.0.0.0/16 → local
│   │
│   └── DB Subnet: 10.0.21.0/24 (no internet route)
│
├── AZ-b
│   ├── Public Subnet:  10.0.2.0/24
│   ├── Private Subnet: 10.0.12.0/24
│   └── DB Subnet:      10.0.22.0/24
│
└── AZ-c
    ├── Public Subnet:  10.0.3.0/24
    ├── Private Subnet: 10.0.13.0/24
    └── DB Subnet:      10.0.23.0/24
```

### VPC Peering

VPC Peering creates a private, low-latency connection between two VPCs:
- Can be within the same account, different accounts, or different regions
- Traffic stays on the cloud provider's backbone (no internet)
- CIDRs must not overlap
- **Non-transitive:** A↔B and B↔C does NOT mean A↔C; you must peer A↔C separately
- Update route tables on both sides to enable traffic

```
VPC-A (10.0.0.0/16) ←─── Peering ───→ VPC-B (172.16.0.0/16)
Route in VPC-A: 172.16.0.0/16 → pcx-xxx
Route in VPC-B: 10.0.0.0/16  → pcx-xxx
```

### AWS-Specific Components

#### Elastic Network Interface (ENI)
- Virtual NIC attached to an EC2 instance
- Has a primary private IP, optional secondary IPs, and an optional public IP
- Can be detached and re-attached (floating IPs)
- Security Groups are applied at the ENI level

#### VPC Endpoints
Allow private connectivity to AWS services without internet or NAT:
- **Gateway endpoint:** S3, DynamoDB (free, routes via route table)
- **Interface endpoint (PrivateLink):** 100+ AWS services (ENI in your subnet, costs money)

#### VPC Flow Logs
- Capture metadata about IP traffic flowing through VPC, subnet, or ENI
- Stored in CloudWatch Logs or S3
- **Not packet captures** — just metadata (src, dst, port, protocol, action, bytes)
- Essential for security audits and troubleshooting

### GCP VPC Equivalents

| AWS | GCP | Notes |
|-----|-----|-------|
| VPC | VPC Network | GCP VPCs are global (not regional) |
| Subnet | Subnet | Regional in GCP |
| IGW | (automatic) | GCP handles automatically |
| NAT Gateway | Cloud NAT | Regional, auto-scalable |
| VPC Peering | VPC Network Peering | Same non-transitive rule |
| Route Table | Routes | Applied at network level |
| Security Group | Firewall Rules | GCP: tag/SA-based, not per-instance |

### Azure VPC Equivalents

| AWS | Azure | Notes |
|-----|-------|-------|
| VPC | Virtual Network (VNet) | |
| Subnet | Subnet | |
| IGW | (automatic for public IPs) | |
| NAT Gateway | NAT Gateway | |
| VPC Peering | VNet Peering | |
| Security Group | NSG (Network Security Group) | Applied to subnet or NIC |
| Route Table | Route Table (UDR) | User Defined Routes |

---

## Hands-On Labs

### Lab 1: AWS VPC Design (CLI)
```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]' \
  --query 'Vpc.VpcId' --output text)
echo "VPC: $VPC_ID"

# Create public subnet (AZ-a)
SUBNET_PUB=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]' \
  --query 'Subnet.SubnetId' --output text)

# Create private subnet (AZ-a)
SUBNET_PRIV=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.11.0/24 --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]' \
  --query 'Subnet.SubnetId' --output text)

# Create and attach Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Create public route table
RTB_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PUB --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RTB_PUB --subnet-id $SUBNET_PUB

echo "Setup complete:"
echo "  VPC:            $VPC_ID"
echo "  Public subnet:  $SUBNET_PUB"
echo "  Private subnet: $SUBNET_PRIV"
echo "  IGW:            $IGW_ID"
```

### Lab 2: VPC Flow Log Analysis (Simulated)
```python
#!/usr/bin/env python3
# Parse and analyze VPC Flow Log format

SAMPLE_FLOW_LOGS = """
2 123456789010 eni-abc123 10.0.1.50 10.0.2.100 443 52341 6 10 840 1620000000 1620000060 ACCEPT OK
2 123456789010 eni-abc123 203.0.113.5 10.0.1.50 22 51234 6 5 300 1620000010 1620000070 REJECT OK
2 123456789010 eni-abc123 10.0.1.50 8.8.8.8 53 61000 17 1 73 1620000020 1620000021 ACCEPT OK
2 123456789010 eni-abc123 192.168.1.1 10.0.1.50 3389 54321 6 0 0 1620000030 1620000090 REJECT OK
2 123456789010 eni-abc123 10.0.1.50 10.0.2.200 5432 52500 6 20 15000 1620000040 1620000100 ACCEPT OK
""".strip()

FIELDS = ['version','account','interface','srcaddr','dstaddr','srcport','dstport',
          'protocol','packets','bytes','start','end','action','log-status']

PROTOS = {'6': 'TCP', '17': 'UDP', '1': 'ICMP'}

print("VPC Flow Log Analysis")
print("=" * 65)
accepted = rejected = 0

for line in SAMPLE_FLOW_LOGS.split('\n'):
    fields = dict(zip(FIELDS, line.split()))
    proto = PROTOS.get(fields['protocol'], fields['protocol'])
    action = fields['action']
    print(f"{action:7} {fields['srcaddr']:>15}:{fields['srcport']:<6} → "
          f"{fields['dstaddr']:>15}:{fields['dstport']:<6} {proto}  {fields['bytes']}B")
    if action == 'ACCEPT': accepted += 1
    else: rejected += 1

print(f"\nSummary: {accepted} ACCEPTED, {rejected} REJECTED")
```

### Lab 3: VPC Architecture Planner
```python
#!/usr/bin/env python3
import ipaddress

def plan_vpc(vpc_cidr, azs, tiers):
    vpc = ipaddress.IPv4Network(vpc_cidr)
    print(f"\nVPC Architecture Plan: {vpc_cidr}")
    print(f"AZs: {azs}, Tiers: {tiers}")
    print("=" * 55)

    subnets = list(vpc.subnets(prefixlen_diff=8))  # /24s from /16
    idx = 0
    for tier in tiers:
        print(f"\n[{tier}]")
        for az in azs:
            if idx < len(subnets):
                subnet = subnets[idx]
                print(f"  {az}: {subnet}  ({subnet.num_addresses - 2} usable hosts)")
                idx += 1

plan_vpc("10.0.0.0/16",
         azs=["us-east-1a", "us-east-1b", "us-east-1c"],
         tiers=["Public", "Private-App", "Private-DB"])
```

---

## Practical Exercises

### Exercise 1: VPC CIDR Planner
```python
#!/usr/bin/env python3
import ipaddress

def vpc_subnet_plan(vpc_cidr, num_azs=3, tiers=None):
    if tiers is None:
        tiers = [("Public", "/24"), ("Private", "/24"), ("Database", "/25")]

    vpc = ipaddress.IPv4Network(vpc_cidr)
    print(f"VPC: {vpc_cidr}")
    print("-" * 50)

    all_subnets = []
    for tier_name, prefix_str in tiers:
        prefix_diff = int(prefix_str[1:]) - vpc.prefixlen
        tier_subnets = list(vpc.subnets(prefixlen_diff=prefix_diff))
        for az_idx in range(num_azs):
            if tier_subnets:
                s = tier_subnets[az_idx + (num_azs * [t[0] for t in tiers].index(tier_name))]
                hosts = s.num_addresses - 2
                print(f"  {tier_name:<15} AZ-{chr(97+az_idx)}: {str(s):<20} ({hosts} hosts)")

vpc_subnet_plan("10.10.0.0/16", num_azs=3)
```

### Exercise 2: Connectivity Matrix Generator
```python
#!/usr/bin/env python3

def print_connectivity_matrix(tiers):
    """Print which subnets can talk to which"""
    rules = {
        ("Internet", "Public"): "IGW",
        ("Public", "Private"): "Internal",
        ("Private", "Database"): "Internal",
        ("Private", "Internet"): "NAT GW",
    }
    print("Connectivity Matrix:")
    print(f"{'Source':<15} → {'Destination':<15} {'Path'}")
    print("-" * 50)
    for src, dst, allowed, path in [
        ("Internet", "Public-Web", "✓", "IGW"),
        ("Internet", "Private-App", "✗", "Blocked"),
        ("Internet", "Database", "✗", "Blocked"),
        ("Public-Web", "Private-App", "✓", "Security Group"),
        ("Private-App", "Database", "✓", "Security Group"),
        ("Private-App", "Internet", "✓", "NAT Gateway"),
        ("Database", "Internet", "✗", "No route"),
    ]:
        print(f"  {src:<18} {allowed} {dst:<18} ({path})")

print_connectivity_matrix(["Public", "Private", "Database"])
```

---

## Sample Exercises

1. **Subnet Design:** Design a VPC for a fintech app requiring strict isolation: public ALB tier, private app tier, private DB tier, management bastion — all across 3 AZs.
2. **Routing:** An EC2 instance in a private subnet cannot reach S3. It has a NAT Gateway. What are the possible causes?
3. **VPC Peering:** You have 5 VPCs that all need to communicate with each other. How many peering connections do you need? Is there a better solution?
4. **Flow Logs:** A security team notices unusual outbound traffic from a private subnet at 3 AM. How do you use Flow Logs to investigate?
5. **Cost Optimization:** You have 3 NAT Gateways (one per AZ). Application traffic is minimal. How can you reduce cost while maintaining resilience?

## Solutions

1. **Fintech VPC Design:**
   ```
   10.0.0.0/16:
   10.0.1-3.0/24 → ALB (public) — 3 AZs
   10.0.11-13.0/24 → App servers (private) — 3 AZs
   10.0.21-23.0/24 → RDS (private, no NAT) — 3 AZs
   10.0.31.0/28 → Bastion (public, restricted SG) — single AZ
   VPC Endpoints for S3, DynamoDB, Secrets Manager
   ```

2. **Private subnet can't reach S3 causes:** Route table doesn't have `0.0.0.0/0 → NAT GW`; NAT Gateway is in a private subnet itself (must be in public); NAT Gateway's EIP not attached; Security Group blocks outbound; S3 bucket policy denies; VPC endpoint overrides NAT but endpoint policy denies.

3. **5 VPCs full mesh:** n*(n-1)/2 = 5*4/2 = **10 peering connections**. Better solution: **Transit Gateway** — one TGW, each VPC peers once with it (5 attachments), enabling transitive routing.

4. **Flow Log investigation:** Filter Flow Logs for the private subnet CIDR as source, time range 2:30 AM–3:30 AM. Look for: unusual destination IPs, high byte counts, unexpected protocols. Cross-reference destination IPs with threat intelligence. Check which ENI generated the traffic → identify the EC2 instance.

5. **NAT Gateway cost reduction:** For minimal traffic, use a **single NAT Gateway** in one AZ and route all private subnets through it. Trade-off: if that AZ fails, all private subnets lose internet. For critical production, keep one per AZ. Also consider VPC Endpoints for S3/DynamoDB to eliminate NAT costs for those services.

## Completion Checklist
- [ ] Explain VPC components: CIDR, subnets, IGW, NAT GW, route tables
- [ ] Design a multi-AZ, multi-tier VPC architecture
- [ ] Configure route tables for public and private subnets
- [ ] Understand VPC peering (non-transitive, no overlapping CIDRs)
- [ ] Know equivalent services across AWS, GCP, and Azure
- [ ] Analyze VPC Flow Logs for troubleshooting and security

## Key Takeaways
- VPCs isolate cloud resources in a private, configurable network
- Public subnets have routes to IGW; private subnets use NAT GW for outbound only
- Route tables control all traffic flow; security groups control which traffic is allowed
- VPC peering is non-transitive — Transit Gateway solves hub-and-spoke at scale
- VPC Flow Logs are metadata, not packet captures, but invaluable for security auditing

## Next Steps
Proceed to [Day 21: AWS Networking (VPC, SG, NACL, Route Tables, Peering, TGW)](../Day_21/notes_and_exercises.md).