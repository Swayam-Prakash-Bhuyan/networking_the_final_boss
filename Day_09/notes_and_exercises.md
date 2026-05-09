# Day 09: TCP/IP Model & OSI Comparison

## Learning Objectives
By the end of Day 9, you will:
- Understand the TCP/IP model layers and their responsibilities
- Compare TCP/IP and OSI models in depth
- Map real-world protocols to both models
- Apply this knowledge to practical troubleshooting and design

**Estimated Time:** 2-3 hours

## Notes

### The TCP/IP Model Overview
- **Origin:** Developed by DARPA in the 1970s; the foundation of the modern internet
- **Layers:** 4 layers (vs OSI's 7)
- **Purpose:** A practical, implementation-focused model
- **Adoption:** Used in all modern networking and internet communications

### TCP/IP Model Layers

#### Layer 4: Application Layer
- **Combines:** OSI Layers 5, 6, and 7 (Session, Presentation, Application)
- **Protocols:** HTTP, HTTPS, FTP, SSH, DNS, SMTP, DHCP, SNMP, Telnet
- **Responsibilities:** User interaction, data formatting, session management, encryption

#### Layer 3: Transport Layer
- **Equivalent to:** OSI Layer 4 (Transport)
- **Protocols:** TCP, UDP
- **Responsibilities:** End-to-end delivery, segmentation, flow control, error recovery

#### Layer 2: Internet Layer
- **Equivalent to:** OSI Layer 3 (Network)
- **Protocols:** IP (IPv4/IPv6), ICMP, OSPF, BGP, ARP (sometimes placed here)
- **Responsibilities:** Logical addressing, routing, packet forwarding

#### Layer 1: Network Access Layer (Link Layer)
- **Combines:** OSI Layers 1 and 2 (Physical + Data Link)
- **Protocols:** Ethernet, Wi-Fi (802.11), PPP, ARP, Frame Relay
- **Responsibilities:** Physical transmission, framing, MAC addressing, error detection

### OSI vs TCP/IP: Side-by-Side Comparison

```
OSI Model                    TCP/IP Model
┌─────────────────────┐      ┌─────────────────────────┐
│  Layer 7: Application│      │                         │
├─────────────────────┤      │  Layer 4: Application   │
│  Layer 6: Presentation│     │  (HTTP, FTP, DNS, SMTP) │
├─────────────────────┤      │                         │
│  Layer 5: Session   │      └─────────────────────────┘
├─────────────────────┤      ┌─────────────────────────┐
│  Layer 4: Transport │ ───> │  Layer 3: Transport     │
│  (TCP, UDP)         │      │  (TCP, UDP)             │
├─────────────────────┤      └─────────────────────────┘
│  Layer 3: Network   │ ───> ┌─────────────────────────┐
│  (IP, ICMP)         │      │  Layer 2: Internet      │
├─────────────────────┤      │  (IP, ICMP, OSPF, BGP)  │
│  Layer 2: Data Link │      └─────────────────────────┘
├─────────────────────┤      ┌─────────────────────────┐
│  Layer 1: Physical  │ ───> │  Layer 1: Network Access│
└─────────────────────┘      │  (Ethernet, Wi-Fi, PPP) │
                              └─────────────────────────┘
```

### Key Differences

| Feature | OSI Model | TCP/IP Model |
|---------|-----------|--------------|
| Layers | 7 | 4 |
| Development | Theoretical / ISO | Practical / DARPA |
| Protocol Independence | Protocol-agnostic | Protocol-specific |
| Session & Presentation | Separate layers | Merged into Application |
| Physical & Data Link | Separate layers | Merged into Network Access |
| Usage | Reference/teaching | Real-world implementation |
| Adoption | Framework/Conceptual | Internet standard |

### Data Encapsulation in TCP/IP
```
Application Layer:   [Data]
Transport Layer:     [TCP/UDP Header | Data]         → Segment / Datagram
Internet Layer:      [IP Header | TCP Header | Data] → Packet
Network Access:      [Frame Header | IP | TCP | Data | Frame Trailer] → Frame
Physical:            101010110001... (bits)
```

### Protocol Mapping Reference

| OSI Layer | TCP/IP Layer | Example Protocols |
|-----------|--------------|-------------------|
| 7 - Application | Application | HTTP, FTP, DNS, SMTP, SNMP |
| 6 - Presentation | Application | SSL/TLS, JPEG, MIME |
| 5 - Session | Application | NetBIOS, RPC |
| 4 - Transport | Transport | TCP, UDP |
| 3 - Network | Internet | IPv4, IPv6, ICMP, OSPF, BGP |
| 2 - Data Link | Network Access | Ethernet, Wi-Fi, PPP, ARP |
| 1 - Physical | Network Access | Cables, NIC, Hubs, Repeaters |

### Why Both Models Matter
- **OSI:** Excellent for troubleshooting, learning, and vendor-neutral design
- **TCP/IP:** Actual implementation used on the internet and enterprise networks
- **Together:** OSI gives you the framework; TCP/IP gives you the practice

## Hands-On Labs

### Lab 1: Protocol Identification Across Models
```bash
# Capture traffic and identify protocols by layer
sudo tcpdump -i any -c 30 -v 2>/dev/null | head -50

# See ARP (Network Access Layer)
sudo tcpdump -i any arp -c 5

# See ICMP (Internet Layer)
ping -c 3 8.8.8.8

# See TCP (Transport Layer)
curl -s -o /dev/null http://httpbin.org/get

# See DNS (Application Layer)
dig google.com +short
```

### Lab 2: Trace Data Through TCP/IP Layers
```bash
# Full packet trace - Application → Network Access
sudo tcpdump -i any -n -v port 80 &
curl -s http://httpbin.org/get > /dev/null
sudo pkill tcpdump

# View Internet layer (IP routing)
ip route show

# View Network Access layer (ARP / MAC)
ip neighbor show

# View Transport layer (TCP sockets)
ss -tn state established
```

### Lab 3: Model-Based Troubleshooting Script
```bash
#!/bin/bash
# TCP/IP Layer troubleshooting tool

TARGET=${1:-google.com}
echo "=== TCP/IP Model Troubleshooting for: $TARGET ==="

echo -e "\n[Layer 1 - Network Access] Physical/Link Check:"
for iface in $(ls /sys/class/net | grep -v lo); do
    state=$(cat /sys/class/net/$iface/operstate 2>/dev/null)
    echo "  Interface $iface: $state"
done

echo -e "\n[Layer 2 - Internet] IP Routing Check:"
if ping -c 1 -W 2 8.8.8.8 >/dev/null 2>&1; then
    echo "  ✓ Internet layer reachable (8.8.8.8)"
else
    echo "  ✗ Internet layer unreachable"
fi

echo -e "\n[Layer 3 - Transport] TCP Port 443 Check:"
if timeout 3 bash -c "echo >/dev/tcp/$TARGET/443" 2>/dev/null; then
    echo "  ✓ TCP port 443 open on $TARGET"
else
    echo "  ✗ TCP port 443 not reachable on $TARGET"
fi

echo -e "\n[Layer 4 - Application] HTTP/DNS Check:"
if nslookup $TARGET >/dev/null 2>&1; then
    echo "  ✓ DNS resolution working for $TARGET"
else
    echo "  ✗ DNS resolution failed for $TARGET"
fi

if curl -s -I https://$TARGET >/dev/null 2>&1; then
    echo "  ✓ HTTP/HTTPS application layer working"
else
    echo "  ✗ Application layer check failed"
fi
```

## Practical Exercises

### Exercise 1: Protocol Layer Mapper
```python
#!/usr/bin/env python3
# Map protocols to both OSI and TCP/IP models

protocol_map = {
    "HTTP":     {"osi": 7, "tcpip": "Application"},
    "HTTPS":    {"osi": 7, "tcpip": "Application"},
    "FTP":      {"osi": 7, "tcpip": "Application"},
    "DNS":      {"osi": 7, "tcpip": "Application"},
    "SMTP":     {"osi": 7, "tcpip": "Application"},
    "SSH":      {"osi": 7, "tcpip": "Application"},
    "SSL/TLS":  {"osi": 6, "tcpip": "Application"},
    "NetBIOS":  {"osi": 5, "tcpip": "Application"},
    "TCP":      {"osi": 4, "tcpip": "Transport"},
    "UDP":      {"osi": 4, "tcpip": "Transport"},
    "IPv4":     {"osi": 3, "tcpip": "Internet"},
    "IPv6":     {"osi": 3, "tcpip": "Internet"},
    "ICMP":     {"osi": 3, "tcpip": "Internet"},
    "OSPF":     {"osi": 3, "tcpip": "Internet"},
    "BGP":      {"osi": 3, "tcpip": "Internet"},
    "Ethernet": {"osi": 2, "tcpip": "Network Access"},
    "Wi-Fi":    {"osi": 2, "tcpip": "Network Access"},
    "ARP":      {"osi": 2, "tcpip": "Network Access"},
    "Cables":   {"osi": 1, "tcpip": "Network Access"},
}

print(f"{'Protocol':<12} {'OSI Layer':<12} {'TCP/IP Layer'}")
print("-" * 40)
for proto, info in protocol_map.items():
    print(f"{proto:<12} Layer {info['osi']:<7} {info['tcpip']}")
```

### Exercise 2: Packet Anatomy Visualizer
```bash
#!/bin/bash
# Visualize a packet's structure across TCP/IP layers

echo "Packet Anatomy (HTTP GET Request)"
echo "==================================="
echo ""
echo "[ Application Layer ]"
echo "  GET /index.html HTTP/1.1"
echo "  Host: www.example.com"
echo ""
echo "[ Transport Layer - TCP Header Added ]"
echo "  Src Port: 52341  Dst Port: 80"
echo "  Seq: 1000  Ack: 0  Flags: SYN"
echo ""
echo "[ Internet Layer - IP Header Added ]"
echo "  Src IP: 192.168.1.100  Dst IP: 93.184.216.34"
echo "  TTL: 64  Protocol: TCP"
echo ""
echo "[ Network Access Layer - Ethernet Frame ]"
echo "  Src MAC: AA:BB:CC:DD:EE:FF"
echo "  Dst MAC: 00:11:22:33:44:55"
echo "  Type: IPv4 (0x0800)"
echo ""
echo "[ Physical ] → 101010110001010010101... (bits on wire)"
```

### Exercise 3: Model Comparison Report
```python
#!/usr/bin/env python3
# Generate a comparison summary

comparison = [
    ("Layers", "7", "4"),
    ("Top Layer", "Application (L7)", "Application (L4)"),
    ("Bottom Layer", "Physical (L1)", "Network Access (L1)"),
    ("Session Mgmt", "Layer 5 (Session)", "Merged into Application"),
    ("Encryption", "Layer 6 (Presentation)", "Merged into Application"),
    ("Routing", "Layer 3 (Network)", "Layer 2 (Internet)"),
    ("Frame/MAC", "Layer 2 (Data Link)", "Layer 1 (Network Access)"),
    ("Cables/Signals","Layer 1 (Physical)","Layer 1 (Network Access)"),
    ("Purpose", "Conceptual Reference", "Practical Internet Standard"),
]

print(f"{'Feature':<22} {'OSI Model':<28} {'TCP/IP Model'}")
print("=" * 75)
for feature, osi, tcpip in comparison:
    print(f"{feature:<22} {osi:<28} {tcpip}")
```

## Sample Exercises

1. **Layer Mapping:** For each protocol below, identify both the OSI layer and TCP/IP layer: DHCP, BGP, PPP, IMAP, ARP, ICMP.

2. **Encapsulation Trace:** Describe step by step what happens to data at each TCP/IP layer when a client uploads a file via SFTP.

3. **Model Selection:** When would you prefer to use the OSI model vs the TCP/IP model for troubleshooting?

4. **Protocol Design:** You are designing a new real-time chat protocol. Which TCP/IP layer does the core logic belong to? What supporting protocols would you use at other layers?

5. **Troubleshooting Scenario:** A user cannot load any websites but CAN ping the default gateway. At which TCP/IP layer is the problem? What are the possible causes?

## Solutions

1. **Layer Mapping:**
   - **DHCP:** OSI L7 / TCP/IP Application
   - **BGP:** OSI L3 / TCP/IP Internet
   - **PPP:** OSI L2 / TCP/IP Network Access
   - **IMAP:** OSI L7 / TCP/IP Application
   - **ARP:** OSI L2 / TCP/IP Network Access
   - **ICMP:** OSI L3 / TCP/IP Internet

2. **SFTP Upload Encapsulation:**
   - **Application:** SFTP command and file data prepared
   - **Transport:** TCP segments with source/destination ports
   - **Internet:** IP packets with source/destination IPs
   - **Network Access:** Ethernet frames with MAC addresses → bits on wire

3. **Model Selection:**
   - **OSI:** Preferred for systematic troubleshooting (layer-by-layer), teaching, and vendor-neutral documentation
   - **TCP/IP:** Preferred when working directly with protocols, writing code, or configuring real network devices

4. **Real-time Chat Protocol:**
   - **Core Logic:** Application Layer (TCP/IP) — chat messages, user presence
   - **Transport:** TCP or UDP (WebSockets use TCP; QUIC uses UDP-based)
   - **Internet:** IP for routing between users
   - **Network Access:** Ethernet/Wi-Fi for local delivery

5. **Troubleshooting Scenario:**
   - **Can ping gateway:** Network Access and Internet layers are working
   - **Cannot load websites:** Problem is at Internet layer (DNS) or Application layer
   - **Likely causes:** DNS server failure, HTTP proxy misconfiguration, firewall blocking port 80/443, or application-level authentication issue

## Completion Checklist
- [ ] Understand the 4 layers of the TCP/IP model
- [ ] Can map all major protocols to both OSI and TCP/IP models
- [ ] Understand data encapsulation in the TCP/IP stack
- [ ] Know the key differences between OSI and TCP/IP
- [ ] Can apply model knowledge to troubleshoot real-world issues

## Key Takeaways
- The TCP/IP model is the practical standard; OSI is the conceptual reference
- TCP/IP collapses 7 OSI layers into 4 by merging Session+Presentation+Application and Physical+DataLink
- Understanding both models makes you a more effective troubleshooter
- Real-world protocols always map to specific layers in both models
- Data is encapsulated at each layer as it travels down the stack and de-encapsulated going up

## Next Steps
Proceed to [Day 10: IP Addressing, Subnetting, VLSM, Supernetting](../Day_10/notes_and_exercises.md) for a deep dive into IP math and address planning.