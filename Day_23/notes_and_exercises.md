# Day 23: Hybrid & Multi-Cloud Networking, Site-to-Site, Transit, SD-WAN

## Learning Objectives
By the end of Day 23, you will:
- Design hybrid network architectures connecting on-premises to cloud
- Configure site-to-site VPN connections across providers
- Understand Transit Gateway and Transit VNet architectures
- Explore SD-WAN concepts and multi-cloud connectivity patterns

**Estimated Time:** 3-4 hours

---

## Notes

### Hybrid Networking Overview

Hybrid networking connects **on-premises data centers** with **cloud environments** through secure, private, or dedicated connections.

**Why hybrid?**
- Gradual cloud migration — some workloads stay on-prem
- Compliance — sensitive data must remain on-prem
- Latency — some apps need local data processing
- Cost — not everything needs to be in the cloud
- Disaster recovery — cloud as DR site

### Connection Options Comparison

| Option | Bandwidth | Latency | Cost | Setup Time | Reliability |
|--------|-----------|---------|------|-----------|-------------|
| Site-to-Site VPN | Up to 1.25 Gbps | Variable (internet) | Low | Minutes | 99.9% |
| AWS Direct Connect | 1/10/100 Gbps | Consistent, low | High | Weeks | 99.9% – 99.99% |
| Azure ExpressRoute | 50 Mbps – 100 Gbps | Consistent | High | Weeks | 99.9% – 99.99% |
| GCP Cloud Interconnect | 10/100 Gbps | Consistent | High | Weeks | 99.9% – 99.99% |
| SD-WAN | Varies | Optimized | Medium | Days | Provider-dependent |

### Site-to-Site VPN Deep Dive

#### IPSec VPN Tunnel Architecture
```
On-Premises                              AWS
┌──────────┐                        ┌──────────┐
│ Customer │                        │ Virtual  │
│ Gateway  │───IPSec Tunnel 1──────→│ Private  │
│ Device   │───IPSec Tunnel 2──────→│ Gateway  │
└──────────┘                        └──────────┘
   (CGW)          Redundant               (VGW)
```

- **Two tunnels** per connection (AWS requires this for redundancy)
- Each tunnel terminates in a different AWS AZ
- Supports **BGP** (dynamic) or **static routing**
- BGP preferred: automatic failover, route exchange

#### BGP with Site-to-Site VPN
```
On-prem (AS 65000) ←─BGP─→ AWS (AS 7224)

On-prem advertises: 192.168.0.0/16
AWS VPC routes: 10.0.0.0/16

BGP benefits:
- Automatic failover between tunnels
- Route summarization
- Traffic engineering with BGP attributes
```

#### VPN Routing Considerations
```
Static routing: manually define which CIDRs go through VPN
  Pro: simple, predictable
  Con: no failover, admin overhead

BGP routing: routers exchange routes automatically
  Pro: failover, scalability, TE
  Con: more complex config
```

### AWS Direct Connect

Direct Connect (DX) provides a **dedicated private network connection** to AWS:

```
On-premises ──[leased line]──→ AWS Direct Connect Location ──→ AWS Region
              (carrier provides                (AWS equipment
               physical circuit)               at DX location)
```

#### DX Connection Types
- **Dedicated Connection:** 1, 10, or 100 Gbps — physical port at DX location
- **Hosted Connection:** Ordered through APN partner — 50 Mbps to 10 Gbps

#### Virtual Interfaces (VIFs)
- **Private VIF:** Connect to VPC resources (via VGW or TGW)
- **Public VIF:** Connect to AWS public endpoints (S3, SQS, etc.) without internet
- **Transit VIF:** Connect to Transit Gateway (multiple VPCs)

#### DX Redundancy Options
```
Level 1: Single connection (no redundancy)
Level 2: Two connections to same DX location
Level 3: Two connections to different DX locations (recommended)
Level 4: Two connections + backup VPN (maximum resilience)
```

### Transit Architecture Patterns

#### AWS Transit Gateway for Hybrid
```
On-premises
    ↓ (VPN or DX + Transit VIF)
Transit Gateway
    ↙     ↓     ↘
VPC-A  VPC-B  VPC-C
```

- Single point of attachment for on-prem connectivity
- Route tables control which VPCs can reach on-premises
- Transit Gateway inter-region peering for global reach

