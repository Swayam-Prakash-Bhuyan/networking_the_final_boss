# Day 22: Azure/GCP Networking (VNets, Firewalls, Peering, Hybrid, Cloud NAT)

## Learning Objectives
By the end of Day 22, you will:
- Master Azure Virtual Network architecture and security
- Understand GCP's global VPC model
- Configure VNet/VPC peering and hybrid connectivity
- Compare Azure and GCP networking with AWS equivalents

**Estimated Time:** 3-4 hours

---

## Notes

### Azure Networking

#### Azure Virtual Network (VNet)
- **Regional resource** — spans a region but NOT global
- CIDR: you define (e.g., `10.0.0.0/16`)
- Subnets are NOT tied to specific AZs — Azure handles AZ placement at the resource level
- No charge for VNet itself; charges for peering, gateways, and public IPs

#### Azure Core Components

| Component | AWS Equivalent | Purpose |
|-----------|---------------|---------|
| Virtual Network (VNet) | VPC | Isolated network |
| Subnet | Subnet | IP subdivision |
| NSG (Network Security Group) | Security Group + NACL | Stateful firewall rules |
| UDR (User Defined Routes) | Route Table | Custom routing |
| VNet Peering | VPC Peering | Private inter-VNet connectivity |
| VPN Gateway | Virtual Private Gateway | Site-to-site/point-to-site VPN |
| ExpressRoute | Direct Connect | Dedicated private WAN link |
| Azure Firewall | AWS Network Firewall | Managed L7 firewall |
| NAT Gateway | NAT Gateway | Outbound NAT for private subnets |
| Azure Bastion | Systems Manager Session Manager | Browser-based RDP/SSH |
| Private Endpoint | VPC Interface Endpoint | Private access to PaaS services |
| Service Endpoint | VPC Gateway Endpoint | Optimized path to Azure services |
| Azure Load Balancer | NLB | L4 load balancing |
| Application Gateway | ALB | L7 load balancing with WAF |
| Azure Front Door | CloudFront + Global Accelerator | Global HTTP load balancing |

#### Network Security Groups (NSG)
Azure NSGs can be applied to:
- **Subnets** (like NACLs) — applies to all resources in subnet
- **Network Interfaces** (like Security Groups) — applies to specific VM

NSGs are **stateful** (unlike NACLs!):

```
NSG Inbound Rules:
Priority | Name        | Port | Protocol | Source       | Action
100      | Allow-HTTPS | 443  | TCP      | 0.0.0.0/0   | Allow
110      | Allow-HTTP  | 80   | TCP      | 0.0.0.0/0   | Allow
200      | Allow-SSH   | 22   | TCP      | 10.0.0.0/8  | Allow
65000    | AllowVnetIn | Any  | Any      | VirtualNetwork | Allow
65001    | AllowAzureLBIn | Any | Any   | AzureLoadBalancer | Allow
65500    | DenyAll     | Any  | Any      | Any          | Deny
```

Azure provides built-in **service tags** (like `Internet`, `VirtualNetwork`, `AzureLoadBalancer`, `Storage`) so you don't need to hardcode IP ranges.

#### Azure VNet Peering
```
VNet-A (10.0.0.0/16) ←── Peering ──→ VNet-B (172.16.0.0/16)

Properties:
  allowVirtualNetworkAccess: true
  allowForwardedTraffic: true      ← needed for hub-spoke
  allowGatewayTransit: true        ← hub can share VPN gateway
  useRemoteGateways: true          ← spokes use hub's gateway
```

#### Azure Hub-Spoke Architecture
```
On-premises ──ExpressRoute──→ Hub VNet (10.0.0.0/16)
                                    │
                         ┌──────────┼──────────┐
                         ↓          ↓          ↓
                   Spoke 1       Spoke 2   Spoke 3
                 (10.1.0.0/16) (10.2.0.0/16) (10.3.0.0/16)
```

#### Azure ExpressRoute vs VPN Gateway
| Feature | ExpressRoute | VPN Gateway |
|---------|-------------|------------|
| Connection | Dedicated private circuit | IPSec over internet |
| Bandwidth | Up to 100 Gbps | Up to 10 Gbps |
| Latency | Consistent, low | Variable |
| SLA | 99.95% | 99.9% |
| Cost | High | Lower |
| Setup | Weeks (physical circuit) | Minutes |

#### User Defined Routes (UDR)
Override Azure's default routing, e.g., force all internet traffic through Azure Firewall:
```
Route Table: force-tunnel
Next hop type: Virtual Appliance
Next hop IP: 10.0.1.4 (Azure Firewall private IP)
Address prefix: 0.0.0.0/0
```

---

### GCP Networking

