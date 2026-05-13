# Day 29: Real-World Scenarios & Interview Questions

## Learning Objectives
By the end of Day 29, you will:
- Solve real-world networking scenarios end-to-end
- Answer common DevOps/SRE/Cloud networking interview questions confidently
- Demonstrate systematic troubleshooting under pressure
- Articulate design decisions with trade-offs clearly

**Estimated Time:** 4-5 hours

---

## Notes

### How to Approach Networking Scenarios

Whether in an interview or real incident, always follow this structure:

```
1. CLARIFY  → Understand the problem scope fully before diving in
2. ISOLATE  → Identify which OSI layer is affected
3. HYPOTHESIZE → Form 2-3 most likely root causes
4. TEST     → Eliminate causes systematically, smallest change first
5. FIX      → Apply the minimal correct change
6. VERIFY   → Confirm resolution end-to-end
7. DOCUMENT → Record cause, fix, and prevention
```

---

## Part 1: Real-World Troubleshooting Scenarios

### Scenario 1: "Website is Down" (Classic Oncall)

**Alert:** `CRITICAL — example.com health check failing`

**Systematic investigation:**
```bash
# Step 1: Is it DNS?
dig example.com +short
nslookup example.com 8.8.8.8

# Step 2: Is the origin reachable at IP level?
ping $(dig example.com +short | head -1)

# Step 3: Is the port open?
nc -zv example.com 443
curl -v --max-time 5 https://example.com

# Step 4: Is it TLS?
echo | openssl s_client -connect example.com:443 2>/dev/null | grep -E "Verify|Cipher"

# Step 5: Is the app responding?
curl -I https://example.com
curl -s https://example.com/health

# Step 6: Is it the load balancer?
# Check ALB target group health in AWS console
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123:targetgroup/web-tg/abc

# Step 7: Is it the backend?
# SSH to a backend server and check the process
ss -tlnp | grep :8080
journalctl -u app.service -n 50
```

**Common causes and fixes:**

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| DNS NXDOMAIN | DNS record deleted/TTL expired | Restore record |
| Connection refused | App not running | Restart service |
| Connection timeout | Firewall blocking | Update security group |
| 502 Bad Gateway | Backend crashed | Fix/restart app |
| SSL error | Cert expired | Renew certificate |
| Partial failure | Some backends unhealthy | Check ALB target health |

---

### Scenario 2: EC2 in Private Subnet Cannot Reach Internet

**Symptom:** `curl: (6) Could not resolve host: packages.ubuntu.com`

```bash
# On the EC2 instance:

# Step 1: Check DNS
cat /etc/resolv.conf     # Should show 169.254.169.253 (AWS VPC DNS)
nslookup packages.ubuntu.com  # Does DNS resolve?
dig @169.254.169.253 packages.ubuntu.com  # Test VPC DNS directly

# Step 2: If DNS works, check routing
ip route show
# Should see: default via 10.0.X.1 dev eth0 (not direct to IGW)

# Step 3: Check if NAT Gateway exists and is in public subnet
aws ec2 describe-nat-gateways --filter Name=state,Values=available

# Step 4: Check route table associated with this subnet
SUBNET_ID=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/)/subnet-id)
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SUBNET_ID

# Likely problem: Route table missing 0.0.0.0/0 → nat-gateway
# Or: NAT Gateway in wrong subnet (must be in PUBLIC subnet with IGW)
# Or: NAT Gateway has no Elastic IP

# Fix: Add route
aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-xxxxxxxx
```

---

### Scenario 3: Intermittent Connection Drops (Packet Loss)

**Symptom:** Users report random disconnects, especially on long sessions

