# Day 14: DHCP, NAT, PAT, Private/Public IPs, Cloud IP Management

## Learning Objectives
By the end of Day 14, you will:
- Understand DHCP lease process and configuration options
- Master NAT types: Static NAT, Dynamic NAT, and PAT (NAPT)
- Manage private and public IP addressing
- Apply cloud IP management concepts (Elastic IPs, floating IPs, cloud NAT)

**Estimated Time:** 3-4 hours

## Notes

### DHCP (Dynamic Host Configuration Protocol)

#### DHCP DORA Process
```
Client                          DHCP Server
  |                                  |
  |──── DISCOVER (broadcast) ───────>|   "I need an IP, anyone there?"
  |                                  |
  |<─── OFFER ──────────────────────|   "Take 192.168.1.100 for 24h"
  |                                  |
  |──── REQUEST (broadcast) ────────>|   "I accept 192.168.1.100"
  |                                  |
  |<─── ACK ────────────────────────|   "It's yours. Lease confirmed."
  |                                  |
```

DHCP uses UDP port **67** (server) and **68** (client).

#### DHCP Lease Options
DHCP assigns more than just an IP — it delivers a full network configuration:

| Option | Code | Description |
|--------|------|-------------|
| Subnet Mask | 1 | 255.255.255.0 |
| Router / Gateway | 3 | 192.168.1.1 |
| DNS Servers | 6 | 8.8.8.8, 8.8.4.4 |
| Domain Name | 15 | example.com |
| Broadcast | 28 | 192.168.1.255 |
| Lease Time | 51 | 86400 (seconds) |
| TFTP Server | 66 | For PXE boot |
| Boot File | 67 | pxelinux.0 |
| NTP Servers | 42 | 192.168.1.10 |

#### DHCP Reservation (Static Assignment)
```
# In dhcpd.conf — assign fixed IP based on MAC
host myserver {
  hardware ethernet 00:1A:2B:3C:4D:5E;
  fixed-address 192.168.1.50;
}
```

#### DHCP Relay Agent
When the DHCP server is on a different subnet, a **relay agent** (usually the router) forwards DHCP broadcasts across subnets:
```
Client → [broadcast] → Router (relay) → [unicast] → DHCP Server
```

### NAT (Network Address Translation)

#### Why NAT?
- IPv4 exhaustion: Only ~4.3 billion addresses for billions of devices
- Private address space (RFC 1918) is non-routable on the internet
- NAT maps private IPs to public IPs at the network boundary

#### NAT Types

**Static NAT**
- One-to-one mapping
- Private IP always maps to the same public IP
- Used for servers that need to be always reachable
```
192.168.1.10 → 203.0.113.10 (always)
```

**Dynamic NAT**
- Pool of public IPs assigned on demand
- Less common today
- No port translation
```
192.168.1.10 → (from pool) 203.0.113.20
192.168.1.11 → (from pool) 203.0.113.21
```

**PAT / NAPT (Port Address Translation)**
- Many-to-one: All private IPs share one public IP
- Port number differentiates connections
- Most common form of NAT (home routers, cloud NAT)
```
192.168.1.10:1234 → 203.0.113.1:40001
192.168.1.11:5678 → 203.0.113.1:40002
192.168.1.12:9012 → 203.0.113.1:40003
```

#### NAT Translation Table
| Protocol | Private Src | Private Port | Public IP | Public Port | Remote |
|----------|-------------|--------------|-----------|-------------|--------|
| TCP | 192.168.1.10 | 1234 | 203.0.113.1 | 40001 | 8.8.8.8:53 |
| TCP | 192.168.1.11 | 5678 | 203.0.113.1 | 40002 | 142.250.80.78:80 |

#### NAT Limitations
- Breaks end-to-end connectivity (inbound connections require port forwarding)
- Complicates protocols that embed IP in payload (FTP, SIP, H.323)
- ALG (Application Layer Gateway) handles protocol-specific NAT traversal
- NAT is not a firewall — it's address translation, not access control

### NAPT / Port Forwarding
Allows external clients to reach internal servers by mapping a public port to a private IP:port:
```
External: 203.0.113.1:8080 → Internal: 192.168.1.50:80
External: 203.0.113.1:2222 → Internal: 192.168.1.50:22
```

### Private vs Public IP Address Ranges

