# Day 00: Introduction & Networking Fundamentals

## Learning Objectives
By the end of Day 0, you will:
- Understand what computer networking is and why it matters
- Learn basic networking terminology and core concepts
- Set up your learning environment and tools
- Get familiar with network topologies and communication models
- Understand the role of networking in DevOps, SRE, and Cloud Engineering

**Estimated Time:** 2-3 hours

## Notes

### What is Computer Networking?
- **Definition:** The practice of connecting computers and devices to share resources, data, and services
- **Purpose:** Enable communication between devices across different locations
- **Importance:** Foundation of modern computing, internet, cloud services, and digital communication
- **Evolution:** From simple point-to-point connections to complex global networks

### Key Networking Concepts

#### Fundamental Terms
- **Host/Node:** Any device connected to a network (computer, server, router, switch)
- **Protocol:** Set of rules governing communication between devices
- **Bandwidth:** Maximum data transfer rate of a network connection
- **Latency:** Time delay in data transmission from source to destination
- **Throughput:** Actual data transfer rate achieved in practice
- **Packet:** Unit of data transmitted over a network

#### Network Communication Models
- **Client-Server:** Centralized model where clients request services from servers
- **Peer-to-Peer (P2P):** Decentralized model where devices communicate directly
- **Hybrid:** Combination of client-server and P2P models

### Network Types by Scope

#### LAN (Local Area Network)
- **Coverage:** Small geographic area (building, campus)
- **Characteristics:** High speed, low latency, private ownership
- **Examples:** Office networks, home networks, school networks
- **Technologies:** Ethernet, Wi-Fi

#### WAN (Wide Area Network)
- **Coverage:** Large geographic area (cities, countries, continents)
- **Characteristics:** Lower speed, higher latency, often leased lines
- **Examples:** Internet, corporate networks spanning multiple locations
- **Technologies:** MPLS, Frame Relay, Internet VPN

#### MAN (Metropolitan Area Network)
- **Coverage:** City or metropolitan area
- **Characteristics:** Between LAN and WAN in scope and performance
- **Examples:** City-wide Wi-Fi, cable TV networks
- **Technologies:** Fiber optic, wireless

#### PAN (Personal Area Network)
- **Coverage:** Individual workspace (few meters)
- **Characteristics:** Very short range, low power
- **Examples:** Bluetooth devices, USB connections
- **Technologies:** Bluetooth, Zigbee, NFC

### Network Topologies

#### Physical Topologies
- **Bus:** Single communication line shared by all devices
- **Star:** Central hub with devices connected individually
- **Ring:** Devices connected in circular fashion
- **Mesh:** Multiple interconnected paths between devices
- **Hybrid:** Combination of multiple topologies

#### Logical Topologies
- **Broadcast:** All devices receive all transmissions
- **Token Passing:** Controlled access using token mechanism
- **Switched:** Direct communication paths between devices

### Networking in Modern IT Roles

#### DevOps Engineers
- **Infrastructure as Code:** Network automation and configuration management
- **CI/CD Pipelines:** Network connectivity for build and deployment systems
- **Monitoring:** Network performance and availability monitoring
- **Security:** Network security policies and implementation

#### SRE (Site Reliability Engineers)
- **Service Reliability:** Network redundancy and failover mechanisms
- **Performance:** Network optimization for application performance
- **Incident Response:** Network troubleshooting and problem resolution
- **Capacity Planning:** Network scaling and resource management

#### Cloud Engineers
- **Virtual Networks:** VPCs, subnets, and cloud networking services
- **Hybrid Connectivity:** On-premises to cloud network integration
- **Multi-Cloud:** Network connectivity across different cloud providers
- **Security:** Cloud network security and compliance

## Hands-On Labs

### Lab 1: Network Discovery and Analysis
```bash
# Check your network interfaces
ip addr show

# View routing table
ip route show

# Check network connectivity
ping -c 4 google.com

# Display network statistics
ss -tuln

# Check DNS resolution
nslookup google.com

# Trace network path
traceroute google.com
```

### Lab 2: Basic Network Tools Installation
```bash
# Update package manager
sudo apt update

# Install essential networking tools
sudo apt install -y net-tools iproute2 dnsutils traceroute \
                    tcpdump wireshark-common nmap netcat-openbsd \
                    iperf3 ethtool wireless-tools

# Verify installations
which ping traceroute nslookup ss ip
```

### Lab 3: Network Environment Setup
```bash
# Create network monitoring script
cat > network_check.sh << 'EOF'
#!/bin/bash
echo "=== Network Environment Check ==="
echo "Date: $(date)"
echo

echo "1. Network Interfaces:"
ip addr show | grep -E "^[0-9]|inet "

echo -e "\n2. Default Gateway:"
ip route | grep default

echo -e "\n3. DNS Servers:"
cat /etc/resolv.conf | grep nameserver

echo -e "\n4. Internet Connectivity:"
ping -c 2 8.8.8.8 > /dev/null && echo "✓ Internet accessible" || echo "✗ No internet"

echo -e "\n5. DNS Resolution:"
nslookup google.com > /dev/null && echo "✓ DNS working" || echo "✗ DNS issues"
EOF

chmod +x network_check.sh
./network_check.sh
```

## Practical Exercises