```bash
# Step 1: Run mtr for 100 cycles
mtr --report -c 100 -n 8.8.8.8

# Interpret output:
# Loss at hop N only           → that hop rate-limits ICMP (not real loss)
# Loss at hop N and all after  → real loss starting at hop N
# Loss only at last hop        → destination drops ICMP but TCP works

# Step 2: Check for NIC errors
ip -s link show eth0
# Look for: rx_errors, tx_errors, rx_dropped, rx_missed_errors

# Step 3: Check TCP retransmissions
netstat -s | grep -i retransmit
ss -ti | grep retrans

# Step 4: Check for buffer overflow (high-traffic NIC)
ethtool -S eth0 | grep -i drop
cat /proc/net/softnet_stat  # columns: total, dropped, time_squeeze

# Step 5: Check for duplex mismatch
ethtool eth0 | grep -E "Speed|Duplex"
# Half-duplex on a switch port causes collision → drop

# Common fixes:
# - Replace damaged cable/SFP
# - Fix duplex: ethtool -s eth0 duplex full speed 1000
# - Increase ring buffer: ethtool -G eth0 rx 4096 tx 4096
# - Tune IRQ affinity for high-throughput NICs
```

---

### Scenario 4: Microservices Latency Spike

**Symptom:** p99 latency on `order-service` jumped from 50ms to 2s

```bash
# Step 1: Is it network or application?
# Check if the service itself is slow or waiting on dependencies

# Step 2: Check what order-service calls
# Look at distributed traces (Jaeger, X-Ray, Zipkin)
# Identify which downstream call is slow

# Step 3: If payment-service is the slow dependency:
# From order-service pod/instance:
curl -w "\nTotal: %{time_total}s\nTTFB: %{time_starttransfer}s\n" \
  http://payment-service:8080/health

# Step 4: Check DNS resolution time
time nslookup payment-service.production.svc.cluster.local
# If slow → CoreDNS issue in Kubernetes / ndots misconfiguration

# Step 5: Check connection pool exhaustion
ss -tn | grep payment-service | wc -l  # number of connections
ss -tn state time-wait | grep payment-service | wc -l  # TIME_WAIT connections

# Step 6: Check if it's a specific AZ
# Compare latency from instances in different AZs
# Cross-AZ traffic adds ~1ms but should not cause 2s latency

# Common causes of microservice latency spikes:
# - Connection pool exhausted → new connections = 3-way handshake overhead
# - DNS resolution failing → timeout + fallback (5-30s)
# - Downstream service overloaded
# - Network congestion between AZs
# - Resource leak causing GC pauses
```

---

### Scenario 5: VPN Tunnel Flapping

**Symptom:** Site-to-site VPN disconnects every 8 hours

```bash
# Step 1: Check IKE SA lifetime
# AWS default: IKE SA = 8 hours (this IS the cause!)
# Solution: Configure DPD (Dead Peer Detection) and re-authentication

# Step 2: Check VPN logs on the on-prem device
# Cisco ASA:
#   show crypto isakmp sa
#   show crypto ipsec sa
#   debug crypto isakmp 7

# Step 3: Ensure DPD is configured
# DPD keeps the tunnel alive and re-establishes when needed
# AWS requires DPD or NAT-T keepalive from customer side

# Step 4: Check BGP session uptime
# If using BGP, the BGP session should recover within seconds
# Check BGP hold timers: should be ≤30s with 10s keepalive

# Step 5: For AWS Site-to-Site VPN:
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-xxx \
  --query 'VpnConnections[0].VgwTelemetry'

# Fix options:
# 1. Enable DPD on customer device
# 2. Lower IKE lifetime to 3600s to force more frequent renegotiation
# 3. Configure BGP with fast timers so failover to tunnel-2 is fast
# 4. Use AWS Transit Gateway which automatically handles tunnel redundancy
```

---

## Part 2: Interview Questions & Model Answers

### OSI / Fundamentals

**Q1: Explain what happens when you type "google.com" in your browser.**

**Model Answer:**
```
1. Browser checks local cache → nothing found
2. DNS resolution:
   - Check /etc/hosts → not found
   - Query local resolver (e.g., 8.8.8.8)
   - Resolver: root → .com TLD → google.com authoritative NS
   - Returns A record: 142.250.x.x, TTL cached
3. TCP 3-way handshake to 142.250.x.x:443
   SYN → SYN-ACK → ACK
4. TLS 1.3 handshake:
   ClientHello → ServerHello + Certificate → Finished (1 RTT)
5. HTTP/2 GET request sent
6. Google server processes, returns 200 OK with HTML
7. Browser parses HTML, fires parallel requests for CSS/JS/images
8. Page renders
```

---

**Q2: What is the difference between TCP and UDP? When would you use each?**