#### Azure Virtual WAN (vWAN)
Azure's equivalent of AWS Transit Gateway:
- **Hub-and-spoke at global scale**
- Automated spoke VNet connections
- Integrated VPN, ExpressRoute, and SD-WAN
- Supports User VPN (point-to-site) and Site-to-Site

#### GCP Network Connectivity Center
- Centralized hub for hybrid connectivity
- Connects Cloud VPN, Interconnect, and Router appliances
- Supports spoke-to-spoke traffic (not just hub-spoke)

### SD-WAN (Software-Defined WAN)

SD-WAN abstracts the WAN transport and provides intelligent routing across multiple links (MPLS, broadband, LTE, cloud VPN):

**Key SD-WAN features:**
- **Path selection:** Route traffic over best available link (latency, loss, jitter)
- **Application awareness:** Prioritize VoIP over the best path, bulk data over cheaper link
- **Zero-touch provisioning:** Branch offices come up automatically
- **Centralized management:** One pane of glass for all sites
- **Cloud on-ramp:** Direct connections to cloud providers without hairpinning through HQ

**SD-WAN vendors:** Cisco Viptela, VMware SD-WAN (VeloCloud), Fortinet, Palo Alto Prisma SD-WAN, Silver Peak

#### SD-WAN + Cloud Integration
```
Branch offices
    ↓ SD-WAN
SD-WAN Gateway ──→ AWS/Azure/GCP (via cloud on-ramp)
    ↓
HQ / Data Center
```

### Multi-Cloud Networking Patterns

#### Challenge: Two different cloud providers
- AWS VPCs and Azure VNets need to communicate
- No native peering between AWS and Azure
- Options:

1. **VPN between clouds:**
```
AWS VGW ──IPSec──→ Azure VPN Gateway
(Simple but uses internet, limited bandwidth)
```

2. **Dedicated interconnect via Exchange:**
```
AWS Direct Connect Location ──→ Equinix/Megaport ──→ Azure ExpressRoute
(High bandwidth, low latency, but expensive)
```

3. **SD-WAN overlay:**
```
SD-WAN node in AWS VPC ←→ SD-WAN node in Azure VNet
(Application-aware, managed centrally)
```

4. **Cloud Exchange / NaaS:**
- Equinix Fabric, Megaport, PacketFabric
- Interconnect clouds without physical presence

---

## Hands-On Labs

### Lab 1: Site-to-Site VPN Configuration (AWS CLI)
```bash
# Step 1: Create Customer Gateway (represents your on-prem device)
CGW_ID=$(aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.1 \        # Your on-prem public IP
  --bgp-asn 65000 \
  --query 'CustomerGateway.CustomerGatewayId' --output text)

# Step 2: Create Virtual Private Gateway
VGW_ID=$(aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --query 'VpnGateway.VpnGatewayId' --output text)

# Step 3: Attach VGW to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id $VGW_ID \
  --vpc-id vpc-xxx

# Step 4: Create VPN Connection
VPN_ID=$(aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW_ID \
  --vpn-gateway-id $VGW_ID \
  --options StaticRoutesOnly=false \  # Use BGP
  --query 'VpnConnection.VpnConnectionId' --output text)

# Step 5: Download config for on-prem device
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $VPN_ID \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text

echo "VPN $VPN_ID created. Download config and configure your on-prem device."
```

