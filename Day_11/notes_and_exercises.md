# Day 11: Routing Basics, Static & Dynamic Routing, Route Tables

## Learning Objectives
By the end of Day 11, you will:
- Understand how routers make forwarding decisions
- Configure static routes and default routes
- Learn dynamic routing protocols (RIP, OSPF, BGP, EIGRP)
- Read and manipulate Linux routing tables

**Estimated Time:** 3-4 hours

## Notes

### What is Routing?
- **Routing** is the process of selecting a path for traffic in a network
- Routers operate at **Layer 3 (Network Layer)**
- Each router makes **hop-by-hop** decisions based on its routing table
- The goal: get a packet from source to destination across potentially many networks

### Routing Table Components
```
Destination     Gateway         Genmask         Flags  Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG     eth0   ← Default route
192.168.1.0     0.0.0.0         255.255.255.0   U      eth0   ← Connected network
10.10.0.0       192.168.1.254   255.255.0.0     UG     eth0   ← Static/dynamic route
```

- **Destination:** Target network or host
- **Gateway:** Next-hop IP (0.0.0.0 = directly connected)
- **Genmask:** Subnet mask for the destination
- **Flags:** U=Up, G=Gateway, H=Host, R=Reject
- **Iface:** Outgoing interface

### Route Selection: Longest Prefix Match
When multiple routes match a destination, the router picks the **most specific** (longest prefix):
```
Packet to 10.1.1.50:
  Route: 10.0.0.0/8  via 1.1.1.1  ← less specific
  Route: 10.1.0.0/16 via 2.2.2.2  ← more specific
  Route: 10.1.1.0/24 via 3.3.3.3  ← most specific → CHOSEN
```

### Administrative Distance (AD)
AD determines which routing source is trusted when the same destination is learned from multiple sources:

| Routing Source | AD |
|----------------|----|
| Connected interface | 0 |
| Static route | 1 |
| EIGRP (internal) | 90 |
| OSPF | 110 |
| RIP | 120 |
| EIGRP (external) | 170 |
| BGP (external) | 20 |
| Unknown | 255 |

### Static Routing

Static routes are manually configured by an administrator.

**Advantages:**
- Predictable, no routing overhead
- Secure (no route injection from outside)
- Ideal for small networks and stub networks

**Disadvantages:**
- Does not adapt to failures
- Hard to manage at scale

#### Static Route Examples (Linux)
```bash
# Add a static route
sudo ip route add 10.0.0.0/8 via 192.168.1.1

# Add a default route (gateway of last resort)
sudo ip route add default via 192.168.1.1

# Add route via a specific interface
sudo ip route add 172.16.0.0/12 dev eth0

# Delete a route
sudo ip route del 10.0.0.0/8
```

### Dynamic Routing Protocols

#### RIP (Routing Information Protocol)
- **Type:** Distance-vector
- **Metric:** Hop count (max 15; 16 = unreachable)
- **Update interval:** Every 30 seconds (broadcast/multicast)
- **Convergence:** Slow
- **Version:** RIPv2 supports CIDR and authentication
- **Use case:** Small, simple networks

#### OSPF (Open Shortest Path First)
- **Type:** Link-state
- **Metric:** Cost (based on bandwidth: 100 Mbps / interface bandwidth)
- **Algorithm:** Dijkstra's SPF (Shortest Path First)
- **Areas:** Hierarchical (Area 0 = backbone)
- **Convergence:** Fast
- **Use case:** Enterprise networks, large ISPs

```
OSPF Hierarchy:
[Area 1] ─── [Area 0 (Backbone)] ─── [Area 2]
               ABR connects areas
```

#### EIGRP (Enhanced Interior Gateway Routing Protocol)
- **Type:** Hybrid (distance-vector with link-state characteristics)
- **Metric:** Composite (bandwidth + delay + reliability + load)
- **Algorithm:** DUAL (Diffusing Update Algorithm)
- **Convergence:** Very fast
- **Use case:** Cisco-only environments
- **AD:** 90 (internal), 170 (external)

#### BGP (Border Gateway Protocol)
- **Type:** Path-vector
- **Metric:** AS Path, MED, Local Preference, Weight
- **Usage:** Internet routing between Autonomous Systems (AS)
- **Port:** TCP 179
- **Convergence:** Slow (policy-driven)
- **Types:** iBGP (within AS), eBGP (between AS)
- **Use case:** Internet backbone, multi-homed enterprises, cloud connectivity

### Routing Protocol Comparison

| Protocol | Type | Metric | AD | Convergence | Scope |
|----------|------|--------|----|-------------|-------|
| RIP | Distance-vector | Hop count | 120 | Slow | Small |
| OSPF | Link-state | Cost | 110 | Fast | Enterprise |
| EIGRP | Hybrid | Composite | 90 | Very fast | Cisco |
| BGP | Path-vector | AS Path | 20/200 | Slow | Internet |