**Model Answer:**
```
TCP:
- Connection-oriented (3-way handshake)
- Reliable: acknowledgments, retransmission, ordering
- Flow control and congestion control
- Higher overhead (20-byte header)
- Use when: data integrity critical (HTTP, database, file transfer, SSH)

UDP:
- Connectionless, no handshake
- No reliability guarantees, no ordering
- No flow/congestion control — just send
- Low overhead (8-byte header)
- Use when: speed > reliability (DNS, VoIP, video streaming,
  online gaming, DHCP)
- Application handles retransmission if needed (QUIC/HTTP3 does this)
```

---

**Q3: You have a /24 network and need 5 subnets — one with 100 hosts, two with 50 hosts, two with 20 hosts. Design it using VLSM.**

**Model Answer:**
```
Start with 192.168.1.0/24:

Largest first:
100 hosts → need /25 (126 hosts) → 192.168.1.0/25   (.1 – .126)
50 hosts  → need /26 (62 hosts)  → 192.168.1.128/26  (.129 – .190)
50 hosts  → need /26 (62 hosts)  → 192.168.1.192/26  (.193 – .254)
20 hosts  → need /27 (30 hosts)  → 192.168.1.192 already used...

Wait — 192.168.1.192/26 ends at .255, leaving no room.
Correct VLSM order (largest to smallest, sequential allocation):

192.168.1.0/25   → 100-host subnet  (.1 – .126)
192.168.1.128/26 → 50-host subnet A (.129 – .190)
192.168.1.192/26 → 50-host subnet B (.193 – .254)

No room for /27s in this /24 after two /26s.
Better: use only one /26 for second 50-host:
192.168.1.0/25   → 100-host subnet  (.1 – .126)
192.168.1.128/26 → 50-host subnet A (.129 – .190)
192.168.1.192/27 → 20-host subnet A (.193 – .222)
192.168.1.224/27 → 20-host subnet B (.225 – .254)
Now need another /26 for second 50-host → can't fit

Correct resolution: use 192.168.1.0/23 as base network,
giving 512 addresses:
192.168.0.0/25  → 100 hosts
192.168.0.128/26 → 50 hosts A
192.168.0.192/26 → 50 hosts B
192.168.1.0/27  → 20 hosts A
192.168.1.32/27 → 20 hosts B
```

---

### Cloud Networking

**Q4: An EC2 in a private subnet can't reach S3. What are the possible causes?**

**Model Answer:**
```
Ordered by likelihood:

1. No VPC endpoint for S3 and no NAT Gateway
   → Check route table for 0.0.0.0/0 → nat-xxx
   → Or add S3 Gateway Endpoint (free)

2. NAT Gateway exists but route table is wrong
   → Route might point to IGW instead of NAT GW
   → Private subnet must route to NAT GW, not IGW

3. Security Group blocks outbound HTTPS (443)
   → Check outbound rules on instance SG

4. S3 bucket policy denies the IAM role
   → Check bucket policy and instance IAM role

5. VPC Endpoint policy is too restrictive
   → If endpoint exists, check endpoint policy

6. DNS resolution failing for s3.amazonaws.com
   → Check /etc/resolv.conf, VPC DNS setting (enableDnsSupport)

Diagnosis commands:
  curl https://s3.amazonaws.com  # test connectivity
  aws s3 ls  # test with IAM creds
  ip route show  # check routing
  curl -I https://s3.us-east-1.amazonaws.com  # skip DNS
```

---

**Q5: What is VPC peering and what are its limitations?**

**Model Answer:**
```
VPC Peering creates a private, low-latency connection between two VPCs
(same or different account, same or different region).

Setup:
- Requester creates peering connection → accepter accepts
- Both sides update route tables
- Security groups reference the peer VPC CIDR

Key limitations:
1. Non-transitive: A↔B and B↔C does NOT give A↔C
   → Need Transit Gateway for hub-spoke at scale
2. No overlapping CIDRs allowed
3. No edge-to-edge routing (can't route through peer to internet)
4. Max 125 peering connections per VPC (soft limit)
5. Bandwidth limited to instance type, not the peering link
6. Cross-region peering has data transfer costs

When to use peering vs TGW:
- Peering: ≤5 VPCs, simple topology, low cost
- TGW: many VPCs, shared services, hybrid connectivity, segmentation
```