### Lab 2: Hybrid Connectivity Planner
```python
#!/usr/bin/env python3

class HybridConnPlanner:
    def __init__(self, bandwidth_gbps, budget_level, latency_sensitivity, sites):
        self.bandwidth = bandwidth_gbps
        self.budget = budget_level      # low/medium/high
        self.latency = latency_sensitivity  # low/medium/high
        self.sites = sites

    def recommend(self):
        recommendations = []

        if self.bandwidth > 10:
            recommendations.append(("Dedicated Interconnect (100G)", "High bandwidth, consistent latency"))
        elif self.bandwidth > 1:
            if self.latency == "high":
                recommendations.append(("Dedicated Interconnect (10G)", "Consistent latency"))
            else:
                recommendations.append(("Hosted Interconnect (1-10G)", "Cost-effective for moderate BW"))
        else:
            if self.latency == "high" and self.budget == "high":
                recommendations.append(("Dedicated Interconnect (1G)", "Consistent, but overkill"))
            else:
                recommendations.append(("Site-to-Site VPN", "Cost-effective for <1 Gbps"))

        if self.sites > 5:
            recommendations.append(("SD-WAN", "Centralized management for many sites"))

        if self.budget == "high" and self.latency == "high":
            recommendations.append(("Direct Connect + VPN backup", "Maximum resilience"))

        return recommendations

# Example scenarios
scenarios = [
    HybridConnPlanner(50, "high", "high", 3),
    HybridConnPlanner(0.5, "low", "medium", 10),
    HybridConnPlanner(5, "medium", "high", 2),
]

for i, planner in enumerate(scenarios, 1):
    print(f"\nScenario {i}: {planner.bandwidth} Gbps, {planner.budget} budget, "
          f"{planner.latency} latency sensitivity, {planner.sites} sites")
    for rec, reason in planner.recommend():
        print(f"  → {rec}: {reason}")
```

### Lab 3: Multi-Cloud Latency Test
```bash
#!/bin/bash
# Test latency to cloud provider endpoints

echo "Multi-Cloud Connectivity Test"
echo "============================="

# AWS regions
echo -e "\n[AWS Endpoints]"
for endpoint in "ec2.us-east-1.amazonaws.com" "ec2.eu-west-1.amazonaws.com" "ec2.ap-south-1.amazonaws.com"; do
    latency=$(ping -c 3 -W 2 $endpoint 2>/dev/null | tail -1 | awk -F'/' '{print $5}')
    echo "  $endpoint: ${latency:-timeout} ms"
done

# GCP endpoints
echo -e "\n[GCP Endpoints]"
for endpoint in "compute.googleapis.com" "storage.googleapis.com"; do
    latency=$(ping -c 3 -W 2 $endpoint 2>/dev/null | tail -1 | awk -F'/' '{print $5}')
    echo "  $endpoint: ${latency:-timeout} ms"
done

# Azure endpoints
echo -e "\n[Azure Endpoints]"
for endpoint in "management.azure.com" "login.microsoftonline.com"; do
    latency=$(ping -c 3 -W 2 $endpoint 2>/dev/null | tail -1 | awk -F'/' '{print $5}')
    echo "  $endpoint: ${latency:-timeout} ms"
done
```

---

## Practical Exercises

### Exercise 1: VPN Tunnel Health Monitor
```bash
#!/bin/bash
# Monitor VPN tunnel status via AWS CLI

VPN_ID=${1:-"vpn-xxxxxxxxxx"}

if ! command -v aws &>/dev/null; then
    echo "AWS CLI required"
    exit 0
fi

while true; do
    echo "=== VPN Tunnel Status: $(date) ==="
    aws ec2 describe-vpn-connections \
      --vpn-connection-ids $VPN_ID \
      --query 'VpnConnections[0].VgwTelemetry[*].[OutsideIpAddress,Status,StatusMessage,LastStatusChange]' \
      --output table 2>/dev/null || echo "Could not fetch VPN status"
    sleep 60
done
```

### Exercise 2: Bandwidth Estimator
```python
#!/usr/bin/env python3

def estimate_connection(daily_transfer_gb, peak_concurrent_sessions, rtl_ms_required):
    """Estimate required WAN bandwidth"""
    # Daily transfer → average Mbps
    seconds_per_day = 86400
    avg_mbps = (daily_transfer_gb * 8 * 1024) / seconds_per_day

    # Peak is typically 4x average
    peak_mbps = avg_mbps * 4

    # Session overhead
    session_overhead_mbps = peak_concurrent_sessions * 0.01  # 10 Kbps per session

    total_mbps = peak_mbps + session_overhead_mbps

    print(f"Bandwidth Analysis")
    print(f"  Daily transfer:        {daily_transfer_gb} GB")
    print(f"  Average throughput:    {avg_mbps:.1f} Mbps")
    print(f"  Peak throughput:       {peak_mbps:.1f} Mbps")
    print(f"  Session overhead:      {session_overhead_mbps:.1f} Mbps")
    print(f"  Recommended capacity:  {total_mbps:.0f} Mbps ({total_mbps/1000:.2f} Gbps)")
    print()

    if total_mbps < 100:
        rec = "Site-to-Site VPN (internet)"
    elif total_mbps < 1000:
        rec = "Hosted Direct Connect / Interconnect (1G)"
    elif total_mbps < 10000:
        rec = "Dedicated Direct Connect (10G)"
    else:
        rec = "Dedicated Direct Connect (100G) or multiple 10G"

    print(f"  Recommendation:        {rec}")
    if rtl_ms_required < 20:
        print(f"  Latency req ({rtl_ms_required}ms):  Use dedicated link, not VPN")
    else:
        print(f"  Latency req ({rtl_ms_required}ms):  VPN may be acceptable")

estimate_connection(daily_transfer_gb=500, peak_concurrent_sessions=1000, rtl_ms_required=50)
```

