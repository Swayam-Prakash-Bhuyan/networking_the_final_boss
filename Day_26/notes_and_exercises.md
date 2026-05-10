# Day 26: Zero Trust, Security Groups, Microsegmentation, SASE

## Learning Objectives
By the end of Day 26, you will:
- Understand Zero Trust Network Access (ZTNA) principles and architecture
- Implement microsegmentation using Security Groups and network policies
- Explore SASE (Secure Access Service Edge) and its components
- Apply Zero Trust to cloud and hybrid environments

**Estimated Time:** 3-4 hours

---

## Notes

### The Old Perimeter Model vs Zero Trust

**Traditional perimeter ("castle and moat"):**
```
Outside: Untrusted (internet)
─────────────── Firewall ───────────────
Inside: Fully trusted (LAN)
"If you're inside, you can access everything"
```
Problem: Once an attacker breaches the perimeter, they have free movement inside.

**Zero Trust Model:**
```
"Never trust, always verify"
Every request must be authenticated, authorized, and encrypted
Regardless of whether it comes from inside or outside the network
```

### Zero Trust Principles (NIST SP 800-207)

1. **Verify explicitly** — Authenticate and authorize every request using all available data (identity, location, device health, service, workload)
2. **Use least-privilege access** — Limit user access with just-in-time and just-enough-access (JIT/JEA)
3. **Assume breach** — Minimize blast radius, segment access, verify end-to-end encryption

### Zero Trust Architecture Components

```
┌─────────────────────────────────────────────────────────┐
│                   Control Plane                         │
│  ┌───────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │  Policy   │  │   Identity   │  │  Device/Posture │  │
│  │  Engine   │  │   Provider   │  │   Assessment    │  │
│  └───────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ↓ enforce
┌─────────────────────────────────────────────────────────┐
│                    Data Plane                           │
│  User/Device → [Policy Enforcement Point] → Resource   │
│                    (PEP/Proxy)                          │
└─────────────────────────────────────────────────────────┘
```

**Key components:**
- **Policy Engine (PE):** Evaluates access requests against policy
- **Policy Administrator (PA):** Grants/revokes access tokens
- **Policy Enforcement Point (PEP):** The gateway that allows/denies connections
- **Identity Provider (IdP):** Okta, Azure AD, Google Workspace
- **Device posture:** Is the device compliant? Encrypted? Patched?

### ZTNA vs VPN

| Feature | Traditional VPN | ZTNA |
|---------|----------------|------|
| Access model | Network-level (full tunnel) | Application-level |
| Trust model | Trust once connected | Verify every request |
| Lateral movement risk | High | Low |
| Performance | Hairpin through datacenter | Direct-to-app |
| Visibility | Limited | Full identity + context |
| Examples | OpenVPN, Cisco AnyConnect | Cloudflare Access, Zscaler, BeyondCorp |

### Microsegmentation

Microsegmentation creates **fine-grained security zones** down to the workload or process level, preventing lateral movement inside the network.

#### Flat Network (No Segmentation) — Dangerous
```
All servers on 10.0.0.0/24 — any server can reach any other server
→ Compromised web server can directly attack the database
```

#### Microsegmentation — Safe
```
web-tier-sg: can only send to app-tier-sg on port 8080
app-tier-sg: can only send to db-tier-sg on port 5432
db-tier-sg: can only receive from app-tier-sg on port 5432
→ Compromised web server CANNOT reach the database
```

#### Implementation Approaches

**AWS — Security Groups as microsegmentation:**
```hcl
# web can only talk to app on 8080
resource "aws_security_group_rule" "web_to_app" {
  type                     = "egress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.app.id
  security_group_id        = aws_security_group.web.id
}

# app can only accept from web on 8080
resource "aws_security_group_rule" "app_from_web" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.web.id
  security_group_id        = aws_security_group.app.id
}
```

**Kubernetes — NetworkPolicy:**
```yaml
# Only allow payment pods to reach database pods on port 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-ingress-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: payment
      ports:
        - protocol: TCP
          port: 5432
```