---

**Q6: Explain the difference between AWS Security Groups and NACLs.**

**Model Answer:**
```
Security Groups (SG):
- Applied at: ENI/instance level
- Stateful: return traffic automatically allowed
- Rules: allow only (cannot deny)
- Evaluation: all rules evaluated, most permissive wins
- Default: deny all inbound, allow all outbound
- Can reference other SGs as source/dest

NACLs:
- Applied at: subnet level
- Stateless: must explicitly allow both directions
  (including ephemeral ports 1024-65535 for return traffic)
- Rules: allow AND deny
- Evaluation: rules evaluated in order (lowest number first); stop at first match
- Default: allow all (newly created NACLs: deny all)

Key interview insight:
"A NACL denying traffic at the subnet level takes priority
 over a Security Group allowing it — the packet is dropped
 before it ever reaches the instance."

Use together for defense in depth:
- SG for per-service micro-segmentation
- NACL for subnet-level blanket deny (block known-bad CIDRs)
```

---

**Q7: How would you design a highly available, multi-region architecture for a global web app?**

**Model Answer:**
```
Global Layer:
- Route 53 with latency-based routing + health checks
- CloudFront CDN in front of both regions
- Global Accelerator for non-HTTP (TCP/UDP)

Each Region (Active-Active):
  Public Tier (multi-AZ):
  - ALB across 3 AZs
  - Auto Scaling Group of web servers

  App Tier (private, multi-AZ):
  - Internal ALB
  - Auto Scaling Group of app servers

  Data Tier (multi-AZ):
  - Aurora Global Database (primary region)
  - Read replicas in secondary region
  - <1s replication lag

  Cross-region:
  - Aurora Global DB failover (<1 min RTO)
  - Route 53 health check → failover DNS
  - S3 Cross-Region Replication for objects

Failover:
- Route 53 detects primary unhealthy
- Shifts traffic to secondary region
- Aurora Global DB promoted to primary
- RTO: <5 minutes, RPO: <1 second
```

---

### Performance & Security

**Q8: Walk me through how you'd troubleshoot high latency for a microservice.**

**Model Answer:**
```
Start with data, not guesses.

1. Identify the layer:
   - Is it the service itself or a dependency?
   - Check distributed traces (Jaeger/X-Ray): which span is slow?

2. If network:
   - Measure RTT: ping/mtr from service to dependency
   - Check DNS: time nslookup <dep>
     (K8s ndots misconfiguration adds 5 failed lookups before resolution)
   - Check connection pool: ss -tn | grep <dep> | wc -l
   - Check TIME_WAIT: ss -tn state time-wait | wc -l
     (high TIME_WAIT → port exhaustion → new conn = slow)

3. If it's cross-AZ:
   - Verify traffic isn't taking unexpected path
   - Check if NLB/ALB is cross-zone load balancing correctly

4. Check at the application layer:
   - Connection pooling configured? (avoid new TCP per request)
   - Keep-alive headers set?
   - Timeout values appropriate?

5. Metrics to check:
   - p50/p95/p99 (outliers vs general slowdown)
   - Error rate (timeouts masking as latency)
   - Throughput (saturation?)

6. Fix → verify → monitor
```

---

**Q9: What is Zero Trust and how does it differ from perimeter security?**

**Model Answer:**
```
Perimeter security ("castle and moat"):
- Trust is based on network location
- Inside the firewall = trusted
- Problem: lateral movement once inside is unconstrained

Zero Trust ("never trust, always verify"):
Core principles (NIST SP 800-207):
1. Verify explicitly: authenticate every request using identity,
   device health, location, and behavior
2. Least privilege: JIT/JEA, minimal permissions
3. Assume breach: segment everything, encrypt all comms,
   minimize blast radius

Practical implementation:
- Identity: MFA + SSO + continuous risk assessment
- Device: posture check (patched, encrypted, MDM-enrolled)
- Network: microsegmentation (SG-to-SG, K8s NetworkPolicy)
- Application: mTLS between services (Istio, Consul Connect)
- Data: encryption at rest + in transit, field-level encryption

Example: "Even if an attacker has valid credentials,
they still can't reach the database because the app-sg
is the only SG allowed to talk to db-sg on port 5432."
```

