# Day 21: AWS Networking (VPC, SG, NACL, Route Tables, Peering, TGW)

## Learning Objectives
By the end of Day 21, you will:
- Master all AWS VPC networking components in depth
- Distinguish Security Groups from NACLs and configure both
- Implement VPC peering and Transit Gateway topologies
- Design production-grade AWS network architectures

**Estimated Time:** 3-4 hours

---

## Notes

### AWS VPC Deep Dive

#### Default VPC
- Every AWS account has a default VPC per region
- CIDR: `172.31.0.0/16`; one /20 subnet per AZ
- Comes with IGW, route tables, and default SG preconfigured
- Suitable for development; create custom VPCs for production

#### VPC Limits (per region, default)
- VPCs: 5
- Subnets per VPC: 200
- Security Groups per VPC: 2500
- Rules per SG: 60 inbound + 60 outbound

### Security Groups (SG)

Security Groups act as **stateful virtual firewalls at the instance/ENI level**.

**Key rules:**
- Stateful: return traffic automatically allowed
- Allow-only: cannot explicitly deny
- Evaluated as a whole (all rules processed, most permissive wins)
- Can reference other SGs as source/destination

#### SG Reference (Security Group Chaining)
```
# Instead of hardcoding IPs, reference another SG
# alb-sg: allows 0.0.0.0/0:443
# app-sg: allows inbound from alb-sg (not the ALB IP)

Inbound rule in app-sg:
  Type: Custom TCP | Port: 8080 | Source: sg-alb-id
```

This is more maintainable and automatically updates when ALB scales.

#### SG Limits and Best Practices
```bash
# AWS CLI: Create and configure a Security Group
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web server security group" \
  --vpc-id vpc-12345678

aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-12345678 \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
```

### Network ACLs (NACLs)

NACLs act as **stateless packet filters at the subnet boundary**.

**Key rules:**
- Stateless: both inbound AND outbound must be explicitly allowed
- Support both Allow and Deny rules
- Rules evaluated in numerical order (lowest first); stop at first match
- Rule 32767 (default): deny all

#### Proper NACL for Web Traffic
```
Inbound Rules:
100  TCP  443   0.0.0.0/0        ALLOW   ← HTTPS from internet
110  TCP  80    0.0.0.0/0        ALLOW   ← HTTP from internet
120  TCP  22    10.0.0.0/8       ALLOW   ← SSH from internal
130  TCP  1024-65535  0.0.0.0/0  ALLOW   ← Return traffic (ephemeral ports!)
*    ALL  ALL   0.0.0.0/0        DENY

Outbound Rules:
100  TCP  443   0.0.0.0/0        ALLOW   ← HTTPS to internet
110  TCP  80    0.0.0.0/0        ALLOW   ← HTTP to internet
120  TCP  1024-65535  0.0.0.0/0  ALLOW   ← Ephemeral ports for responses
*    ALL  ALL   0.0.0.0/0        DENY
```

### Route Tables in Depth

#### Main vs Custom Route Tables
- **Main route table:** Applied to subnets not explicitly associated with another
- **Custom route table:** Created and associated manually with specific subnets

#### Route Priority (Longest Prefix Match)
```
Routes in a route table:
10.0.0.0/16  → local           (most specific for VPC traffic)
0.0.0.0/0    → igw-xxx         (default for internet)

Packet to 10.0.1.50 → local (more specific wins)
Packet to 8.8.8.8   → igw-xxx (only default matches)
```

#### Targets for Routes
| Target | Use Case |
|--------|---------|
| local | Within-VPC traffic (automatic) |
| igw-xxx | Internet access (public subnets) |
| nat-xxx | Outbound internet for private subnets |
| pcx-xxx | VPC peering connection |
| tgw-xxx | Transit Gateway |
| vgw-xxx | Virtual Private Gateway (VPN) |
| vpce-xxx | VPC Endpoint (S3, DynamoDB) |
| eni-xxx | Network appliance (firewall, NVA) |

### VPC Peering

```
Account A                        Account B
VPC-A (10.0.0.0/16) ←─── pcx ───→ VPC-B (172.16.0.0/16)

Route in VPC-A: 172.16.0.0/16 → pcx-xxxxxxxx
Route in VPC-B: 10.0.0.0/16  → pcx-xxxxxxxx

Security Groups must also allow the traffic!
```

**Peering characteristics:**
- Uses AWS backbone (not internet)
- Cross-account: requester/accepter model
- Cross-region: inter-region VPC peering (latency varies)
- Non-transitive (no hub routing)
- CIDRs cannot overlap
- DNS hostnames must be enabled for private DNS to work across peers