**Consul Intentions (service mesh):**
```bash
# Allow: payment → database
consul intention create payment database

# Deny: everything else → database
consul intention create -deny "*" database

# Allow: frontend → payment (L7 with path)
consul intention create \
  -action allow \
  -source frontend \
  -destination payment
```

### SASE (Secure Access Service Edge)

SASE (pronounced "sassy") is a cloud-delivered architecture that converges **networking** and **security** into a unified cloud service.

#### SASE Components
```
SASE = SD-WAN + SSE (Security Service Edge)

SSE includes:
  ├── ZTNA (Zero Trust Network Access)
  ├── SWG (Secure Web Gateway) — web filtering, SSL inspection
  ├── CASB (Cloud Access Security Broker) — SaaS security
  └── FWaaS (Firewall as a Service) — cloud-delivered firewall
```

#### Traditional vs SASE Architecture

**Traditional:**
```
Branch → MPLS → HQ Data Center → Internet (all traffic backhauled)
```

**SASE:**
```
Branch → SASE PoP (nearest) → Internet/Cloud
User anywhere → SASE PoP → Application
(Direct-to-cloud, no backhauling)
```

**SASE Vendors:** Cloudflare One, Zscaler, Palo Alto Prisma SASE, Cisco Umbrella, Netskope

### Implementing Zero Trust in AWS