| Range | Type | RFC |
|-------|------|-----|
| 10.0.0.0/8 | Private (Class A) | RFC 1918 |
| 172.16.0.0/12 | Private (Class B) | RFC 1918 |
| 192.168.0.0/16 | Private (Class C) | RFC 1918 |
| 100.64.0.0/10 | Shared (CGNAT) | RFC 6598 |
| 169.254.0.0/16 | Link-local (APIPA) | RFC 3927 |
| 127.0.0.0/8 | Loopback | RFC 5735 |
| Everything else | Public (routable) | IANA |

### Cloud IP Management

#### AWS
- **Elastic IP (EIP):** Static public IPv4 that you own; survives instance stop/start
- **Auto-assigned Public IP:** Temporary; lost on stop
- **Private IP:** Assigned to ENI; persistent
- **VPC CIDR:** You choose the private range; secondary CIDRs supported
- **AWS NAT Gateway:** Fully managed NAT for private subnets; charged per GB

```bash
# AWS CLI: Allocate and associate Elastic IP
aws ec2 allocate-address --domain vpc
aws ec2 associate-address --instance-id i-xxxx --allocation-id eipalloc-xxxx
```

#### GCP
- **Static External IP:** Reserved, can be promoted from ephemeral
- **Ephemeral External IP:** Temporary per VM
- **Internal IP:** Persistent within VPC
- **Cloud NAT:** Regional; auto-scales; logs to Cloud Logging

#### Azure
- **Public IP (Static/Dynamic):** Assigned to NIC or load balancer
- **Private IP:** From VNet subnet range
- **Azure NAT Gateway:** Managed outbound NAT for subnets

## Hands-On Labs

### Lab 1: DHCP Lease Inspection
```bash
# View DHCP lease from client side
cat /var/lib/dhcp/dhclient.leases 2>/dev/null || \
  cat /var/lib/NetworkManager/dhclient-*.lease 2>/dev/null || \
  echo "Check: journalctl | grep DHCP"

# Show current DHCP-assigned network config
ip addr show
ip route show
cat /etc/resolv.conf

# Manually release and renew DHCP lease
sudo dhclient -r eth0 2>/dev/null || echo "using NetworkManager"
sudo dhclient eth0 2>/dev/null

# With nmcli
nmcli connection down "Wired connection 1" && nmcli connection up "Wired connection 1"
```

### Lab 2: View and Configure NAT (iptables)
```bash
# View current NAT rules
sudo iptables -t nat -L -n -v

# Enable IP forwarding (required for NAT)
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Masquerade (PAT) for all outbound traffic on eth0
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Port forwarding (forward external :8080 to internal :80)
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.50:80
sudo iptables -A FORWARD -p tcp -d 192.168.1.50 --dport 80 -j ACCEPT

# List all NAT rules
sudo iptables -t nat -L -n --line-numbers

# Remove a NAT rule
# sudo iptables -t nat -D POSTROUTING 1

# Save rules
sudo iptables-save > /tmp/iptables_backup.rules
```

### Lab 3: DHCP Server Setup (ISC DHCP)
```bash
# Install DHCP server
sudo apt install isc-dhcp-server -y 2>/dev/null || echo "Package install skipped"

# Example dhcpd.conf
cat <<'EOF'
# /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 7200;

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  option domain-name "example.local";
  option broadcast-address 192.168.1.255;
}

# Static assignment
host myserver {
  hardware ethernet 00:1A:2B:3C:4D:5E;
  fixed-address 192.168.1.50;
  option host-name "myserver";
}
EOF
echo "DHCP config example displayed"
```

## Practical Exercises

