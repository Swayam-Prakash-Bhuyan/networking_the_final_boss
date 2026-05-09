# Day 10: IP Addressing, Subnetting, VLSM, Supernetting

## Learning Objectives
By the end of Day 10, you will:
- Master IPv4 address classes, private ranges, and special addresses
- Perform subnetting calculations with confidence
- Apply VLSM (Variable Length Subnet Masking) for efficient address allocation
- Understand supernetting and route summarization

**Estimated Time:** 3-4 hours

## Notes

### IPv4 Address Structure Recap
- **32-bit** address split into 4 octets (bytes)
- Each octet ranges from 0–255
- Written in dotted-decimal: `192.168.1.1`
- Divided into **Network** portion and **Host** portion by the subnet mask

### IPv4 Address Classes

| Class | First Octet | Default Mask | Network Bits | Host Bits | Usable Hosts |
|-------|-------------|--------------|--------------|-----------|--------------|
| A | 1–126 | /8 (255.0.0.0) | 8 | 24 | 16,777,214 |
| B | 128–191 | /16 (255.255.0.0) | 16 | 16 | 65,534 |
| C | 192–223 | /24 (255.255.255.0) | 24 | 8 | 254 |
| D | 224–239 | Multicast | — | — | — |
| E | 240–255 | Reserved/Research | — | — | — |

### Special and Private Address Ranges

| Range | Purpose |
|-------|---------|
| 10.0.0.0/8 | Private (Class A) |
| 172.16.0.0/12 | Private (Class B) |
| 192.168.0.0/16 | Private (Class C) |
| 127.0.0.0/8 | Loopback |
| 169.254.0.0/16 | APIPA (link-local) |
| 0.0.0.0/8 | This network |
| 255.255.255.255 | Limited broadcast |
| 224.0.0.0/4 | Multicast |

### Subnetting Fundamentals

#### The Subnet Mask
- Marks which bits are **network** (1s) and which are **host** (0s)
- Example: `/24` = `255.255.255.0` = `11111111.11111111.11111111.00000000`

#### Key Formulas
```
Number of subnets  = 2^(borrowed bits)
Hosts per subnet   = 2^(host bits) - 2
Block size         = 256 - (last non-zero octet of mask)
```

#### Subnetting Example: `192.168.1.0/26`
```
Network:        192.168.1.0
Subnet mask:    255.255.255.192  (/26)
Block size:     256 - 192 = 64
Subnets:        2^2 = 4
Hosts/subnet:   2^6 - 2 = 62

Subnet 1:  192.168.1.0   → 192.168.1.63   (hosts: .1 – .62)
Subnet 2:  192.168.1.64  → 192.168.1.127  (hosts: .65 – .126)
Subnet 3:  192.168.1.128 → 192.168.1.191  (hosts: .129 – .190)
Subnet 4:  192.168.1.192 → 192.168.1.255  (hosts: .193 – .254)
```

### CIDR Notation Reference Table

| CIDR | Mask | Hosts | Block |
|------|------|-------|-------|
| /24 | 255.255.255.0 | 254 | 256 |
| /25 | 255.255.255.128 | 126 | 128 |
| /26 | 255.255.255.192 | 62 | 64 |
| /27 | 255.255.255.224 | 30 | 32 |
| /28 | 255.255.255.240 | 14 | 16 |
| /29 | 255.255.255.248 | 6 | 8 |
| /30 | 255.255.255.252 | 2 | 4 |
| /16 | 255.255.0.0 | 65,534 | — |
| /8  | 255.0.0.0 | 16,777,214 | — |

### VLSM (Variable Length Subnet Masking)

VLSM allows **different-sized subnets** from the same address space — unlike fixed-length subnetting.

#### VLSM Design Process
1. **Sort requirements** from largest to smallest
2. **Assign the smallest subnet** that fits each requirement
3. **Allocate sequentially** to avoid overlap