#### GCP VPC Key Differences from AWS
GCP VPCs are **global** (not regional), which is a fundamental architectural difference:
- A single VPC can have subnets in any region globally
- No need to peer VPCs just to span regions
- Subnets are regional (but the VPC is global)

#### GCP Core Components

| Component | AWS Equivalent | Notes |
|-----------|---------------|-------|
| VPC Network | VPC | Global in GCP |
| Subnet | Subnet | Regional |
| Firewall Rules | Security Groups + NACLs | Network-level, tag-based |
| VPC Network Peering | VPC Peering | Same non-transitive rules |
| Cloud Router | — | BGP routing for Cloud VPN/Interconnect |
| Cloud NAT | NAT Gateway | Regional; auto-scales |
| Cloud VPN | Site-to-Site VPN | HA VPN = 99.99% SLA |
| Cloud Interconnect | Direct Connect | Dedicated/Partner |
| Cloud Load Balancing | ALB/NLB | Global or regional |
| Cloud Armor | WAF + Shield | DDoS + L7 filtering |
| Private Service Connect | PrivateLink | Access Google/partner services privately |

#### GCP Firewall Rules
Unlike AWS where SGs are per-instance, GCP firewall rules are applied to **the network** and target VMs via:
- **Target tags:** VM has a tag like `web-server`; rule targets that tag
- **Target service accounts:** Rule targets VMs running a specific service account

```yaml
# GCP Firewall rule example
name: allow-https-from-internet
network: default
direction: INGRESS
priority: 1000
sourceRanges: ["0.0.0.0/0"]
targetTags: ["web-server"]
allowed:
  - IPProtocol: tcp
    ports: ["443"]
```

```bash
# GCP CLI: Create firewall rule
gcloud compute firewall-rules create allow-https \
  --network=prod-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --rules=tcp:443

# List firewall rules
gcloud compute firewall-rules list --format="table(name,network,direction,priority,sourceRanges,allowed)"
```

#### Cloud NAT
- Provides outbound NAT for VMs without external IPs
- Fully managed, auto-scales
- Regional; one Cloud NAT per region per VPC
- Integrated with Cloud Router

```bash
# Create Cloud NAT
gcloud compute routers create nat-router \
  --network=prod-vpc \
  --region=us-central1

gcloud compute routers nats create cloud-nat \
  --router=nat-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

#### GCP Cloud Interconnect
- **Dedicated Interconnect:** 10/100 Gbps direct link to Google PoP
- **Partner Interconnect:** Through a service provider (50 Mbps – 50 Gbps)
- **HA Interconnect:** Two connections in two metro areas for 99.99% SLA

#### GCP vs AWS vs Azure Comparison

| Feature | AWS | GCP | Azure |
|---------|-----|-----|-------|
| VPC scope | Regional | Global | Regional |
| Firewall | Security Group (instance) + NACL (subnet) | Firewall rules (network, tag-based) | NSG (subnet or NIC) |
| Stateful firewall | Yes (SG) | Yes | Yes (NSG) |
| NAT | NAT Gateway | Cloud NAT | NAT Gateway |
| Load Balancer | ALB, NLB, GLB | Cloud Load Balancing (global) | Application Gateway, Azure LB, Front Door |
| Private link | PrivateLink | Private Service Connect | Private Endpoint |
| Dedicated link | Direct Connect | Cloud Interconnect | ExpressRoute |
| VPN | Site-to-Site VPN | HA Cloud VPN | VPN Gateway |

---

## Hands-On Labs

### Lab 1: Azure Networking (Azure CLI)
```bash
# Login to Azure
# az login

# Create resource group
az group create --name NetworkLab --location eastus

# Create VNet with subnets
az network vnet create \
  --resource-group NetworkLab \
  --name prod-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name public-subnet \
  --subnet-prefix 10.0.1.0/24

az network vnet subnet create \
  --resource-group NetworkLab \
  --vnet-name prod-vnet \
  --name private-subnet \
  --address-prefix 10.0.11.0/24

# Create NSG
az network nsg create \
  --resource-group NetworkLab \
  --name web-nsg

# Add NSG rules
az network nsg rule create \
  --resource-group NetworkLab \
  --nsg-name web-nsg \
  --name AllowHTTPS \
  --priority 100 \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --access Allow

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group NetworkLab \
  --vnet-name prod-vnet \
  --name public-subnet \
  --network-security-group web-nsg

echo "Azure VNet configured"
```

### Lab 2: GCP VPC Configuration
```bash
# Authenticate
# gcloud auth login

# Create custom VPC (not auto-mode)
gcloud compute networks create prod-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Create regional subnets
gcloud compute networks subnets create public-subnet \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access

gcloud compute networks subnets create private-subnet \
  --network=prod-vpc \
  --region=us-central1 \
  --range=10.0.11.0/24 \
  --enable-private-ip-google-access