### Exercise 1: NAT Table Simulator
```python
#!/usr/bin/env python3
import random

class NATTable:
    def __init__(self, public_ip):
        self.public_ip = public_ip
        self.table = {}           # (private_ip, private_port) → public_port
        self.reverse = {}         # public_port → (private_ip, private_port)
        self.port_counter = 40000
    
    def translate_outbound(self, private_ip, private_port, dest_ip, dest_port, proto="TCP"):
        """PAT outbound translation"""
        key = (private_ip, private_port, dest_ip, dest_port)
        
        if key not in self.table:
            pub_port = self.port_counter
            self.port_counter += 1
            self.table[key] = pub_port
            self.reverse[pub_port] = key
        
        pub_port = self.table[key]
        return (self.public_ip, pub_port)
    
    def translate_inbound(self, pub_port, from_ip, from_port):
        """PAT inbound translation (return traffic)"""
        entry = self.reverse.get(pub_port)
        if entry:
            priv_ip, priv_port, _, _ = entry
            return (priv_ip, priv_port)
        return None
    
    def show_table(self):
        print(f"\nNAT Translation Table (Public IP: {self.public_ip})")
        print(f"{'Private Src':<22} {'Public Src':<22} {'Destination'}")
        print("-" * 70)
        for (priv_ip, priv_port, dst_ip, dst_port), pub_port in self.table.items():
            print(f"{priv_ip}:{priv_port:<10} → {self.public_ip}:{pub_port:<10} → {dst_ip}:{dst_port}")

# Simulate
nat = NATTable("203.0.113.1")

# Multiple internal hosts making connections
connections = [
    ("192.168.1.10", 1234, "8.8.8.8", 53),
    ("192.168.1.11", 5678, "142.250.80.78", 80),
    ("192.168.1.10", 2345, "142.250.80.78", 443),
    ("192.168.1.12", 9012, "1.1.1.1", 53),
]

for priv_ip, priv_port, dst_ip, dst_port in connections:
    pub_ip, pub_port = nat.translate_outbound(priv_ip, priv_port, dst_ip, dst_port)
    print(f"OUT: {priv_ip}:{priv_port} → {pub_ip}:{pub_port} (to {dst_ip}:{dst_port})")

nat.show_table()
```

### Exercise 2: DHCP Lease Monitor
```bash
#!/bin/bash
# Monitor active DHCP leases

LEASE_FILE="/var/lib/dhcp/dhcpd.leases"

if [ ! -f "$LEASE_FILE" ]; then
    echo "DHCP server lease file not found at $LEASE_FILE"
    echo "Sample DHCP lease format:"
    cat <<'EOF'
lease 192.168.1.100 {
  starts 1 2024/01/15 10:00:00;
  ends 1 2024/01/15 11:00:00;
  hardware ethernet 00:1A:2B:3C:4D:5E;
  client-hostname "laptop1";
}
EOF
    exit 0
fi

echo "Active DHCP Leases:"
echo "==================="
echo "IP Address         MAC Address         Hostname         Expires"
echo "---------- -------- ----------- -------"

grep -A6 "^lease" $LEASE_FILE | \
  awk '/^lease/{ip=$2} /hardware ethernet/{mac=$3} /client-hostname/{host=$2} /ends/{print ip, mac, host, $2, $3}' | \
  while read ip mac host day time; do
    printf "%-18s %-20s %-16s %s %s\n" "${ip}" "${mac//;/}" "${host//\";/}" "$day" "$time"
  done
```

### Exercise 3: IP Classification and Management
```python
#!/usr/bin/env python3
import ipaddress

def classify_ip(ip_str):
    """Classify an IP address"""
    try:
        ip = ipaddress.IPv4Address(ip_str)
    except:
        return "Invalid IP"
    
    if ip.is_loopback:
        return "Loopback"
    elif ip.is_link_local:
        return "Link-local (APIPA)"
    elif ip.is_private:
        if ip in ipaddress.IPv4Network("10.0.0.0/8"):
            return "Private Class A (RFC 1918)"
        elif ip in ipaddress.IPv4Network("172.16.0.0/12"):
            return "Private Class B (RFC 1918)"
        elif ip in ipaddress.IPv4Network("192.168.0.0/16"):
            return "Private Class C (RFC 1918)"
    elif ip in ipaddress.IPv4Network("100.64.0.0/10"):
        return "Shared/CGNAT (RFC 6598)"
    elif ip in ipaddress.IPv4Network("169.254.0.0/16"):
        return "Link-local"
    elif ip.is_multicast:
        return "Multicast"
    elif ip.is_reserved:
        return "Reserved"
    elif ip.is_global:
        return "Public (globally routable)"
    return "Unknown"

test_ips = [
    "10.0.0.1", "172.20.5.5", "192.168.1.100", "100.64.0.5",
    "169.254.10.10", "8.8.8.8", "127.0.0.1", "224.0.0.1",
    "203.0.113.1", "255.255.255.255"
]

print(f"{'IP Address':<20} {'Classification'}")
print("-" * 55)
for ip in test_ips:
    print(f"{ip:<20} {classify_ip(ip)}")
```