### Exercise 1: Network Inventory Script
```bash
#!/bin/bash
# Network device inventory

echo "Network Device Inventory"
echo "========================"

# Local network interfaces
echo "Local Interfaces:"
ip link show | grep -E "^[0-9]" | awk '{print $2}' | sed 's/://'

# Network neighbors (ARP table)
echo -e "\nNetwork Neighbors:"
ip neighbor show | head -10

# Active network connections
echo -e "\nActive Connections:"
ss -tuln | head -10

# Wireless networks (if available)
if command -v iwlist &> /dev/null; then
    echo -e "\nWireless Networks:"
    sudo iwlist scan 2>/dev/null | grep ESSID | head -5
fi
```

### Exercise 2: Network Performance Baseline
```python
#!/usr/bin/env python3
import subprocess
import time
import json

def run_command(cmd):
    """Run shell command and return output"""
    try:
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        return result.stdout.strip()
    except Exception as e:
        return f"Error: {e}"

def network_baseline():
    """Create network performance baseline"""
    baseline = {
        'timestamp': time.strftime('%Y-%m-%d %H:%M:%S'),
        'tests': {}
    }
    
    # Ping test
    ping_result = run_command('ping -c 5 8.8.8.8 | tail -1')
    baseline['tests']['ping_google'] = ping_result
    
    # DNS resolution test
    dns_start = time.time()
    run_command('nslookup google.com')
    dns_time = time.time() - dns_start
    baseline['tests']['dns_resolution_time'] = f"{dns_time:.3f}s"
    
    # Interface statistics
    interface_stats = run_command('cat /proc/net/dev | grep eth0')
    baseline['tests']['interface_stats'] = interface_stats
    
    return baseline

if __name__ == "__main__":
    baseline = network_baseline()
    print(json.dumps(baseline, indent=2))
```

### Exercise 3: Network Topology Mapper
```bash
#!/bin/bash
# Simple network topology discovery

echo "Network Topology Discovery"
echo "=========================="

# Get local network information
LOCAL_IP=$(ip route get 8.8.8.8 | awk '{print $7; exit}')
GATEWAY=$(ip route | grep default | awk '{print $3}')
NETWORK=$(ip route | grep $LOCAL_IP | grep -v default | awk '{print $1}' | head -1)

echo "Local IP: $LOCAL_IP"
echo "Gateway: $GATEWAY"
echo "Network: $NETWORK"

echo -e "\nDiscovering local devices..."

# Simple ping sweep (first 10 IPs)
NETWORK_BASE=$(echo $NETWORK | cut -d'.' -f1-3)
for i in {1..10}; do
    IP="$NETWORK_BASE.$i"
    if ping -c 1 -W 1 $IP &>/dev/null; then
        echo "✓ $IP is active"
    fi
done
```

## Sample Exercises

1. **Network Types:** Classify the following scenarios into LAN, WAN, MAN, or PAN:
   - Office computers connected via Ethernet
   - Bluetooth headphones connected to phone
   - Company branches connected via internet
   - City-wide fiber network

2. **Topology Analysis:** Draw the network topology for your home/office network including all devices and connections.

3. **Protocol Identification:** Identify which protocols are used for:
   - Web browsing
   - Email sending
   - File transfer
   - Domain name resolution

4. **Performance Measurement:** Measure and compare network performance between:
   - Wired vs wireless connection
   - Different times of day
   - Different websites/services

5. **Career Application:** Explain how networking knowledge applies to your target role (DevOps/SRE/Cloud Engineer).

## Solutions

1. **Network Types Classification:**
   - **LAN:** Office computers connected via Ethernet
   - **PAN:** Bluetooth headphones connected to phone
   - **WAN:** Company branches connected via internet
   - **MAN:** City-wide fiber network

2. **Topology Analysis Example:**
   ```
   Internet
      |
   Router/Modem
      |
   Switch/Hub
   /    |    \
  PC1  PC2  Printer
   |
  WiFi AP
     |
   Laptop
   ```

3. **Protocol Identification:**
   - **Web browsing:** HTTP/HTTPS
   - **Email sending:** SMTP
   - **File transfer:** FTP/SFTP
   - **Domain name resolution:** DNS

4. **Performance Measurement Factors:**
   - **Wired vs Wireless:** Wired typically faster and more stable
   - **Time of day:** Peak hours may show congestion
   - **Different services:** CDN locations affect performance
   - **Tools:** Use ping, traceroute, speedtest for measurements

5. **Career Applications:**
   - **DevOps:** Network automation, infrastructure monitoring, CI/CD connectivity
   - **SRE:** Service reliability, performance optimization, incident response
   - **Cloud Engineer:** Virtual networks, hybrid connectivity, multi-cloud networking

## Completion Checklist
- [ ] Understand basic networking concepts and terminology
- [ ] Know different network types and their characteristics
- [ ] Familiar with network topologies and their applications
- [ ] Have networking tools installed and tested
- [ ] Can perform basic network discovery and analysis
- [ ] Understand networking's role in modern IT careers

## Key Takeaways
- Networking is fundamental to all modern computing and IT operations
- Different network types serve different purposes and scales
- Network topology affects performance, reliability, and scalability
- Proper tools and baseline measurements are essential for network management
- Networking skills are crucial for DevOps, SRE, and Cloud Engineering roles

## Next Steps
Proceed to [Day 1: OSI Model Overview & Why It Matters](../Day_01/notes_and_exercises.md) to learn the foundational framework for understanding network communication.