```bash
# Create peering connection
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-aaa \
  --peer-vpc-id vpc-bbb \
  --peer-region us-west-2  # for cross-region

# Accept the connection (from peer account/region)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id pcx-12345678
```

### AWS Transit Gateway (TGW)

Transit Gateway is a **regional network transit hub** that connects VPCs and on-premises networks.

#### Why TGW over Peering?
- Peering = n*(n-1)/2 connections for full mesh (10 VPCs = 45 connections!)
- TGW = each VPC has 1 attachment → hub routing (10 VPCs = 10 attachments)
- Transitive routing supported
- Scales to thousands of VPCs

```
VPC-A ──┐
VPC-B ──┤
VPC-C ──┼──→ Transit Gateway ──→ VPN → On-premises
VPC-D ──┤
VPC-E ──┘
```

#### TGW Route Tables
- Each TGW has route tables (separate from VPC route tables)
- Attachments can be associated with specific TGW route tables
- Supports route propagation and static routes
- Can segment traffic (e.g., prod VPCs cannot route to dev VPCs)

```bash
# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "Production TGW" \
  --options AutoAcceptSharedAttachments=enable \
  --query 'TransitGateway.TransitGatewayId' --output text)

# Attach a VPC
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id vpc-aaa \
  --subnet-ids subnet-1a subnet-1b

# Update VPC route table to send traffic through TGW
aws ec2 create-route \
  --route-table-id rtb-xxx \
  --destination-cidr-block 10.1.0.0/16 \
  --transit-gateway-id $TGW_ID
```

### VPC Endpoints

#### Gateway Endpoints (Free)
- S3 and DynamoDB only
- Route table entry: `pl-xxxxx (S3) → vpce-xxx`
- No extra hops, stays within AWS backbone

#### Interface Endpoints (PrivateLink)
- 100+ AWS services + customer/partner services
- Creates an ENI in your subnet with a private IP
- Cost: ~$0.01/hour + $0.01/GB
- Critical for compliance environments (no internet)

```bash
# Create S3 gateway endpoint (free)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private

# Create interface endpoint for ECR
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxx \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --subnet-ids subnet-private-1a subnet-private-1b \
  --security-group-ids sg-endpoint
```

### VPC Flow Logs
```bash
# Enable Flow Logs for entire VPC → CloudWatch
aws ec2 create-flow-logs \
  --resource-ids vpc-xxx \
  --resource-type VPC \
  --traffic-type ALL \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123:role/FlowLogsRole

# Query with CloudWatch Insights
# fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, protocol, action
# | filter action = "REJECT"
# | stats count() by srcAddr
# | sort by count desc
# | limit 20
```

---

## Hands-On Labs

### Lab 1: VPC Inspection Script
```bash
#!/bin/bash
# Audit an AWS VPC configuration

VPC_ID=${1:-$(aws ec2 describe-vpcs --query 'Vpcs[0].VpcId' --output text 2>/dev/null)}

if [ -z "$VPC_ID" ] || [ "$VPC_ID" = "None" ]; then
    echo "No VPC found or AWS CLI not configured."
    echo "This lab requires AWS CLI with appropriate permissions."
    exit 0
fi

echo "VPC Audit: $VPC_ID"
echo "===================="

echo -e "\n[Subnets]"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table

echo -e "\n[Route Tables]"
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId]]' \
  --output text

echo -e "\n[Internet Gateways]"
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query 'InternetGateways[*].InternetGatewayId' --output text

echo -e "\n[NAT Gateways]"
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId]' --output table
```

### Lab 2: Security Group Auditor
```bash
#!/bin/bash
# Find overly permissive security groups

echo "Security Group Audit"
echo "===================="

if ! command -v aws &>/dev/null; then
    echo "AWS CLI required. Install with: pip install awscli"
    exit 0
fi

echo "Security Groups with 0.0.0.0/0 inbound rules:"
aws ec2 describe-security-groups \
  --query "SecurityGroups[?IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']]].[GroupId,GroupName,IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']].[FromPort,ToPort]]" \
  --output json 2>/dev/null | python3 -c "
import json, sys
data = json.load(sys.stdin)
for sg in data:
    print(f'  {sg[0]} ({sg[1]})')
    for rule in sg[2]:
        if rule:
            for r in rule:
                print(f'    Ports: {r[0]}-{r[1]} from 0.0.0.0/0')
" 2>/dev/null || echo "Could not query AWS. Check credentials."
```