#### VLSM Example: `10.0.0.0/24`
```
Requirements: 100 hosts, 50 hosts, 20 hosts, 5 hosts, 2 point-to-point links

100 hosts → /25 (126 hosts) → 10.0.0.0/25    (10.0.0.1 – 10.0.0.126)
50 hosts  → /26 (62 hosts)  → 10.0.0.128/26  (10.0.0.129 – 10.0.0.190)
20 hosts  → /27 (30 hosts)  → 10.0.0.192/27  (10.0.0.193 – 10.0.0.222)
5 hosts   → /29 (6 hosts)   → 10.0.0.224/29  (10.0.0.225 – 10.0.0.230)
P2P link  → /30 (2 hosts)   → 10.0.0.232/30  (10.0.0.233 – 10.0.0.234)
P2P link  → /30 (2 hosts)   → 10.0.0.236/30  (10.0.0.237 – 10.0.0.238)
```

### Supernetting and Route Summarization

Supernetting (route summarization) **combines multiple smaller networks** into a single larger advertisement — reducing routing table size.

#### Supernetting Example
```
Networks to summarize:
192.168.0.0/24
192.168.1.0/24
192.168.2.0/24
192.168.3.0/24

Binary:
192.168.00000000.0
192.168.00000001.0
192.168.00000010.0
192.168.00000011.0
         ^^^^^^^^ — last 2 bits vary → shared 22 bits

Summary route: 192.168.0.0/22  (covers all 4 networks)
```

#### When to Summarize
- Reduces routing table entries
- Speeds up routing decisions
- Requires **contiguous** address blocks
- Common at network boundaries (edge routers, ISP handoffs)

## Hands-On Labs

### Lab 1: Subnet Calculation Practice
```bash
# Install ipcalc
sudo apt install ipcalc -y

# Calculate subnet details
ipcalc 192.168.10.0/26
ipcalc 10.0.0.0/22
ipcalc 172.16.0.0/20

# Split a network into subnets
ipcalc 192.168.1.0/24 --split 4 2>/dev/null || \
  echo "Splitting: 4 subnets of /26 each from 192.168.1.0/24"
```

### Lab 2: VLSM Planning Script
```bash
#!/bin/bash
# VLSM subnet planner

echo "VLSM Subnet Allocation"
echo "Base Network: 10.0.0.0/24"
echo "======================="

# Display allocations
declare -A subnets=(
    ["Engineering (100 hosts)"]="10.0.0.0/25"
    ["Sales (50 hosts)"]="10.0.0.128/26"
    ["HR (20 hosts)"]="10.0.0.192/27"
    ["Management (5 hosts)"]="10.0.0.224/29"
    ["Point-to-Point A"]="10.0.0.232/30"
    ["Point-to-Point B"]="10.0.0.236/30"
)

for dept in "${!subnets[@]}"; do
    subnet=${subnets[$dept]}
    echo "$dept → $subnet"
    ipcalc $subnet 2>/dev/null | grep -E "Network|Broadcast|Hosts" | head -3
    echo "---"
done
```

### Lab 3: Route Summarization Tool
```python
#!/usr/bin/env python3
import ipaddress

def summarize_networks(networks):
    """Attempt to summarize a list of networks into the smallest supernet"""
    net_objects = [ipaddress.IPv4Network(n) for n in networks]
    
    try:
        summary = list(ipaddress.collapse_addresses(net_objects))
        return summary
    except Exception as e:
        return [f"Error: {e}"]

# Example
networks = [
    "192.168.0.0/24",
    "192.168.1.0/24",
    "192.168.2.0/24",
    "192.168.3.0/24"
]

print("Networks to summarize:")
for n in networks:
    print(f"  {n}")

print("\nSummary routes:")
for s in summarize_networks(networks):
    print(f"  {s}")
    net = ipaddress.IPv4Network(str(s))
    print(f"    Hosts: {net.num_addresses - 2}")
    print(f"    Range: {net.network_address} – {net.broadcast_address}")
```

