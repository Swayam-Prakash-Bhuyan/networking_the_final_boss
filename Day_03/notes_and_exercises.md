# Day 03: Layer 2 - Data Link (Ethernet, VLANs, MAC, Switching, ARP)

## Learning Objectives
By the end of Day 3, you will:
- Master Ethernet frame structure and operations
- Understand MAC addressing and Address Resolution Protocol (ARP)
- Learn VLAN concepts, configuration, and benefits
- Explore switching fundamentals and Spanning Tree Protocol
- Apply Layer 2 troubleshooting techniques

**Estimated Time:** 3-4 hours

## Notes

### Data Link Layer Functions
- **Framing:** Organize bits from physical layer into frames
- **Error Detection:** Use checksums (CRC) to detect transmission errors
- **Flow Control:** Manage data transmission rate between sender and receiver
- **Access Control:** Coordinate access to shared media (CSMA/CD, CSMA/CA)
- **Addressing:** Use MAC addresses for local network communication

### Ethernet Fundamentals

#### Ethernet Frame Structure (IEEE 802.3)
```
| Preamble | SFD | Dest MAC | Src MAC | Type/Length | Data | FCS |
|    7     |  1  |    6     |    6    |      2      |46-1500| 4  |
```
- **Preamble:** 7 bytes of alternating 1s and 0s for synchronization
- **SFD (Start Frame Delimiter):** 1 byte (10101011) marks frame start
- **Destination MAC:** 6-byte hardware address of receiving device
- **Source MAC:** 6-byte hardware address of sending device
- **Type/Length:** Protocol type (>1536) or frame length (≤1500)
- **Data:** 46-1500 bytes of payload data
- **FCS (Frame Check Sequence):** 4-byte CRC for error detection

#### Ethernet Standards Evolution
- **10BASE-T:** 10 Mbps over twisted pair (Cat3)
- **100BASE-TX:** 100 Mbps Fast Ethernet (Cat5)
- **1000BASE-T:** 1 Gbps Gigabit Ethernet (Cat5e)
- **10GBASE-T:** 10 Gbps over twisted pair (Cat6a)
- **25/40/100 Gbps:** High-speed data center standards

### MAC Addressing

#### MAC Address Format
- **Length:** 48 bits (6 bytes) expressed as 12 hexadecimal digits
- **Format:** XX:XX:XX:XX:XX:XX or XX-XX-XX-XX-XX-XX
- **OUI (Organizationally Unique Identifier):** First 3 bytes assigned by IEEE
- **NIC Specific:** Last 3 bytes assigned by manufacturer
- **Example:** 00:1B:44:11:3A:B7

#### Special MAC Addresses
- **Broadcast:** FF:FF:FF:FF:FF:FF (all devices on local network)
- **Multicast:** First bit of first byte = 1 (01:XX:XX:XX:XX:XX)
- **Unicast:** First bit of first byte = 0 (normal device addresses)
- **Locally Administered:** Second bit of first byte = 1

### ARP (Address Resolution Protocol)

#### ARP Purpose and Process
1. **Need:** Host knows destination IP but needs MAC address for local delivery
2. **ARP Request:** Broadcast "Who has IP 192.168.1.100?" (ARP opcode 1)
3. **ARP Reply:** Target responds "192.168.1.100 is at MAC 00:1B:44:11:3A:B7" (ARP opcode 2)
4. **Cache Update:** Requesting host stores IP-to-MAC mapping in ARP table
5. **Communication:** Host can now send frames directly to target MAC

#### ARP Message Format
```
| Hardware Type | Protocol Type | HW Len | Proto Len | Opcode |
|       2       |       2       |   1    |     1     |   2    |
| Sender HW Addr | Sender Proto Addr | Target HW Addr | Target Proto Addr |
|       6        |        4          |       6        |        4          |
```

#### ARP Table Management
- **Dynamic Entries:** Learned through ARP requests/replies (timeout ~20 minutes)
- **Static Entries:** Manually configured permanent mappings
- **Gratuitous ARP:** Host announces its own IP-to-MAC mapping
- **Proxy ARP:** Router responds to ARP requests on behalf of other devices

### VLANs (Virtual Local Area Networks)

#### VLAN Benefits
- **Broadcast Domain Separation:** Reduce broadcast traffic and improve performance
- **Security:** Logical segmentation isolates traffic between departments
- **Flexibility:** Software-defined network segments without physical rewiring
- **Cost Reduction:** Single switch can support multiple logical networks

#### VLAN Types
- **Data VLAN:** Carries user-generated traffic
- **Management VLAN:** Used for switch management and monitoring
- **Native VLAN:** Handles untagged traffic on trunk ports (default VLAN 1)
- **Voice VLAN:** Dedicated for VoIP traffic with QoS prioritization