#### Identity-based access with IAM
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::production-data/*",
    "Condition": {
      "StringEquals": {
        "aws:RequestedRegion": "us-east-1"
      },
      "Bool": {
        "aws:MultiFactorAuthPresent": "true"
      },
      "IpAddress": {
        "aws:SourceIp": ["10.0.0.0/8"]
      }
    }
  }]
}
```

#### VPC Lattice (AWS Zero Trust for services)
AWS VPC Lattice enables zero trust connectivity between services across VPCs and accounts:
```
Service A (VPC-1, Account-A)
    → Lattice Service Network
        → Service B (VPC-2, Account-B)

Authentication: IAM auth policy on each service
Authorization: IAM resource policy
Encryption: TLS end-to-end
```

#### AWS Verified Access (ZTNA for enterprise)
```
Remote User → Verified Access Endpoint
               ↓
          Trust Provider (AWS IAM Identity Center, Okta)
               ↓
          Device Posture (AWS Verified Access Trust Provider)
               ↓
          Application (internal EC2, ECS, ALB)
```

---

## Hands-On Labs

### Lab 1: Microsegmentation Audit
```bash
#!/bin/bash
# Audit current network segmentation

echo "Microsegmentation Audit"
echo "======================="

echo -e "\n[Firewall Rules (iptables)]"
sudo iptables -L FORWARD -n -v 2>/dev/null | head -15

echo -e "\n[Open Ports (listening services)]"
ss -tlnp | grep LISTEN | awk '{print "  Service:", $4, "PID:", $6}'

echo -e "\n[Active connections by destination]"
ss -tn state established | awk 'NR>1 {print $5}' | awk -F: '{print $1}' | \
  sort | uniq -c | sort -rn | head -10

echo -e "\n[Segmentation check — can we reach common risky ports?]"
for port in 3306 5432 6379 27017 9200 2379; do
    result=$(timeout 1 bash -c "echo >/dev/tcp/localhost/$port" 2>/dev/null && echo "OPEN" || echo "closed")
    echo "  localhost:$port — $result"
done
```

### Lab 2: Zero Trust Policy Simulator
```python
#!/usr/bin/env python3
import json
from datetime import datetime

class ZeroTrustPolicyEngine:
    def __init__(self):
        self.policies = []

    def add_policy(self, name, conditions, action):
        self.policies.append({"name": name, "conditions": conditions, "action": action})

    def evaluate(self, request):
        """Evaluate a request against all policies"""
        results = []
        for policy in self.policies:
            match = all(
                request.get(k) == v
                for k, v in policy["conditions"].items()
            )
            if match:
                results.append((policy["name"], policy["action"]))

        if not results:
            return "DENY", "no-matching-policy"

        # Last matching policy wins
        name, action = results[-1]
        return action, name

# Define Zero Trust policies
engine = ZeroTrustPolicyEngine()

engine.add_policy(
    "block-unverified-mfa",
    {"mfa_verified": False},
    "DENY"
)
engine.add_policy(
    "block-unhealthy-device",
    {"device_compliant": False},
    "DENY"
)
engine.add_policy(
    "allow-internal-verified",
    {"mfa_verified": True, "device_compliant": True, "network": "corporate"},
    "ALLOW"
)
engine.add_policy(
    "allow-remote-verified",
    {"mfa_verified": True, "device_compliant": True, "network": "remote"},
    "ALLOW"
)

# Test requests
requests = [
    {"user": "alice", "mfa_verified": True,  "device_compliant": True,  "network": "corporate", "resource": "payroll"},
    {"user": "bob",   "mfa_verified": False, "device_compliant": True,  "network": "remote",    "resource": "payroll"},
    {"user": "carol", "mfa_verified": True,  "device_compliant": False, "network": "corporate", "resource": "wiki"},
    {"user": "dave",  "mfa_verified": True,  "device_compliant": True,  "network": "remote",    "resource": "code"},
    {"user": "eve",   "mfa_verified": True,  "device_compliant": True,  "network": "unknown",   "resource": "database"},
]

print("Zero Trust Policy Evaluation")
print("=" * 65)
print(f"{'User':<8}{'MFA':>5}{'Device':>8}{'Network':>12}{'Resource':>12}  {'Result'}")
print("-" * 65)

for req in requests:
    action, policy = engine.evaluate(req)
    icon = "✓ ALLOW" if action == "ALLOW" else "✗ DENY"
    print(f"{req['user']:<8}{str(req['mfa_verified']):>5}{str(req['device_compliant']):>8}"
          f"{req['network']:>12}{req['resource']:>12}  {icon} ({policy})")
```

### Lab 3: Network Policy Generator for Kubernetes
```python
#!/usr/bin/env python3
import yaml

def generate_network_policy(name, namespace, pod_selector, ingress_rules, egress_rules=None):
    policy = {
        "apiVersion": "networking.k8s.io/v1",
        "kind": "NetworkPolicy",
        "metadata": {"name": name, "namespace": namespace},
        "spec": {
            "podSelector": {"matchLabels": pod_selector},
            "policyTypes": ["Ingress"] + (["Egress"] if egress_rules else []),
            "ingress": ingress_rules,
        }
    }
    if egress_rules:
        policy["spec"]["egress"] = egress_rules
    return policy

# 3-tier app microsegmentation
policies = [
    generate_network_policy(
        name="web-ingress",
        namespace="production",
        pod_selector={"role": "web"},
        ingress_rules=[{
            "from": [{"ipBlock": {"cidr": "0.0.0.0/0"}}],
            "ports": [{"protocol": "TCP", "port": 443}, {"protocol": "TCP", "port": 80}]
        }]
    ),
    generate_network_policy(
        name="app-ingress",
        namespace="production",
        pod_selector={"role": "app"},
        ingress_rules=[{
            "from": [{"podSelector": {"matchLabels": {"role": "web"}}}],
            "ports": [{"protocol": "TCP", "port": 8080}]
        }]
    ),
    generate_network_policy(
        name="db-ingress",
        namespace="production",
        pod_selector={"role": "database"},
        ingress_rules=[{
            "from": [{"podSelector": {"matchLabels": {"role": "app"}}}],
            "ports": [{"protocol": "TCP", "port": 5432}]
        }]
    ),
    generate_network_policy(
        name="deny-all-egress-db",
        namespace="production",
        pod_selector={"role": "database"},
        ingress_rules=[],
        egress_rules=[]  # deny all egress from DB pods
    ),
]

print("# Kubernetes NetworkPolicy — 3-Tier Microsegmentation")
print("# Apply with: kubectl apply -f network-policies.yml")
print()
for policy in policies:
    print(yaml.dump(policy, default_flow_style=False))
    print("---")
```

---

## Sample Exercises

1. **Zero Trust vs VPN:** A company uses site-to-site VPN. An attacker compromises a contractor's laptop that is VPN-connected. With Zero Trust, how would this scenario differ?
2. **Microsegmentation design:** Design a Security Group policy for: ALB → Web (port 443), Web → App (port 8080), App → Cache (port 6379), App → DB (port 5432). No other traffic allowed between tiers.
3. **SASE components:** A remote employee accesses a SaaS app (Salesforce) and an internal app (ERP). Which SASE components inspect each flow?
4. **Network policy:** Write a Kubernetes NetworkPolicy that allows only pods with label `role: monitoring` to access pods with label `role: api` on port 9090 (Prometheus scraping).
5. **Blast radius:** Explain "minimise blast radius" in Zero Trust terms. If a web server is compromised in a Zero Trust environment, what can the attacker do vs a flat network?

## Solutions

1. **Zero Trust vs VPN breach scenario:** With traditional VPN, the compromised laptop has full network access to everything the VPN route table permits — it can scan and attack internal resources. With ZTNA, access is per-application, per-session, and evaluated continuously. Even on the VPN, the laptop fails device posture checks → access is revoked immediately. Even if it passes initial checks, it can only reach the specific apps explicitly permitted for that user's identity, not the entire network.

2. **Security Group policy:**
   ```
   alb-sg:  Inbound 443 from 0.0.0.0/0
   web-sg:  Inbound 443 from alb-sg; Egress 8080 to app-sg
   app-sg:  Inbound 8080 from web-sg; Egress 6379 to cache-sg; Egress 5432 to db-sg
   cache-sg: Inbound 6379 from app-sg only
   db-sg:   Inbound 5432 from app-sg only
   All outbound default deny except explicitly listed
   ```

3. **SASE components per flow:**
   - Salesforce (SaaS): **CASB** (SaaS security policy, DLP, shadow IT detection) + **SWG** (web filtering, SSL inspection) + **ZTNA** (identity verification)
   - Internal ERP: **ZTNA** (application-level access, no network tunnel) + **FWaaS** (traffic inspection)

4. **Kubernetes NetworkPolicy for Prometheus:**
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-prometheus-scrape
   spec:
     podSelector:
       matchLabels:
         role: api
     policyTypes: [Ingress]
     ingress:
       - from:
           - podSelector:
               matchLabels:
                 role: monitoring
         ports:
           - protocol: TCP
             port: 9090
   ```

5. **Blast radius in Zero Trust:** In a flat network, a compromised web server can scan and attack: the database server (no firewall between them), the cache, other web servers, admin interfaces, internal APIs — everything on the same network. In Zero Trust with microsegmentation: the compromised web server can only reach the app server on port 8080. The attacker cannot directly reach the database, cache, or other tiers. The blast radius is limited to the web tier and the single port to the app tier, which also has authentication.

## Completion Checklist
- [ ] Explain Zero Trust principles (verify explicitly, least privilege, assume breach)
- [ ] Compare ZTNA with traditional VPN
- [ ] Implement microsegmentation with Security Groups and K8s NetworkPolicy
- [ ] Describe SASE components and their functions
- [ ] Evaluate access requests using a policy engine
- [ ] Design a Zero Trust architecture for a hybrid cloud environment

## Key Takeaways
- Zero Trust eliminates implicit trust — every request is authenticated and authorized
- Microsegmentation prevents lateral movement even after a breach
- Security Groups in AWS are a practical microsegmentation tool when used with SG-to-SG references
- Kubernetes NetworkPolicy enables pod-level microsegmentation
- SASE converges networking (SD-WAN) and security (ZTNA, SWG, CASB, FWaaS) into a cloud service
- "Assume breach" mindset drives architecture decisions that limit blast radius

## Next Steps
Proceed to [Day 27: CDN, Anycast, Global Load Balancing, CloudFront](../Day_27/notes_and_exercises.md).