# Create firewall rules
gcloud compute firewall-rules create allow-https-web \
  --network=prod-vpc \
  --direction=INGRESS \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --rules=tcp:443,tcp:80

gcloud compute firewall-rules create allow-internal \
  --network=prod-vpc \
  --direction=INGRESS \
  --source-ranges=10.0.0.0/8 \
  --rules=all

echo "GCP VPC configured"
```

### Lab 3: Cross-Cloud Architecture Comparison Script
```python
#!/usr/bin/env python3

clouds = {
    "AWS": {
        "vpc_scope": "Regional",
        "firewall_level": "Instance (SG) + Subnet (NACL)",
        "stateful": "Yes (SG), No (NACL)",
        "nat": "NAT Gateway (managed, per-AZ)",
        "lb_options": "ALB (L7), NLB (L4), GWLB",
        "interconnect": "Direct Connect (1/10/100 Gbps)",
        "global_lb": "CloudFront + Global Accelerator",
        "private_access": "PrivateLink / VPC Endpoints",
    },
    "GCP": {
        "vpc_scope": "Global",
        "firewall_level": "Network (tag/SA based)",
        "stateful": "Yes",
        "nat": "Cloud NAT (managed, auto-scale)",
        "lb_options": "Global HTTP(S), TCP/UDP, Internal",
        "interconnect": "Cloud Interconnect (10/100 Gbps)",
        "global_lb": "Cloud Load Balancing (native global)",
        "private_access": "Private Service Connect",
    },
    "Azure": {
        "vpc_scope": "Regional (VNet)",
        "firewall_level": "Subnet or NIC (NSG)",
        "stateful": "Yes (NSG)",
        "nat": "NAT Gateway (managed)",
        "lb_options": "App Gateway (L7), Azure LB (L4)",
        "interconnect": "ExpressRoute (50 Mbps – 100 Gbps)",
        "global_lb": "Azure Front Door",
        "private_access": "Private Endpoint",
    }
}

features = list(next(iter(clouds.values())).keys())
print(f"{'Feature':<22}", end="")
for cloud in clouds:
    print(f"{cloud:<30}", end="")
print()
print("-" * 85)

for feature in features:
    print(f"{feature:<22}", end="")
    for cloud, props in clouds.items():
        val = props.get(feature, "N/A")
        print(f"{val[:28]:<30}", end="")
    print()
```

---

## Practical Exercises

### Exercise 1: NSG Rule Simulator
```python
#!/usr/bin/env python3

class AzureNSG:
    def __init__(self, name):
        self.name = name
        self.rules = []

    def add_rule(self, priority, name, port, protocol, source, action):
        self.rules.append({
            "priority": priority, "name": name, "port": port,
            "protocol": protocol, "source": source, "action": action
        })
        self.rules.sort(key=lambda r: r["priority"])

    def evaluate(self, src_ip, port, protocol):
        import ipaddress
        for rule in self.rules:
            port_match = str(rule["port"]) == str(port) or rule["port"] == "*"
            proto_match = rule["protocol"].lower() in [protocol.lower(), "*", "any"]
            try:
                src_match = ipaddress.IPv4Address(src_ip) in ipaddress.IPv4Network(rule["source"])
            except:
                src_match = rule["source"] in ["*", "Internet", src_ip]

            if port_match and proto_match and src_match:
                return rule["action"], rule["name"], rule["priority"]
        return "Deny", "Default deny", 65500

# Build NSG
nsg = AzureNSG("web-nsg")
nsg.add_rule(100, "Allow-HTTPS", 443, "TCP", "0.0.0.0/0", "Allow")
nsg.add_rule(110, "Allow-HTTP", 80, "TCP", "0.0.0.0/0", "Allow")
nsg.add_rule(200, "Allow-SSH", 22, "TCP", "10.0.0.0/8", "Allow")
nsg.add_rule(500, "Deny-SSH-Internet", 22, "TCP", "0.0.0.0/0", "Deny")

tests = [
    ("203.0.113.5", 443, "TCP"),
    ("203.0.113.5", 22, "TCP"),
    ("10.5.1.100", 22, "TCP"),
    ("203.0.113.5", 8080, "TCP"),
]

print(f"NSG: {nsg.name}")
print(f"{'Source IP':<20}{'Port':<8}{'Action':<10}{'Rule'}")
print("-" * 55)
for src, port, proto in tests:
    action, rule_name, prio = nsg.evaluate(src, port, proto)
    icon = "✓" if action == "Allow" else "✗"
    print(f"{src:<20}{port:<8}{icon} {action:<8} [{prio}] {rule_name}")
```

### Exercise 2: GCP Firewall Rule Builder
```python
#!/usr/bin/env python3