#### 802.1Q VLAN Tagging
```
Original Frame: | Dest MAC | Src MAC | Type | Data | FCS |
Tagged Frame:   | Dest MAC | Src MAC | 802.1Q Tag | Type | Data | FCS |

802.1Q Tag (4 bytes):
| TPID | PCP | DEI | VID |
|  2   |  3  |  1  | 12  |
```
- **TPID (Tag Protocol ID):** 0x8100 identifies 802.1Q frame
- **PCP (Priority Code Point):** 3-bit QoS priority (0-7)
- **DEI (Drop Eligible Indicator):** 1-bit congestion notification
- **VID (VLAN ID):** 12-bit VLAN identifier (1-4094, 0 and 4095 reserved)

### Switching Fundamentals

#### Switch Operations
1. **Learning:** Build MAC address table by examining source addresses
2. **Flooding:** Forward frames to all ports when destination unknown
3. **Filtering:** Drop frames destined for same port they arrived on
4. **Forwarding:** Send frames to specific port based on MAC address table
5. **Aging:** Remove unused MAC entries after timeout period

#### MAC Address Table
```
VLAN | MAC Address       | Port | Age
-----|-------------------|------|----
1    | 00:1B:44:11:3A:B7 | Fa0/1| 5
1    | 00:0C:29:A1:B2:C3 | Fa0/2| 12
10   | 00:50:56:C0:00:01 | Fa0/5| 3
```

#### Spanning Tree Protocol (STP)
- **Purpose:** Prevent switching loops in redundant topologies
- **Root Bridge:** Switch with lowest bridge ID becomes root
- **Port States:** Blocking, Listening, Learning, Forwarding, Disabled
- **BPDU (Bridge Protocol Data Unit):** Messages exchanged between switches
- **Convergence:** Process of calculating loop-free topology

## Sample Exercises

1. **Frame Analysis:** Given an Ethernet frame capture, identify the source MAC, destination MAC, and frame type.

2. **ARP Troubleshooting:** A host cannot communicate with another host on the same subnet. The IP configuration is correct. What ARP-related issues could cause this?

3. **VLAN Design:** Design a VLAN scheme for a company with Sales (20 users), Engineering (50 users), and Management (10 users) departments.

4. **Switch Configuration:** Configure VLANs 10, 20, and 30 on a switch with appropriate port assignments.

5. **STP Analysis:** In a network with three switches connected in a triangle topology, predict which ports will be blocked by STP.

## Solutions

1. **Frame Analysis Example:**
   ```
   Dest MAC: FF:FF:FF:FF:FF:FF (Broadcast)
   Src MAC:  00:1B:44:11:3A:B7 (Sender's NIC)
   Type:     0x0806 (ARP)
   ```
   - This is an ARP request frame being broadcast to all devices

2. **ARP Troubleshooting Issues:**
   - **Duplicate IP addresses:** Two hosts with same IP cause ARP confusion
   - **ARP cache poisoning:** Malicious or incorrect ARP entries
   - **Network segmentation:** Hosts on different VLANs without routing
   - **Firewall blocking:** Security software blocking ARP traffic
   - **Switch MAC table full:** Switch cannot learn new MAC addresses
   - **Static ARP conflicts:** Incorrect manual ARP entries

3. **VLAN Design Solution:**
   ```
   VLAN 10: Sales Department (192.168.10.0/24)
   VLAN 20: Engineering Department (192.168.20.0/24)
   VLAN 30: Management Department (192.168.30.0/24)
   VLAN 99: Management VLAN for switch administration
   
   Benefits:
   - Broadcast domain separation
   - Security isolation between departments
   - Easier network management and troubleshooting
   ```

4. **Switch VLAN Configuration:**
   ```
   # Create VLANs
   vlan 10
   name Sales
   vlan 20
   name Engineering
   vlan 30
   name Management
   
   # Configure access ports
   interface range fa0/1-10
   switchport mode access
   switchport access vlan 10
   
   interface range fa0/11-20
   switchport mode access
   switchport access vlan 20
   
   # Configure trunk port
   interface gi0/1
   switchport mode trunk
   switchport trunk allowed vlan 10,20,30
   ```

5. **STP Analysis:**
   - **Root Bridge:** Switch with lowest bridge ID (priority + MAC)
   - **Root Ports:** Each non-root switch has one port toward root
   - **Designated Ports:** One per network segment, forwarding toward root
   - **Blocked Ports:** Remaining ports to prevent loops
   - **Result:** Two ports forwarding, one port blocked in triangle topology

## Completion Checklist
- [ ] Understand Ethernet frame structure and components
- [ ] Know MAC address format and special address types
- [ ] Understand ARP process and troubleshooting
- [ ] Can design and configure VLANs appropriately
- [ ] Understand switch operations and MAC address learning
- [ ] Familiar with Spanning Tree Protocol basics

## Key Takeaways
- Layer 2 provides reliable frame delivery between adjacent network devices
- MAC addresses enable communication within local network segments
- VLANs provide logical network segmentation for security and performance
- Switches create collision-free networks through dedicated bandwidth per port
- ARP bridges the gap between Layer 2 (MAC) and Layer 3 (IP) addressing
- Spanning Tree Protocol prevents loops in redundant switch topologies

## Next Steps
Proceed to [Day 4: Layer 3 - Network](../Day_04/notes_and_exercises.md) to learn about IP addressing, routing, subnetting, CIDR, and NAT concepts.