---

**Q10: Explain BGP and its role in internet routing.**

**Model Answer:**
```
BGP (Border Gateway Protocol) is the routing protocol of the internet.
It's a path-vector protocol that routes traffic between Autonomous Systems (AS).

Key concepts:
- AS: a collection of IP prefixes under one administrative domain
  (each ISP, cloud provider, large company has an ASN)
- eBGP: between different ASes (internet routing)
- iBGP: within the same AS (internal routing)

How it works:
- ASes advertise IP prefixes (e.g., "I own 203.0.113.0/24")
- BGP peers exchange these advertisements
- AS_PATH attribute: list of ASes the route traversed
  → prevents loops; longer path = less preferred

Route selection (in order):
1. Highest LOCAL_PREF (within AS)
2. Shortest AS_PATH (fewer hops)
3. Lowest MED (metric from neighbor)
4. eBGP over iBGP
5. Lowest IGP cost to next-hop
6. Lowest router-id (tiebreaker)

Real-world uses:
- ISP peering: Comcast ↔ AT&T exchange routes
- Multihoming: company connects to 2 ISPs, BGP selects best
- Cloud: AWS Direct Connect uses BGP for route exchange
- Anycast: same prefix advertised from many locations
```

---

## Part 3: Rapid-Fire Questions

### Quick Answers Reference

| Question | Answer |
|----------|--------|
| Default subnet mask for Class C? | /24 (255.255.255.0) |
| Port for HTTPS? | 443 (TCP) |
| Port for DNS? | 53 (TCP/UDP) |
| Port for SSH? | 22 (TCP) |
| What does ARP do? | Resolves IP → MAC address |
| What is the loopback address? | 127.0.0.1 (IPv4), ::1 (IPv6) |
| How many usable hosts in a /27? | 30 (2^5 - 2) |
| What does CIDR stand for? | Classless Inter-Domain Routing |
| Difference between hub and switch? | Hub: broadcasts to all; Switch: learns MACs, forwards selectively |
| What is MTU? | Maximum Transmission Unit (1500 bytes for Ethernet) |
| What is a floating IP? | An IP that can be moved between instances instantly |
| What prevents switching loops? | Spanning Tree Protocol (STP) |
| What is NAT64? | Translates IPv6 to IPv4 for transition |
| What is ECMP? | Equal-Cost Multi-Path: load balances across multiple equal-cost routes |
| What is a BGP AS path? | List of AS numbers a route has traversed |
| What is the private IP range for class B? | 172.16.0.0 – 172.31.255.255 (/12) |
| What AWS resource costs money even when unused? | Elastic IP (when not attached) |
| What is split tunneling? | VPN only for certain traffic; rest goes direct |
| What is DNSSEC? | Cryptographic signing of DNS records to prevent spoofing |
| How many bits in IPv6? | 128 bits |

---

## Hands-On Labs

### Lab 1: Full Scenario Walkthrough Script
```bash
#!/bin/bash
# Simulated oncall scenario: "users can't reach the app"

echo "=== Oncall Scenario: Application Unreachable ==="
echo "Starting systematic investigation..."
echo ""

check() {
    local desc=$1
    local cmd=$2
    local expect=$3
    result=$(eval "$cmd" 2>/dev/null)
    if echo "$result" | grep -q "$expect" 2>/dev/null || [ "$expect" = "any" ]; then
        echo "  ✓ $desc"
        return 0
    else
        echo "  ✗ $desc → ISSUE FOUND"
        return 1
    fi
}

echo "[Layer 1] Physical / Interface"
check "Interface UP" "ip link show | grep -v lo | grep -c 'state UP'" "1"

echo ""
echo "[Layer 3] Network / Routing"
check "Default route exists" "ip route | grep -c default" "1"
GW=$(ip route | awk '/default/{print $3}' | head -1)
check "Gateway reachable ($GW)" "ping -c1 -W2 $GW" "1 received"
check "Internet reachable (8.8.8.8)" "ping -c1 -W3 8.8.8.8" "1 received"

echo ""
echo "[Layer 7] DNS / Application"
check "DNS resolving" "dig +short google.com | grep -E '^[0-9]'" "."
check "HTTPS reachable" "curl -sk --max-time 5 https://google.com -o /dev/null -w '%{http_code}'" "200"

echo ""
echo "[Security] Firewall / Ports"
echo "  Listening services:"
ss -tlnp | grep LISTEN | awk '{printf "    %s (%s)\n", $4, $6}' | head -8

echo ""
echo "=== Investigation Complete ==="
echo "Check any ✗ items above for root cause."
```