def gcp_firewall_rule(name, network, direction, source_ranges, target_tags, rules, priority=1000):
    gcloud_cmd = f"""gcloud compute firewall-rules create {name} \\
  --network={network} \\
  --direction={direction} \\
  --priority={priority} \\
  --source-ranges={','.join(source_ranges)} \\
  --target-tags={','.join(target_tags)} \\
  --rules={','.join(rules)}"""
    return gcloud_cmd

# Generate rules for a typical web app
configs = [
    ("allow-https-public", "prod-vpc", "INGRESS", ["0.0.0.0/0"], ["web-server"], ["tcp:443", "tcp:80"], 1000),
    ("allow-app-from-lb", "prod-vpc", "INGRESS", ["10.0.1.0/24"], ["app-server"], ["tcp:8080"], 1000),
    ("allow-db-from-app", "prod-vpc", "INGRESS", ["10.0.11.0/24"], ["db-server"], ["tcp:5432"], 1000),
    ("allow-ssh-iap", "prod-vpc", "INGRESS", ["35.235.240.0/20"], ["web-server", "app-server"], ["tcp:22"], 1000),
    ("deny-all-internet", "prod-vpc", "INGRESS", ["0.0.0.0/0"], ["db-server"], ["all"], 2000),
]

print("GCP Firewall Rules (gcloud commands):")
print("=" * 60)
for config in configs:
    print(gcp_firewall_rule(*config))
    print()
```

---

## Sample Exercises

1. **GCP vs AWS firewall model:** Explain the fundamental difference in how GCP and AWS apply firewall rules to VMs.
2. **Azure Hub-Spoke:** What Azure Firewall setting allows spoke VNets to use the hub's VPN Gateway to reach on-premises?
3. **Cloud NAT limits:** When would Cloud NAT (GCP) or NAT Gateway (Azure) reach SNAT port exhaustion, and how do you fix it?
4. **ExpressRoute vs Direct Connect:** You need 5 Gbps of dedicated bandwidth from an on-prem data center to Azure. What are the options?
5. **GCP global VPC advantage:** Give a concrete example where GCP's global VPC eliminates the need for something that AWS and Azure require.

## Solutions

1. **Firewall model difference:** AWS Security Groups are attached per-instance (ENI); GCP firewall rules are attached to the **VPC network** and target specific VMs via **network tags** or **service accounts**. This means a GCP rule with `target-tags: web-server` automatically applies to any new VM with that tag, regardless of which subnet or zone it's in. In AWS, you must explicitly attach a Security Group to each new instance.

2. **Hub-Spoke VPN transit:** Enable `allowGatewayTransit: true` on the hub VNet peering (hub side) and `useRemoteGateways: true` on each spoke VNet peering (spoke side). This allows spoke VNets to route through the hub's VPN Gateway to reach on-premises.

3. **SNAT port exhaustion:** Occurs when many VMs make many simultaneous outbound connections to the same destination IP, exhausting the ~64K ports per NAT IP. Fix: add more static external IP addresses to the NAT pool (GCP: `--nat-external-ip-pool`; Azure: add more IPs to NAT Gateway).

4. **ExpressRoute 5 Gbps options:** ExpressRoute doesn't offer exactly 5 Gbps. Options: (a) 10 Gbps Dedicated circuit (overprovision); (b) Partner Interconnect via a service provider that offers 5 Gbps increments; (c) Two 2.5 Gbps Partner circuits for redundancy. Use ExpressRoute FastPath for the highest performance.

5. **GCP global VPC advantage:** A company with offices in us-central1, europe-west1, and asia-east1 can use a single GCP VPC with subnets in each region. All subnets can communicate with each other using private IPs without any VPC peering, Transit Gateway, or extra routing configuration. In AWS, you'd need VPCs in each region and Transit Gateway peering between them (with additional costs and configuration).

## Completion Checklist
- [ ] Design an Azure VNet with subnets, NSG, and UDR
- [ ] Write GCP firewall rules using tags and service accounts
- [ ] Explain the GCP global VPC model vs AWS/Azure regional model
- [ ] Configure VNet peering with Gateway Transit in Azure
- [ ] Set up Cloud NAT in GCP using Cloud Router
- [ ] Compare ExpressRoute, Direct Connect, and Cloud Interconnect

## Key Takeaways
- GCP's global VPC eliminates the need for cross-region peering for private connectivity
- Azure NSGs are stateful and can be applied at subnet or NIC level — more flexible than NACLs
- GCP uses tag-based firewall rules; AWS uses per-instance Security Groups; Azure uses NSGs on subnets/NICs
- All three clouds have managed NAT services that abstract IP management
- Dedicated links (ExpressRoute, Direct Connect, Cloud Interconnect) provide consistent, low-latency hybrid connectivity

## Next Steps
Proceed to [Day 23: Hybrid & Multi-Cloud Networking](../Day_23/notes_and_exercises.md).