## Hands-On Labs

### Lab 1: Read and Modify the Linux Routing Table
```bash
# View routing table (modern)
ip route show

# View routing table (legacy)
route -n

# View routing table with details
ip route show table main

# Add a static route
sudo ip route add 192.168.100.0/24 via 192.168.1.1

# Verify it was added
ip route show | grep 192.168.100

# Check which route would be used for a destination
ip route get 8.8.8.8

# Remove the route
sudo ip route del 192.168.100.0/24

# View all routing tables
ip route show table all | head -30
```

### Lab 2: Configure Default Route and Test
```bash
# Check current default route
ip route | grep default

# Remove existing default (careful in production!)
# sudo ip route del default

# Add a default route
sudo ip route add default via 192.168.1.1 dev eth0

# Test connectivity through default route
ping -c 3 8.8.8.8
traceroute 8.8.8.8

# View gateway ARP entry
ip neighbor show | grep $(ip route | grep default | awk '{print $3}')
```

### Lab 3: Policy-Based Routing (Advanced)
```bash
# Create a second routing table
echo "200 custom_table" | sudo tee -a /etc/iproute2/rt_tables

# Add routes to the custom table
sudo ip route add default via 192.168.1.1 table custom_table

# Create a rule to use custom table for specific source
sudo ip rule add from 192.168.1.100 table custom_table

# View routing rules
ip rule show

# View custom table
ip route show table custom_table

# Clean up
sudo ip rule del from 192.168.1.100 table custom_table
sudo ip route flush table custom_table
```

## Practical Exercises

### Exercise 1: Route Table Analyzer
```python
#!/usr/bin/env python3
import subprocess
import ipaddress

def get_routes():
    """Parse Linux routing table"""
    result = subprocess.run(['ip', 'route', 'show'], capture_output=True, text=True)
    routes = []
    
    for line in result.stdout.strip().split('\n'):
        parts = line.split()
        if not parts:
            continue
        
        route = {'dest': parts[0], 'raw': line}
        
        if 'via' in parts:
            idx = parts.index('via')
            route['gateway'] = parts[idx + 1]
        else:
            route['gateway'] = 'direct'
        
        if 'dev' in parts:
            idx = parts.index('dev')
            route['interface'] = parts[idx + 1]
        
        routes.append(route)
    return routes

def find_route(destination, routes):
    """Find best matching route for destination (longest prefix match)"""
    try:
        dest_ip = ipaddress.IPv4Address(destination)
        best_match = None
        best_prefix = -1
        
        for route in routes:
            try:
                if route['dest'] == 'default':
                    network = ipaddress.IPv4Network('0.0.0.0/0')
                else:
                    network = ipaddress.IPv4Network(route['dest'], strict=False)
                
                if dest_ip in network and network.prefixlen > best_prefix:
                    best_match = route
                    best_prefix = network.prefixlen
            except:
                continue
        
        return best_match
    except Exception as e:
        return None

routes = get_routes()
print("Current Routing Table:")
print("-" * 60)
for r in routes:
    gw = r.get('gateway', 'direct')
    iface = r.get('interface', 'unknown')
    print(f"  {r['dest']:<25} via {gw:<18} dev {iface}")

print("\nRoute lookup examples:")
for dest in ['8.8.8.8', '192.168.1.50', '10.0.0.1']:
    route = find_route(dest, routes)
    if route:
        print(f"  {dest} → {route.get('gateway', 'direct')} via {route.get('interface','?')}")
    else:
        print(f"  {dest} → No route found")
```

### Exercise 2: Static Route Manager
```bash
#!/bin/bash
# Simple static route manager

add_route() {
    local dest=$1
    local gw=$2
    echo "Adding route: $dest via $gw"
    sudo ip route add $dest via $gw
    echo "✓ Route added"
}

del_route() {
    local dest=$1
    echo "Removing route: $dest"
    sudo ip route del $dest
    echo "✓ Route removed"
}

list_routes() {
    echo "Current routes:"
    ip route show | awk '{printf "  %-25s %s\n", $1, substr($0, index($0,$2))}'
}

case $1 in
    add)   add_route $2 $3 ;;
    del)   del_route $2 ;;
    list)  list_routes ;;
    *)     echo "Usage: $0 {add <dest> <gw> | del <dest> | list}" ;;
esac
```