### Lab 2: Interview Q&A Practice Tool
```python
#!/usr/bin/env python3
import random

QA = [
    {
        "q": "How many usable hosts are in a /26 subnet?",
        "a": "62 hosts. A /26 has 6 host bits: 2^6 = 64 total IPs, minus network and broadcast = 62 usable."
    },
    {
        "q": "What is the difference between stateful and stateless firewalls?",
        "a": "Stateful tracks connection state — return traffic is automatically allowed. Stateless evaluates each packet independently — you must explicitly allow return traffic (e.g., ephemeral ports 1024-65535). AWS SGs are stateful; NACLs are stateless."
    },
    {
        "q": "What TCP flags are used in the 3-way handshake?",
        "a": "SYN (client→server), SYN-ACK (server→client), ACK (client→server). The connection is established after the ACK."
    },
    {
        "q": "What causes ARP poisoning and how do you prevent it?",
        "a": "An attacker sends gratuitous ARP replies mapping their MAC to a victim's IP, redirecting traffic. Prevention: Dynamic ARP Inspection (DAI) on switches, static ARP entries for critical hosts, 802.1X port authentication, network segmentation."
    },
    {
        "q": "What is the purpose of a NAT Gateway in AWS?",
        "a": "Allows instances in private subnets to initiate outbound internet connections (software updates, API calls) without having a public IP. Placed in a public subnet; private subnets route 0.0.0.0/0 to it. Fully managed, HA within an AZ — deploy one per AZ for full HA."
    },
    {
        "q": "Explain TCP slow start.",
        "a": "TCP slow start is a congestion control mechanism. A new connection begins with a small congestion window (cwnd=1 MSS) and doubles it each RTT until reaching ssthresh or detecting loss. This prevents a new connection from overwhelming the network. After slow start, TCP enters congestion avoidance (linear growth)."
    },
    {
        "q": "What is the purpose of the TTL field in IP packets?",
        "a": "Time To Live — decremented by 1 at each router hop. When it reaches 0, the packet is dropped and the router sends an ICMP Time Exceeded back to the sender. Prevents packets from circulating forever in routing loops. traceroute works by sending packets with incrementing TTL to discover each hop."
    },
    {
        "q": "What is OSPF and how does it differ from BGP?",
        "a": "OSPF is a link-state IGP (Interior Gateway Protocol) used within a single AS. It builds a complete map of the network using LSAs and runs Dijkstra's SPF to find shortest paths. BGP is an EGP (Exterior Gateway Protocol) that routes between ASes using path vectors and policy. OSPF converges faster; BGP is more scalable and policy-driven."
    },
    {
        "q": "How does CloudFront improve performance for dynamic content?",
        "a": "Two ways: (1) TCP connection termination at the nearest PoP — user's 3-way handshake only travels to the nearest edge node (5-20ms vs 200ms cross-continent); the PoP reuses a persistent TCP connection to origin. (2) Edge compute (Lambda@Edge, CF Functions) — logic runs at the PoP without round-tripping to origin at all."
    },
    {
        "q": "What is the BGP attribute used to prefer one path over another from the same AS?",
        "a": "LOCAL_PREF (Local Preference) — set by the receiving AS on iBGP routes. Higher LOCAL_PREF = preferred. Used to control which exit point to use when leaving the AS. (Inbound traffic control uses AS_PATH prepending or MED)."
    },
]

def quiz():
    print("Networking Interview Quiz")
    print("Press Enter to see the answer, 'q' to quit\n")
    random.shuffle(QA)
    score = correct = 0

    for item in QA:
        score += 1
        print(f"Q{score}: {item['q']}")
        ans = input("Your answer (press Enter for model answer): ").strip()
        print(f"\nModel Answer:\n  {item['a']}\n")
        rating = input("Did you get it right? (y/n/q): ").strip().lower()
        if rating == 'q':
            break
        if rating == 'y':
            correct += 1
        print()

    print(f"Score: {correct}/{score} ({correct/score*100:.0f}%)")

quiz()
```

