# Day 00: Introduction & Networking Fundamentals

## Learning Objectives
- Understand what computer networking is and why it matters
- Learn basic networking terminology and concepts
- Set up your learning environment
- Get familiar with network topologies and basic protocols

## Theory

### What is Computer Networking?
Computer networking is the practice of connecting computers and other devices to share resources, data, and services. Networks enable communication between devices across different locations.

### Key Networking Concepts
- **Host/Node**: Any device connected to a network
- **Protocol**: Rules governing communication between devices
- **Bandwidth**: Maximum data transfer rate
- **Latency**: Time delay in data transmission
- **Throughput**: Actual data transfer rate achieved

### Network Types
- **LAN** (Local Area Network): Small geographic area
- **WAN** (Wide Area Network): Large geographic area
- **MAN** (Metropolitan Area Network): City-wide network
- **PAN** (Personal Area Network): Individual workspace

### Basic Network Topologies
- **Bus**: Single communication line
- **Star**: Central hub with connected devices
- **Ring**: Circular connection pattern
- **Mesh**: Multiple interconnected paths

## Hands-On Labs

### Lab 1: Network Discovery
```bash
# Check your network interface
ip addr show

# View routing table
ip route show

# Check network connectivity
ping google.com

# Display network statistics
ss -tuln
```

### Lab 2: Basic Network Tools
```bash
# Check DNS resolution
nslookup google.com

# Trace network path
traceroute google.com

# Monitor network traffic
netstat -i

# Check ARP table
arp -a
```

### Lab 3: Network Configuration
```bash
# View network manager status
nmcli general status

# List network connections
nmcli connection show

# Check wireless networks
nmcli device wifi list
```

## Practical Exercises

1. **Network Inventory**: Document all devices on your home/office network
2. **Speed Test**: Measure your internet connection speed and analyze results
3. **Network Mapping**: Draw a diagram of your local network topology
4. **Protocol Analysis**: Identify protocols used by different applications

## Key Takeaways
- Networks enable resource sharing and communication
- Understanding basic concepts is crucial for advanced topics
- Hands-on practice with tools builds practical skills
- Network topology affects performance and reliability

## Next Steps
Tomorrow we'll dive deep into the OSI Model and understand how network communication is structured in layers.

## Resources
- [RFC 1180 - TCP/IP Tutorial](https://tools.ietf.org/html/rfc1180)
- [Computer Networks - Tanenbaum](https://www.pearson.com/store/p/computer-networks/P100000648863)
- [Wireshark Network Analysis](https://www.wireshark.org/docs/)