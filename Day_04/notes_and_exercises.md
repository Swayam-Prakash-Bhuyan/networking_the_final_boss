# Day 04: Layer 3 - Network (IPv4/IPv6, Routing, Subnetting, CIDR, NAT)

## Learning Objectives
By the end of Day 4, you will:
- Master IPv4 and IPv6 addressing schemes and calculations
- Understand subnetting, CIDR notation, and VLSM concepts
- Learn routing fundamentals and dynamic routing protocols
- Explore NAT types and address translation mechanisms
- Apply Layer 3 troubleshooting and optimization techniques

**Estimated Time:** 3-4 hours

## Notes

### IP Addressing Fundamentals

#### IPv4 Address Structure
- **Format:** 32-bit address divided into 4 octets (192.168.1.1)
- **Classes:** A (1-126), B (128-191), C (192-223), D (224-239), E (240-255)
- **Private Ranges:** 
  - Class A: 10.0.0.0/8 (16.7M addresses)
  - Class B: 172.16.0.0/12 (1M addresses)  
  - Class C: 192.168.0.0/16 (65K addresses)
- **Special Addresses:** 127.0.0.1 (loopback), 169.254.0.0/16 (APIPA)

#### IPv6 Address Structure
- **Format:** 128-bit address in 8 groups of 4 hexadecimal digits
- **Full:** 2001:0db8:85a3:0000:0000:8a2e:0370:7334
- **Compressed:** 2001:db8:85a3::8a2e:370:7334
- **Types:** Unicast (global, link-local), Multicast, Anycast
- **Special:** ::1 (loopback), fe80::/10 (link-local), ff00::/8 (multicast)

### Subnetting and CIDR

#### Subnet Mask Concepts
- **Purpose:** Separate network and host portions of IP address
- **Format:** Dotted decimal (255.255.255.0) or CIDR (/24)
- **Calculation:** Network = IP AND Subnet Mask

#### CIDR (Classless Inter-Domain Routing)
```
Example: 192.168.1.0/24
- Network bits: 24
- Host bits: 32 - 24 = 8
- Subnet mask: 255.255.255.0
- Network address: 192.168.1.0
- Broadcast address: 192.168.1.255
- Usable hosts: 2^8 - 2 = 254
- Host range: 192.168.1.1 - 192.168.1.254
```

#### VLSM (Variable Length Subnet Masking)
Allows different subnet sizes within the same major network:
```
192.168.1.0/24 divided into:
- 192.168.1.0/26   (62 hosts) - Sales
- 192.168.1.64/26  (62 hosts) - Engineering  
- 192.168.1.128/27 (30 hosts) - Management
- 192.168.1.160/27 (30 hosts) - Servers
- 192.168.1.192/28 (14 hosts) - Printers
```

### Routing Fundamentals

#### Routing Table Components
- **Destination Network:** Target network address and mask
- **Next Hop:** IP address of next router in path
- **Interface:** Outgoing interface for the route
- **Metric:** Cost or preference value for route selection
- **Administrative Distance:** Trustworthiness of routing source

#### Routing Types
- **Static Routing:** Manually configured routes
- **Dynamic Routing:** Automatically learned and updated routes
- **Default Route:** 0.0.0.0/0 (catch-all for unknown destinations)

#### Dynamic Routing Protocols
- **RIP (Routing Information Protocol):** Distance vector, hop count metric, max 15 hops
- **OSPF (Open Shortest Path First):** Link state, cost-based metric, hierarchical
- **BGP (Border Gateway Protocol):** Path vector, policy-based, Internet routing
- **EIGRP (Enhanced Interior Gateway Routing Protocol):** Hybrid, Cisco proprietary

### NAT (Network Address Translation)

#### NAT Types
- **Static NAT:** One-to-one mapping between private and public IP
- **Dynamic NAT:** Pool of public IPs assigned dynamically
- **PAT (Port Address Translation):** Many-to-one with port mapping (most common)
- **NAT64:** IPv6 to IPv4 translation for dual-stack networks

#### NAT Process (PAT Example)
1. Internal host (192.168.1.100:1234) sends packet to 8.8.8.8:53
2. NAT device translates to public IP (203.0.113.1:5678)
3. Creates translation table entry
4. Return traffic translated back to internal address

## Hands-On Labs

### Lab 1: IP Address Configuration and Analysis
```bash
# View current IP configuration
ip addr show

# Add IP address to interface
sudo ip addr add 192.168.100.10/24 dev eth0

# View routing table
ip route show

# Add static route
sudo ip route add 10.0.0.0/8 via 192.168.1.1

# Test connectivity
ping -c 4 8.8.8.8

# Trace route path
traceroute google.com
```

### Lab 2: Subnetting Practice
```bash
# Install ipcalc for subnet calculations
sudo apt install ipcalc

# Calculate subnet information
ipcalc 192.168.1.0/24
ipcalc 10.0.0.0/16
ipcalc 172.16.0.0/20

# Subnet a network into smaller subnets
ipcalc 192.168.1.0/24 --split 4
```

### Lab 3: IPv6 Configuration
```bash
# Enable IPv6 (if disabled)
echo 0 | sudo tee /proc/sys/net/ipv6/conf/all/disable_ipv6

# Add IPv6 address
sudo ip -6 addr add 2001:db8::1/64 dev eth0

# View IPv6 addresses
ip -6 addr show

# IPv6 routing table
ip -6 route show

# Test IPv6 connectivity
ping6 -c 4 2001:4860:4860::8888
```

## Practical Exercises