### Lab 3: TGW Architecture Planner
```python
#!/usr/bin/env python3

def plan_transit_gateway(vpcs):
    """Plan a Transit Gateway network"""
    print("Transit Gateway Network Plan")
    print("=" * 50)
    print(f"\n{'VPC Name':<20}{'CIDR':<20}{'Attachment'}")
    print("-" * 50)

    attachments = []
    for i, (name, cidr, subnet_count) in enumerate(vpcs):
        att_id = f"tgw-attach-{i+1:04d}"
        attachments.append(att_id)
        print(f"{name:<20}{cidr:<20}{att_id}")

    print(f"\nTotal VPC attachments: {len(attachments)}")
    peer_connections = len(vpcs) * (len(vpcs)-1) // 2
    print(f"VPC Peering equivalent: {peer_connections} connections")
    print(f"TGW connections needed: {len(vpcs)} (savings: {peer_connections - len(vpcs)})")

    print("\nTGW Route Table entries needed:")
    for name, cidr, _ in vpcs:
        print(f"  {cidr:<20} → tgw-attach (from {name})")

vpcs = [
    ("prod-vpc", "10.0.0.0/16", 6),
    ("staging-vpc", "10.1.0.0/16", 4),
    ("dev-vpc", "10.2.0.0/16", 4),
    ("shared-services", "10.10.0.0/16", 2),
    ("security-vpc", "10.20.0.0/16", 2),
]
plan_transit_gateway(vpcs)
```

---

## Sample Exercises

1. **SG vs NACL:** An EC2 instance is in a subnet with NACL rule 100 allowing TCP 443 inbound. The instance's SG allows TCP 443 from the load balancer's SG. An external client hits the ALB on 443. Trace the path, explaining each security check.
2. **TGW Segmentation:** You have Prod, Dev, and Shared-Services VPCs. Prod and Dev should both access Shared-Services but not each other. How do you configure TGW route tables?
3. **Route Table Debugging:** An EC2 in a private subnet can't reach the internet, but you have a NAT Gateway. You check the route table and see `0.0.0.0/0 → igw-xxx`. What's wrong?
4. **VPC Endpoint:** Without a VPC endpoint, what path does traffic take from a private EC2 instance to S3? Why is this a problem for security-sensitive environments?
5. **NACL Troubleshooting:** You add a NACL rule allowing TCP 8080 inbound (rule 100), but clients still can't connect. What did you forget?

## Solutions

1. **Traffic path with SG+NACL:**
   - Client → ALB (internet, no NACL on ALB directly — it's managed)
   - ALB → EC2 subnet NACL inbound: checks rule 100 (TCP 443 ALLOW) ✓
   - ALB → EC2 SG inbound: checks if ALB SG is allowed on port 443 ✓
   - EC2 responds → SG outbound: stateful, automatically allowed ✓
   - EC2 subnet NACL outbound: must explicitly allow ephemeral ports → if missing, response drops ✗

2. **TGW Segmentation:**
   - Create two TGW route tables: `prod-rt` and `dev-rt`
   - Prod VPC attachment → associated with `prod-rt`; propagate Shared-Services
   - Dev VPC attachment → associated with `dev-rt`; propagate Shared-Services
   - Shared-Services routes to both (propagate to both route tables)
   - Prod and Dev are NOT in each other's route tables → cannot communicate

3. **Route table bug:** Private subnets should route `0.0.0.0/0 → nat-xxx` (the NAT Gateway ID), not `igw-xxx`. The Internet Gateway is only for public subnets with public IPs. Fix: replace the IGW route with the NAT Gateway route.

4. **EC2 to S3 without VPC Endpoint:** Traffic goes: EC2 → NAT Gateway → Internet → S3 public endpoint. Problems: (a) NAT Gateway costs per GB, (b) traffic leaves the AWS network to the public internet, (c) violates compliance policies requiring all traffic to stay within AWS. Fix: Use a free S3 Gateway Endpoint.

5. **NACL TCP 8080 still blocked:** For TCP, you also need to allow **outbound ephemeral ports** (1024-65535) so the server's response can reach the client. Add outbound NACL rule: TCP 1024-65535 → 0.0.0.0/0 ALLOW.

## Completion Checklist
- [ ] Configure Security Groups with SG-to-SG references
- [ ] Write NACLs with correct ephemeral port rules
- [ ] Explain VPC peering setup and route table requirements
- [ ] Design a Transit Gateway topology for multiple VPCs
- [ ] Configure VPC Endpoints for S3 and AWS services
- [ ] Enable and query VPC Flow Logs

## Key Takeaways
- Security Groups = stateful, instance-level; NACLs = stateless, subnet-level
- NACLs require explicit outbound rules for ephemeral return ports (1024-65535)
- VPC peering is non-transitive; Transit Gateway solves hub-and-spoke at scale
- VPC Endpoints keep traffic within AWS backbone — mandatory for compliance
- Always tag and document your VPC resources for maintainability

## Next Steps
Proceed to [Day 22: Azure/GCP Networking](../Day_22/notes_and_exercises.md).