## Sample Exercises

1. **DHCP Design:** Design a DHCP server configuration for a network with 3 VLANs: Management (VLAN 10, /28), Servers (VLAN 20, /26), and Clients (VLAN 30, /23).

2. **NAT Troubleshooting:** An internal host `192.168.1.50` can browse the web but an external host cannot reach the web server running on port 80 at that same IP. What is likely missing and how do you fix it?

3. **PAT Calculation:** How many simultaneous connections can a single PAT device theoretically support with one public IP?

4. **Cloud IP Strategy:** Design an IP addressing strategy for a 3-tier AWS architecture (web, app, database) across 3 AZs.

5. **DHCP vs Static:** When should you use static IP assignment vs DHCP reservation vs full DHCP? Justify your decisions.

## Solutions

1. **DHCP VLAN Configuration:**
   ```
   VLAN 10 (Management): 192.168.10.0/28 → pool .2-.14 (13 IPs), GW .1
   VLAN 20 (Servers): 192.168.20.0/26 → pool .10-.62 (53 IPs), GW .1
   VLAN 30 (Clients): 192.168.30.0/23 → pool .10-510 (500 IPs), GW .1
   Note: Requires DHCP relay on each VLAN interface pointing to DHCP server
   ```

2. **NAT Troubleshooting — Port Forwarding Missing:**
   ```bash
   # Add DNAT rule to forward external :80 to internal server
   sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT \
     --to-destination 192.168.1.50:80
   sudo iptables -A FORWARD -p tcp -d 192.168.1.50 --dport 80 -j ACCEPT
   # PAT handles outbound; DNAT handles inbound
   ```

3. **PAT Connection Capacity:**
   - TCP/UDP ports: 1–65535 (practical: 1024–65535 = ~64,000 per protocol)
   - With TCP + UDP: ~128,000 concurrent connections per destination IP
   - Real-world: per 5-tuple (proto, src IP, src port, dst IP, dst port)
   - Theoretical max: hundreds of thousands of connections with port multiplexing

4. **AWS 3-Tier IP Strategy:**
   ```
   VPC: 10.0.0.0/16
   
   AZ-a:
     Public (Web):  10.0.1.0/24
     Private (App): 10.0.11.0/24
     Private (DB):  10.0.21.0/24
   
   AZ-b:
     Public (Web):  10.0.2.0/24
     Private (App): 10.0.12.0/24
     Private (DB):  10.0.22.0/24
   
   AZ-c:
     Public (Web):  10.0.3.0/24
     Private (App): 10.0.13.0/24
     Private (DB):  10.0.23.0/24
   
   NAT Gateways: One per AZ in public subnets
   Elastic IPs: On NAT Gateways (not on instances)
   ```

5. **DHCP Assignment Strategy:**
   - **Static IP:** Network devices (routers, switches, firewalls), DNS servers, NTP servers — anything that other devices depend on finding at a predictable address
   - **DHCP Reservation:** Servers and printers — they get a predictable IP from DHCP but benefit from central management; easier to change than editing host files
   - **Full DHCP:** Client workstations, laptops, phones — devices that move or are temporary; no need for a fixed address

## Completion Checklist
- [ ] Explain the DHCP DORA process and all key options
- [ ] Distinguish Static NAT, Dynamic NAT, and PAT
- [ ] Configure iptables MASQUERADE and DNAT rules
- [ ] Identify private, public, link-local, and CGNAT ranges
- [ ] Understand cloud IP management (EIP, Cloud NAT, Azure Public IP)
- [ ] Design DHCP schemes for multi-VLAN environments

## Key Takeaways
- DHCP automates IP assignment with a 4-step DORA exchange
- PAT (overloaded NAT) is the dominant form of NAT in homes and enterprises
- Private IPs require NAT to access the internet; public IPs do not
- Port forwarding (DNAT) allows inbound access to private servers
- Cloud providers abstract NAT into managed services; Elastic IPs provide persistence

## Next Steps
Proceed to [Day 15: Firewalls (iptables, nftables, ufw, Security Groups, Cloud Firewalls)](../Day_15/notes_and_exercises.md) to learn how to control and secure network traffic.