---

## Sample Exercises

1. **Connection choice:** A company needs to replicate 10 TB of data daily from on-prem to AWS with sub-20ms latency. What connectivity do you recommend?
2. **BGP failover:** You have two Direct Connect connections (primary and secondary). Both use BGP. How do you ensure primary is always preferred?
3. **SD-WAN use case:** A retail chain with 500 stores needs each store to connect to both AWS and Azure. Propose an architecture.
4. **Multi-cloud VPN:** Sketch the components needed for an IPSec VPN between an AWS VPC and an Azure VNet. What are the limitations?
5. **Cost analysis:** Compare monthly costs for a 1 Gbps dedicated connection vs IPSec VPN at 500 Mbps utilization (research current AWS pricing).

## Solutions

1. **10 TB daily, sub-20ms:** 10 TB/day = ~926 Mbps average, ~3.7 Gbps peak. **10G AWS Direct Connect** is required. VPN cannot guarantee sub-20ms latency (depends on internet path). Use BGP for automatic failover between connections.

2. **BGP primary preference:** On the secondary DX connection, advertise routes with a **longer AS path** (AS path prepending) or use **lower Local Preference** for routes learned via secondary. AWS side: routes from secondary have `AS_PATH length` increased, so primary is always preferred.

3. **SD-WAN for 500 stores:** Deploy SD-WAN appliances (virtual or physical) at each store. Use an SD-WAN controller in AWS or Azure. Cloud on-ramp features connect each store directly to AWS and Azure without hairpinning through HQ. Central management through SD-WAN orchestrator. Each store gets internet + optional MPLS backup.

4. **AWS-to-Azure VPN:** AWS side: Customer Gateway + Virtual Private Gateway (or TGW) + VPN Connection. Azure side: Virtual Network Gateway (VPN type) + Local Network Gateway (represents AWS side). Both configured with IPSec/IKEv2 and matching pre-shared keys. Limitations: ~1.25 Gbps max, internet-based (variable latency), no SLA for path quality.

5. **Cost comparison (approximate, check current AWS pricing):** 1G DX Dedicated: ~$1,620/month port + ~$0.03/GB transfer ≈ $1,620 + ~$460/month @ 500 Mbps. VPN: ~$36/month/tunnel + egress ~$0.09/GB ≈ $36 + ~$1,350/month @ 500 Mbps. DX is more expensive upfront but cheaper at scale; break-even depends on actual transfer volume.

## Completion Checklist
- [ ] Explain site-to-site VPN with IPSec and BGP
- [ ] Describe Direct Connect / Interconnect / ExpressRoute architectures
- [ ] Design a Transit Gateway-based hybrid network
- [ ] Understand SD-WAN benefits and use cases
- [ ] Plan multi-cloud connectivity using exchange providers
- [ ] Choose the right connectivity option based on bandwidth, latency, and cost

## Key Takeaways
- VPN is fast to deploy and cheap; dedicated links provide consistent performance
- Always deploy redundant connections (two VPN tunnels, two DX connections)
- BGP enables automatic failover; static routing requires manual intervention
- Transit Gateway centralizes hybrid connectivity for multiple VPCs
- SD-WAN is ideal for many distributed sites needing cloud-direct access
- Multi-cloud networking requires third-party exchanges or overlay VPN

## Next Steps
Proceed to [Day 24: Service Discovery & DNS in Cloud, Consul, CloudMap](../Day_24/notes_and_exercises.md).