### Exercise 1: Subnetting Calculator Script
```python
#!/usr/bin/env python3
import ipaddress

def subnet_calculator(network, new_prefix):
    """Calculate subnets from a given network"""
    net = ipaddress.IPv4Network(network, strict=False)
    subnets = list(net.subnets(new_prefix=new_prefix))
    
    print(f"Original network: {net}")
    print(f"Subnetting into /{new_prefix} networks:")
    print("-" * 50)
    
    for i, subnet in enumerate(subnets[:8]):  # Show first 8 subnets
        hosts = list(subnet.hosts())
        print(f"Subnet {i+1}: {subnet}")
        print(f"  Network: {subnet.network_address}")
        print(f"  Broadcast: {subnet.broadcast_address}")
        print(f"  First Host: {hosts[0] if hosts else 'N/A'}")
        print(f"  Last Host: {hosts[-1] if hosts else 'N/A'}")
        print(f"  Total Hosts: {len(hosts)}")
        print()

# Example usage
subnet_calculator('192.168.1.0/24', 26)
```

### Exercise 2: Route Monitoring Script
```bash
#!/bin/bash
# Monitor routing table changes

LOGFILE="/var/log/route_monitor.log"

echo "Starting route monitoring..."
echo "$(date): Route monitoring started" >> $LOGFILE

while true; do
    CURRENT_ROUTES=$(ip route show | sort)
    
    if [ -f /tmp/previous_routes ]; then
        DIFF=$(diff /tmp/previous_routes <(echo "$CURRENT_ROUTES"))
        if [ ! -z "$DIFF" ]; then
            echo "$(date): Routing table changed" >> $LOGFILE
            echo "$DIFF" >> $LOGFILE
            echo "Route change detected at $(date)"
        fi
    fi
    
    echo "$CURRENT_ROUTES" > /tmp/previous_routes
    sleep 30
done
```

### Exercise 3: NAT Configuration with iptables
```bash
#!/bin/bash
# Configure NAT/PAT with iptables

# Enable IP forwarding
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Configure masquerading (PAT)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Allow forwarding from internal to external
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# Allow established connections back
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT

# View NAT table
sudo iptables -t nat -L -n -v

# Monitor active connections
sudo conntrack -L | head -10
```

## Sample Exercises

1. **Subnet Design:** Design a subnet scheme for a company with 4 departments needing 50, 25, 10, and 5 hosts respectively from 192.168.1.0/24.

2. **Routing Analysis:** Explain the difference between static and dynamic routing, and when to use each.

3. **IPv6 Migration:** Plan an IPv6 deployment strategy for a dual-stack network environment.

4. **NAT Troubleshooting:** A user behind NAT cannot access a specific external service. What could be the issues?

5. **VLSM Calculation:** Subnet 172.16.0.0/16 to accommodate networks of 1000, 500, 250, and 100 hosts.

## Solutions

1. **Subnet Design Solution:**
   ```
   Department requirements: 50, 25, 10, 5 hosts
   Start with largest subnet first:
   
   Subnet 1: 192.168.1.0/26   (62 hosts) - 50 users
   Subnet 2: 192.168.1.64/27  (30 hosts) - 25 users  
   Subnet 3: 192.168.1.96/28  (14 hosts) - 10 users
   Subnet 4: 192.168.1.112/29 (6 hosts)  - 5 users
   
   Remaining: 192.168.1.120/29 through 192.168.1.248/29 for future use
   ```

2. **Static vs Dynamic Routing:**
   - **Static Routing:**
     - Manual configuration, predictable paths
     - Best for: Small networks, security-sensitive environments
     - Pros: Full control, no bandwidth overhead, secure
     - Cons: No automatic failover, manual updates required
   - **Dynamic Routing:**
     - Automatic route learning and updates
     - Best for: Large networks, redundant topologies
     - Pros: Automatic failover, scalable, adaptive
     - Cons: Bandwidth overhead, complexity, potential security risks

3. **IPv6 Migration Strategy:**
   - **Phase 1:** Dual-stack implementation (IPv4 + IPv6)
   - **Phase 2:** IPv6 address planning and allocation
   - **Phase 3:** Application and service IPv6 enablement
   - **Phase 4:** Gradual IPv4 dependency reduction
   - **Considerations:** DNS (AAAA records), firewall rules, monitoring tools

4. **NAT Troubleshooting Issues:**
   - **Port blocking:** Service uses non-standard ports blocked by NAT
   - **Connection limits:** NAT device connection table full
   - **Protocol issues:** Some protocols don't work well with NAT (FTP, SIP)
   - **Firewall rules:** Additional filtering beyond NAT
   - **Application-specific:** Apps requiring inbound connections

5. **VLSM Calculation (172.16.0.0/16):**
   ```
   Requirements: 1000, 500, 250, 100 hosts
   
   1000 hosts: Need /22 (1022 hosts) = 172.16.0.0/22
   500 hosts:  Need /23 (510 hosts)  = 172.16.4.0/23  
   250 hosts:  Need /24 (254 hosts)  = 172.16.6.0/24
   100 hosts:  Need /25 (126 hosts)  = 172.16.7.0/25
   ```

## Completion Checklist
- [ ] Understand IPv4 and IPv6 addressing and calculations
- [ ] Can perform subnetting and VLSM calculations
- [ ] Know different routing types and when to use them
- [ ] Understand NAT types and configuration
- [ ] Can troubleshoot Layer 3 connectivity issues
- [ ] Familiar with routing protocols and their characteristics

## Key Takeaways
- Layer 3 provides end-to-end packet delivery across multiple networks
- Proper subnetting optimizes address space and network performance
- Routing enables communication between different network segments
- NAT allows private networks to access public internet resources
- IPv6 adoption is essential for future network scalability

## Next Steps
Proceed to [Day 5: Layer 4 - Transport](../Day_05/notes_and_exercises.md) to learn about TCP, UDP, ports, sockets, and transport layer troubleshooting.