### Lab 3: Scenario Decision Tree
```python
#!/usr/bin/env python3

class TroubleshootingTree:
    def __init__(self):
        self.tree = {
            "start": {
                "question": "Can you ping the default gateway?",
                "yes": "gw_ok",
                "no": "l1_l2_issue"
            },
            "l1_l2_issue": {
                "question": "Is the interface UP (ip link show)?",
                "yes": "no_gateway",
                "no": "interface_down"
            },
            "interface_down": {
                "answer": "Layer 1/2 issue. Check: cable, NIC, driver, 'ip link set eth0 up'"
            },
            "no_gateway": {
                "answer": "No route to gateway. Check: 'ip route show', correct CIDR, default route configured"
            },
            "gw_ok": {
                "question": "Can you ping 8.8.8.8 (internet IP)?",
                "yes": "internet_ip_ok",
                "no": "no_internet"
            },
            "no_internet": {
                "question": "Are you in a private subnet (no public IP)?",
                "yes": "check_nat",
                "no": "check_igw"
            },
            "check_nat": {
                "answer": "Check: NAT Gateway exists and is AVAILABLE; route table has 0.0.0.0/0 → nat-xxx (NOT igw); NAT GW is in a PUBLIC subnet"
            },
            "check_igw": {
                "answer": "Check: Internet Gateway attached to VPC; route table has 0.0.0.0/0 → igw-xxx; instance has public IP; Security Group allows outbound"
            },
            "internet_ip_ok": {
                "question": "Can you resolve DNS (nslookup google.com)?",
                "yes": "dns_ok",
                "no": "dns_broken"
            },
            "dns_broken": {
                "answer": "DNS issue. Check: /etc/resolv.conf (nameserver); VPC DNS enabled (enableDnsSupport); Security Group/NACL allows UDP 53; test 'dig @8.8.8.8 google.com'"
            },
            "dns_ok": {
                "question": "Can you reach the specific service port (nc -zv host port)?",
                "yes": "port_ok",
                "no": "port_blocked"
            },
            "port_blocked": {
                "answer": "Transport/firewall issue. Check: Security Group inbound rules; NACL rules (both directions!); application actually listening (ss -tlnp); host-based firewall (iptables -L)"
            },
            "port_ok": {
                "answer": "Network is fine. Issue is at application layer. Check: app logs, TLS cert validity, HTTP response codes, application config"
            },
        }

    def run(self):
        print("Network Troubleshooting Decision Tree")
        print("=" * 40)
        node_id = "start"

        while True:
            node = self.tree[node_id]
            if "answer" in node:
                print(f"\n✓ Diagnosis: {node['answer']}")
                break
            print(f"\n? {node['question']}")
            ans = input("  Answer (y/n/q): ").strip().lower()
            if ans == 'q':
                print("Exiting.")
                break
            elif ans == 'y':
                node_id = node["yes"]
            elif ans == 'n':
                node_id = node["no"]
            else:
                print("Please enter y or n")

TroubleshootingTree().run()
```

---

## Sample Interview Scenarios

1. **Design question:** "Design the network for a three-tier web application on AWS that must handle 10,000 concurrent users, be highly available across 3 AZs, and comply with PCI-DSS (no internet to DB tier)."

2. **Debugging question:** "Your Kubernetes pods can't communicate with each other across nodes. Everything worked yesterday. What do you check?"

3. **Calculation question:** "You have 172.20.0.0/16. Design it for: 50 office locations each needing up to 200 hosts, 10 data center segments needing up to 1000 hosts each, 5 WAN links."

4. **Conceptual question:** "Explain how HTTPS protects against a man-in-the-middle attack. What would still be vulnerable?"

5. **Architecture question:** "A company's CDN cache hit rate is 30%. What questions would you ask to improve it to 80%+?"

## Model Answers (Scenarios)