## Practical Exercises

### Exercise 1: Subnet Calculator
```python
#!/usr/bin/env python3
import ipaddress

def subnet_info(cidr):
    """Display full subnet information"""
    net = ipaddress.IPv4Network(cidr, strict=False)
    hosts = list(net.hosts())
    
    print(f"Network:        {net.network_address}")
    print(f"Broadcast:      {net.broadcast_address}")
    print(f"Subnet Mask:    {net.netmask}")
    print(f"CIDR Prefix:    /{net.prefixlen}")
    print(f"Usable Hosts:   {len(hosts)}")
    if hosts:
        print(f"First Host:     {hosts[0]}")
        print(f"Last Host:      {hosts[-1]}")
    print(f"Total IPs:      {net.num_addresses}")

# Test it
for network in ["192.168.1.0/24", "10.0.0.0/8", "172.16.100.0/22"]:
    print(f"\n=== {network} ===")
    subnet_info(network)
```

### Exercise 2: VLSM Allocator
```python
#!/usr/bin/env python3
import ipaddress
import math

def vlsm_allocate(base_network, requirements):
    """Allocate subnets using VLSM"""
    base = ipaddress.IPv4Network(base_network)
    current = base.network_address
    allocations = []
    
    # Sort largest first
    sorted_reqs = sorted(requirements, key=lambda x: x[1], reverse=True)
    
    for name, hosts_needed in sorted_reqs:
        # Calculate prefix length needed
        prefix = 32 - math.ceil(math.log2(hosts_needed + 2))
        prefix = max(prefix, 1)
        
        # Find next aligned block
        subnet = ipaddress.IPv4Network(f"{current}/{prefix}", strict=False)
        
        # Align to block boundary
        if subnet.network_address != current:
            current = subnet.broadcast_address + 1
            subnet = ipaddress.IPv4Network(f"{current}/{prefix}", strict=False)
        
        allocations.append((name, subnet, hosts_needed))
        current = subnet.broadcast_address + 1
    
    return allocations

requirements = [
    ("Engineering", 100),
    ("Sales", 50),
    ("HR", 20),
    ("Management", 5),
    ("WAN Link A", 2),
]

print("VLSM Allocation from 10.10.0.0/24")
print("=" * 55)
results = vlsm_allocate("10.10.0.0/24", requirements)

for dept, subnet, needed in results:
    hosts = list(subnet.hosts())
    print(f"\n{dept} (need {needed} hosts)")
    print(f"  Subnet:    {subnet}")
    print(f"  Usable:    {len(hosts)} hosts")
    if hosts:
        print(f"  Range:     {hosts[0]} – {hosts[-1]}")
```

### Exercise 3: IP Address Validator
```bash
#!/bin/bash
# Validate and classify IP addresses

classify_ip() {
    local ip=$1
    IFS='.' read -r a b c d <<< "$ip"
    
    # Validate format
    if ! [[ "$a" =~ ^[0-9]+$ && "$b" =~ ^[0-9]+$ && \
            "$c" =~ ^[0-9]+$ && "$d" =~ ^[0-9]+$ ]]; then
        echo "$ip: Invalid format"
        return
    fi
    
    # Classify
    if [ $a -eq 127 ]; then
        echo "$ip: Loopback"
    elif [ $a -eq 10 ]; then
        echo "$ip: Private (Class A)"
    elif [ $a -eq 172 ] && [ $b -ge 16 ] && [ $b -le 31 ]; then
        echo "$ip: Private (Class B)"
    elif [ $a -eq 192 ] && [ $b -eq 168 ]; then
        echo "$ip: Private (Class C)"
    elif [ $a -eq 169 ] && [ $b -eq 254 ]; then
        echo "$ip: APIPA / Link-Local"
    elif [ $a -ge 224 ] && [ $a -le 239 ]; then
        echo "$ip: Multicast"
    elif [ $a -ge 1 ] && [ $a -le 126 ]; then
        echo "$ip: Public Class A"
    elif [ $a -ge 128 ] && [ $a -le 191 ]; then
        echo "$ip: Public Class B"
    elif [ $a -ge 192 ] && [ $a -le 223 ]; then
        echo "$ip: Public Class C"
    else
        echo "$ip: Special/Reserved"
    fi
}

for ip in "10.0.0.1" "192.168.1.100" "172.20.5.5" "8.8.8.8" "127.0.0.1" "169.254.1.10" "224.0.0.1"; do
    classify_ip $ip
done
```