### Exercise 3: Routing Protocol Simulator
```python
#!/usr/bin/env python3
# Simple RIP-like hop-count routing table simulator

import heapq

class Router:
    def __init__(self, name):
        self.name = name
        self.routing_table = {}  # dest -> (cost, next_hop)
        self.neighbors = {}      # neighbor_router -> link_cost
    
    def add_neighbor(self, router, cost=1):
        self.neighbors[router.name] = (router, cost)
        # Add directly connected route
        self.routing_table[router.name] = (cost, router.name)
    
    def update_from_neighbor(self, neighbor_name, neighbor_table, link_cost):
        """RIP-style distance-vector update"""
        updated = False
        for dest, (cost, _) in neighbor_table.items():
            new_cost = cost + link_cost
            if dest != self.name:
                if dest not in self.routing_table or self.routing_table[dest][0] > new_cost:
                    self.routing_table[dest] = (new_cost, neighbor_name)
                    updated = True
        return updated
    
    def print_table(self):
        print(f"\nRouter {self.name} Routing Table:")
        print(f"  {'Destination':<15} {'Cost':<8} {'Next Hop'}")
        print(f"  {'-'*35}")
        for dest, (cost, hop) in sorted(self.routing_table.items()):
            print(f"  {dest:<15} {cost:<8} {hop}")

# Build a simple network: A─B─C─D
r_a = Router("A")
r_b = Router("B")
r_c = Router("C")
r_d = Router("D")

r_a.add_neighbor(r_b, cost=1)
r_b.add_neighbor(r_a, cost=1)
r_b.add_neighbor(r_c, cost=1)
r_c.add_neighbor(r_b, cost=1)
r_c.add_neighbor(r_d, cost=1)
r_d.add_neighbor(r_c, cost=1)

# Simulate RIP convergence
print("Simulating RIP convergence (A─B─C─D topology)...")
for iteration in range(4):
    print(f"\n--- Iteration {iteration + 1} ---")
    for src, dst, cost in [
        (r_a, r_b, 1), (r_b, r_a, 1),
        (r_b, r_c, 1), (r_c, r_b, 1),
        (r_c, r_d, 1), (r_d, r_c, 1)
    ]:
        src.update_from_neighbor(dst.name, dst.routing_table, cost)

for router in [r_a, r_b, r_c, r_d]:
    router.print_table()
```

## Sample Exercises

1. **Route Selection:** Given three routes to `10.5.0.0`: one learned via OSPF, one via RIP, one static — which is preferred and why?

2. **Static Route Design:** Design a static routing scheme for a hub-and-spoke topology with one central router and 5 branch routers.

3. **OSPF Cost Calculation:** Calculate OSPF cost for a 1 Gbps and a 100 Mbps link (reference bandwidth = 100 Mbps).

4. **BGP Scenario:** You have two ISP connections. How would you use BGP to prefer ISP-A for outbound traffic while using ISP-B as backup?

5. **Routing Loop Prevention:** Explain three mechanisms used by routing protocols to prevent routing loops.

## Solutions

1. **Route Preference:**
   - Static route wins (AD = 1) over OSPF (AD = 110) and RIP (AD = 120)
   - Lower AD = more trusted

2. **Hub-and-Spoke Static Routing:**
   - Hub router: add specific /24 routes to each branch
   - Branch routers: add a single default route pointing to hub
   - Simple, predictable, and no dynamic overhead

3. **OSPF Cost:**
   - Formula: `100,000,000 / interface_bandwidth_bps`
   - 1 Gbps: 100M / 1000M = **cost 1**
   - 100 Mbps: 100M / 100M = **cost 1** (same — modern networks need higher reference bandwidth)
   - Better: set `auto-cost reference-bandwidth 10000` for 10G environments

4. **BGP Dual-ISP:**
   - **Outbound preference for ISP-A:** Set higher Local Preference for routes learned from ISP-A
   - **Failover to ISP-B:** ISP-B routes have lower Local Preference, used only when ISP-A is down
   - **Inbound preference:** Use AS Path prepending on ISP-B advertisements to make ISP-A preferred inbound

5. **Routing Loop Prevention Mechanisms:**
   - **Split Horizon:** Don't advertise a route back out the interface it was learned on (RIP)
   - **Route Poisoning:** Advertise failed routes with infinite metric immediately (RIP)
   - **TTL (Time to Live):** IP TTL decrements at each hop; packet dropped at TTL=0
   - **SPF Algorithm:** OSPF's link-state database guarantees loop-free topology by design
   - **Hold-down timers:** Ignore updates about a downed route for a period after failure

## Completion Checklist
- [ ] Can read and interpret a Linux routing table
- [ ] Can add, modify, and delete static routes
- [ ] Understand the role of Administrative Distance
- [ ] Know key differences between RIP, OSPF, EIGRP, and BGP
- [ ] Understand longest prefix match for route selection
- [ ] Can design simple static routing schemes

## Key Takeaways
- Routing tables guide packets hop-by-hop toward their destination
- Longest prefix match determines which route is used when multiple match
- Administrative Distance determines which routing source is preferred
- Static routing is simple and predictable; dynamic routing scales and adapts
- OSPF is the preferred IGP for enterprise; BGP governs inter-AS internet routing

## Next Steps
Proceed to [Day 12: Linux Networking Commands](../Day_12/notes_and_exercises.md) to master the tools every network engineer uses daily.