1. **3-tier AWS PCI-DSS design:**
   ```
   VPC 10.0.0.0/16 across us-east-1a/b/c:
   Public:  10.0.1-3.0/24  → ALB (HTTPS 443), WAF attached
   App:     10.0.11-13.0/24 → EC2/ECS, SG only accepts from ALB-SG
   DB:      10.0.21-23.0/24 → RDS Aurora, SG only accepts from App-SG
   
   PCI compliance:
   - No route to internet for DB subnet (no NAT GW in DB route table)
   - VPC Flow Logs enabled, stored encrypted in S3
   - All traffic encrypted (HTTPS, RDS TLS)
   - WAF on ALB (OWASP top 10)
   - Access only via Bastion/SSM (no direct SSH from internet)
   - KMS encryption for all data at rest
   ```

2. **K8s pod-to-pod debugging:**
   ```
   1. Check CNI plugin is healthy (e.g., Calico, Flannel pods running)
   2. Check pod CIDR routing: can node reach pod on other node?
      'ip route show' on both nodes
   3. Check NetworkPolicy: any policy accidentally blocking?
      'kubectl get networkpolicy -A'
   4. Check kube-proxy: is iptables/ipvs rules correct?
      'iptables -L -n | grep <service-ip>'
   5. Check MTU: CNI encapsulation (VXLAN) reduces MTU
      Pod MTU should be 50 bytes less than node MTU
   6. Check what changed: new NetworkPolicy? CNI update? Node added?
   ```

3. **172.20.0.0/16 VLSM:**
   ```
   DC segments (1000 hosts each, need /22):
   10 × /22 = 10 × 1024 IPs = 10,240 IPs
   172.20.0.0/22 through 172.20.36.0/22

   Office locations (200 hosts each, need /24):
   50 × /24 = 50 × 256 IPs = 12,800 IPs
   172.20.40.0/24 through 172.20.89.0/24

   WAN links (2 hosts each, need /30):
   5 × /30 = 5 × 4 IPs
   172.20.90.0/30 through 172.20.90.16/30

   Total used: ~23,000 of 65,536 available — fits in /16
   ```

4. **HTTPS and MITM:**
   - HTTPS prevents MITM by: server presents certificate signed by trusted CA → client verifies CN/SAN matches hostname + chain leads to trusted root → encrypted channel using server's public key.
   - Still vulnerable: if attacker has a valid cert for your domain (rogue CA, misissuance) → use CAA records and certificate transparency logs; if client trusts a rogue root CA (corporate proxy, malware-installed cert); if user ignores certificate warnings; DNS hijacking before TLS even begins (→ fix with HSTS preloading + DNSSEC).

5. **CDN cache hit rate questions:**
   ```
   Ask:
   1. What's the TTL on Cache-Control headers? (too low = frequent misses)
   2. Are query strings included in cache keys? (unique URLs = cache fragmentation)
   3. Are personalized headers (Cookie, Authorization) varying the cache?
   4. Are you caching POST/dynamic responses? (shouldn't be)
   5. What's the content mix? (HTML, images, API)
   
   Solutions:
   - Set s-maxage=86400 for static assets
   - Strip irrelevant query strings in cache key
   - Use separate origin for authenticated API (don't cache)
   - Use versioned filenames for static assets (long TTL + immediate update)
   - Review CloudFront cache behavior per path pattern
   ```

## Completion Checklist
- [ ] Walk through a "website down" incident systematically
- [ ] Solve 5 different networking scenarios using layer-by-layer approach
- [ ] Answer all 10 detailed interview questions without notes
- [ ] Complete the rapid-fire Q&A with 80%+ accuracy
- [ ] Design a multi-region HA architecture on a whiteboard
- [ ] Explain BGP, OSPF, Zero Trust, and VLSM clearly and concisely

## Key Takeaways
- Always troubleshoot layer by layer — resist the urge to jump to conclusions
- In interviews, think out loud: show your process, not just the answer
- For design questions, always discuss trade-offs (cost, complexity, availability)
- Practice subnetting math until it's instant — it comes up in nearly every interview
- "I don't know but here's how I'd find out" is a better answer than guessing
- Real-world experience is built by solving real problems — document every incident

## Next Steps
Proceed to [Day 30: Mega Project — Cloud Networking Challenge](../Day_30/notes_and_exercises.md) to consolidate everything you've learned.