## Sample Exercises

1. **Subnetting:** Subnet `172.16.0.0/16` into subnets of at least 1000 hosts each. How many subnets can you create?

2. **VLSM Design:** You have `192.168.5.0/24`. Allocate for: 120 users, 60 users, 28 users, 12 users, 2 WAN links.

3. **Supernetting:** Summarize `10.1.4.0/24`, `10.1.5.0/24`, `10.1.6.0/24`, `10.1.7.0/24` into a single route.

4. **Troubleshooting:** A host with IP `192.168.1.50/27` cannot communicate with `192.168.1.100`. Why?

5. **IP Classification:** Identify whether each is public, private, or special: `172.31.255.254`, `100.64.0.1`, `203.0.113.5`, `192.0.2.1`.

## Solutions

1. **Subnetting 172.16.0.0/16 for 1000+ hosts:**
   - Need /22 per subnet (1022 hosts)
   - /16 to /22 = 6 borrowed bits → 64 subnets
   - Each subnet: 172.16.0.0/22, 172.16.4.0/22, 172.16.8.0/22 …

2. **VLSM from 192.168.5.0/24:**
   - 120 hosts → /25 → 192.168.5.0/25
   - 60 hosts → /26 → 192.168.5.128/26
   - 28 hosts → /27 → 192.168.5.192/27
   - 12 hosts → /28 → 192.168.5.224/28
   - WAN link → /30 → 192.168.5.240/30
   - WAN link → /30 → 192.168.5.244/30

3. **Supernetting 10.1.4.0–10.1.7.0:**
   - 4 networks, all /24 → borrow 2 bits back → /22
   - Summary: `10.1.4.0/22`

4. **Communication failure:**
   - `/27` has 32 IPs, block size 32
   - 192.168.1.50 is in subnet 192.168.1.32/27 (range .33–.62)
   - 192.168.1.100 is in subnet 192.168.1.96/27 (range .97–.126)
   - They are in **different subnets** — need a router to communicate

5. **IP Classification:**
   - `172.31.255.254` → Private (172.16.0.0/12)
   - `100.64.0.1` → Shared Address Space (CGNAT, RFC 6598)
   - `203.0.113.5` → Documentation/TEST-NET-3 (not routable)
   - `192.0.2.1` → Documentation/TEST-NET-1 (not routable)

## Completion Checklist
- [ ] Know IPv4 classes and their default masks
- [ ] Can identify private, public, and special addresses
- [ ] Can subnet a network given host or subnet count requirements
- [ ] Understand and apply VLSM for efficient address allocation
- [ ] Can summarize routes using supernetting
- [ ] Can troubleshoot IP addressing conflicts and subnet mismatches

## Key Takeaways
- Subnetting divides networks efficiently; VLSM makes it flexible
- Always allocate largest subnets first in VLSM design
- Supernetting reduces routing table size by combining contiguous networks
- Understanding binary math behind IP addressing is essential for accuracy
- Proper address planning prevents wasted IPs and routing problems

## Next Steps
Proceed to [Day 11: Routing Basics, Static & Dynamic Routing, Route Tables](../Day_11/notes_and_exercises.md) to learn how packets find their way across networks.