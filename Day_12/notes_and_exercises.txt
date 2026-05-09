# Day 12: Linux Networking Commands (ip, ifconfig, netstat, ss, nmcli, etc.)

## Learning Objectives
By the end of Day 12, you will:
- Master the `ip` command suite for modern Linux networking
- Use `ss` and `netstat` for connection and socket analysis
- Configure network interfaces with `nmcli` and `ifconfig`
- Perform diagnostics with `ping`, `traceroute`, `nmap`, `tcpdump`, and more

**Estimated Time:** 3-4 hours

## Notes

### Modern vs Legacy Tools

| Legacy (deprecated) | Modern Replacement | Package |
|---------------------|-------------------|---------|
| `ifconfig` | `ip addr` | iproute2 |
| `route` | `ip route` | iproute2 |
| `arp` | `ip neighbor` | iproute2 |
| `netstat` | `ss` | iproute2 |
| `iwconfig` | `iw` | iw |

The `iproute2` package provides the modern `ip` command. Always prefer it in scripts.

### The `ip` Command Suite

#### Interface Management
```bash
# Show all interfaces
ip link show
ip link show eth0          # specific interface

# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Set MTU
sudo ip link set eth0 mtu 9000

# Rename interface
sudo ip link set eth0 name mynic

# Show interface statistics
ip -s link show eth0
```

#### IP Address Management
```bash
# Show all IP addresses
ip addr show
ip addr show eth0

# Add IP address
sudo ip addr add 192.168.1.100/24 dev eth0

# Add secondary IP
sudo ip addr add 192.168.1.101/24 dev eth0 label eth0:1

# Delete IP address
sudo ip addr del 192.168.1.100/24 dev eth0

# Flush all addresses from interface
sudo ip addr flush dev eth0
```

#### Route Management
```bash
# Show routing table
ip route show
ip route show table all

# Add routes
sudo ip route add 10.0.0.0/8 via 192.168.1.1
sudo ip route add default via 192.168.1.1

# Get route for specific destination
ip route get 8.8.8.8

# Delete route
sudo ip route del 10.0.0.0/8

# Show route cache
ip route show cache
```

#### ARP / Neighbor Table
```bash
# Show ARP/neighbor table
ip neighbor show
ip neigh show dev eth0

# Add static ARP entry
sudo ip neigh add 192.168.1.50 lladdr 00:11:22:33:44:55 dev eth0

# Delete ARP entry
sudo ip neigh del 192.168.1.50 dev eth0

# Flush neighbor table
sudo ip neigh flush dev eth0
```

### The `ss` Command (Socket Statistics)

```bash
# Show all sockets
ss -a

# Show TCP sockets
ss -t

# Show UDP sockets
ss -u

# Show listening sockets
ss -l

# Show process info
ss -p

# Common combinations
ss -tuln        # TCP+UDP, listening, no DNS, numeric
ss -tnp         # TCP, no DNS, with process names
ss -s           # Summary statistics

# Filter by state
ss -t state established
ss -t state time-wait
ss -t state listen

# Filter by port
ss -tnp sport = :80
ss -tnp dport = :443

# Show socket timers
ss -to
```

### The `netstat` Command (Legacy)
```bash
# Show all connections
netstat -a

# Show listening ports
netstat -l

# Show TCP connections
netstat -t

# Show UDP
netstat -u

# Show process names
netstat -p

# Show statistics
netstat -s

# Show routing table
netstat -r

# Common: all TCP/UDP listening, numeric, with process
netstat -tulnp
```

### Interface Configuration with `nmcli`
```bash
# Show connection status
nmcli general status

# List all connections
nmcli connection show

# Show active connections
nmcli connection show --active

# Show device status
nmcli device status

# Connect/disconnect
nmcli connection up "Wired connection 1"
nmcli connection down "Wired connection 1"

# Create a new connection
nmcli connection add type ethernet con-name myconn ifname eth0 \
  ip4 192.168.1.200/24 gw4 192.168.1.1

# Modify existing
nmcli connection modify myconn ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection modify myconn ipv4.method manual

# Delete connection
nmcli connection delete myconn

# Scan Wi-Fi
nmcli device wifi list

# Connect to Wi-Fi
nmcli device wifi connect "SSID" password "passphrase"
```

### Diagnostic Tools

#### `ping` — Connectivity Test
```bash
ping google.com              # continuous
ping -c 5 google.com         # 5 packets
ping -i 0.2 google.com       # 200ms interval
ping -s 1400 google.com      # custom packet size (test MTU)
ping -W 1 google.com         # 1 second timeout
ping6 2001:4860:4860::8888   # IPv6 ping
```

#### `traceroute` / `tracepath` — Path Discovery
```bash
traceroute google.com
traceroute -n google.com      # no DNS resolution (faster)
traceroute -T -p 443 google.com  # TCP traceroute (bypass firewalls)
tracepath google.com          # MTU-aware path trace
mtr google.com                # real-time traceroute + ping
mtr --report -n google.com    # 10-run report, no DNS
```

#### `nmap` — Port Scanning
```bash
# Install
sudo apt install nmap

# TCP connect scan
nmap -sT 192.168.1.1

# SYN scan (stealth)
sudo nmap -sS 192.168.1.1

# OS detection + version
sudo nmap -A 192.168.1.1

# Scan a range
nmap 192.168.1.0/24

# Scan specific ports
nmap -p 22,80,443 192.168.1.1

# UDP scan
sudo nmap -sU -p 53,161 192.168.1.1
```

#### `tcpdump` — Packet Capture
```bash
# Capture all traffic on interface
sudo tcpdump -i eth0

# Capture to file
sudo tcpdump -i eth0 -w capture.pcap

# Read capture file
tcpdump -r capture.pcap

# Filter by host
sudo tcpdump -i any host 8.8.8.8

# Filter by port
sudo tcpdump -i any port 80

# Filter by protocol
sudo tcpdump -i any icmp
sudo tcpdump -i any tcp

# Verbose output
sudo tcpdump -i any -v port 443

# No DNS resolution
sudo tcpdump -n -i any port 53
```

#### Other Useful Commands
```bash
# Show DNS configuration
cat /etc/resolv.conf
resolvectl status

# Check hostname
hostname
hostname -I      # show all IPs

# Network stats
cat /proc/net/dev

# Interface statistics
ethtool eth0
ethtool -S eth0     # detailed NIC stats

# Bandwidth monitoring
iftop -i eth0       # install: apt install iftop
nethogs             # per-process bandwidth

# Socket buffer sizes
sysctl net.core.rmem_max
sysctl net.core.wmem_max
```

## Hands-On Labs

### Lab 1: Interface and Address Management
```bash
# Full interface overview
ip -brief addr show

# Detailed stats
ip -s link show eth0 2>/dev/null || ip -s link show $(ip link | grep -v lo | grep -m1 "state UP" | cut -d: -f2 | tr -d ' ')

# Show IPv4 and IPv6
ip -4 addr show
ip -6 addr show

# Add a temporary alias
sudo ip addr add 192.168.99.1/24 dev lo
ip addr show lo
sudo ip addr del 192.168.99.1/24 dev lo
```

### Lab 2: Socket and Connection Analysis
```bash
# Count connections per state
echo "TCP Connection States:"
ss -t | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# Top destination ports
echo -e "\nTop destination ports:"
ss -tn | awk 'NR>1 {print $5}' | awk -F: '{print $NF}' | sort | uniq -c | sort -rn | head -10

# Show connections with PIDs (requires root)
echo -e "\nListening services:"
sudo ss -tlnp | grep LISTEN
```

### Lab 3: Network Diagnostic Script
```bash
#!/bin/bash
# Comprehensive network diagnostic

TARGET=${1:-8.8.8.8}
echo "=== Network Diagnostics: $TARGET ==="

echo -e "\n1. Interface Status:"
ip -brief link show | grep -v "^lo"

echo -e "\n2. IP Addresses:"
ip -brief addr show | grep -v "^lo"

echo -e "\n3. Default Gateway:"
ip route | grep default

echo -e "\n4. DNS Servers:"
grep nameserver /etc/resolv.conf

echo -e "\n5. Ping Test:"
ping -c 3 -W 2 $TARGET | tail -2

echo -e "\n6. Traceroute (first 5 hops):"
traceroute -n -m 5 $TARGET 2>/dev/null | tail -6

echo -e "\n7. Open Ports (local listening):"
ss -tlnp | grep LISTEN | awk '{print $4, $6}' | head -10

echo -e "\n8. Active Established Connections:"
ss -tn state established | wc -l
echo " established TCP connections"
```

## Practical Exercises

### Exercise 1: Network Inventory Tool
```bash
#!/bin/bash
# Complete network inventory

echo "=============================="
echo " Network Inventory Report"
echo " $(date)"
echo "=============================="

echo -e "\n[Interfaces]"
ip -brief link show

echo -e "\n[IP Addresses]"
ip -brief addr show

echo -e "\n[Routes]"
ip route show

echo -e "\n[ARP/Neighbors]"
ip neighbor show | grep -v "^$"

echo -e "\n[DNS]"
cat /etc/resolv.conf | grep -v "^#"

echo -e "\n[Listening Services]"
ss -tlnp | grep LISTEN

echo -e "\n[Established Connections (top 10)]"
ss -tn state established | head -11

echo -e "\n[Interface Statistics]"
ip -s link show | grep -E "^[0-9]|RX|TX" | head -20
```

### Exercise 2: Port Availability Checker
```python
#!/usr/bin/env python3
import socket
import concurrent.futures

def check_port(host, port, timeout=1):
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return port, True
    except:
        return port, False

def scan_ports(host, ports):
    print(f"Scanning {host}...")
    results = {}
    with concurrent.futures.ThreadPoolExecutor(max_workers=50) as executor:
        futures = {executor.submit(check_port, host, p): p for p in ports}
        for future in concurrent.futures.as_completed(futures):
            port, is_open = future.result()
            results[port] = is_open
    return results

common_ports = [21, 22, 23, 25, 53, 80, 110, 143, 443, 993, 995, 3306, 5432, 6379, 8080, 8443]

targets = ["localhost", "8.8.8.8"]
for target in targets:
    print(f"\nResults for {target}:")
    results = scan_ports(target, common_ports)
    for port in sorted(results):
        status = "OPEN  " if results[port] else "closed"
        service = {22: "SSH", 80: "HTTP", 443: "HTTPS", 53: "DNS", 3306: "MySQL"}.get(port, "")
        print(f"  Port {port:<6} {status}  {service}")
```

### Exercise 3: Bandwidth Monitor
```bash
#!/bin/bash
# Simple interface bandwidth monitor

IFACE=${1:-eth0}
INTERVAL=${2:-2}

echo "Monitoring $IFACE every ${INTERVAL}s (Ctrl+C to stop)"
echo "Time        RX Rate      TX Rate"
echo "----------- ------------ ------------"

prev_rx=0
prev_tx=0

while true; do
    rx=$(cat /sys/class/net/$IFACE/statistics/rx_bytes 2>/dev/null || echo 0)
    tx=$(cat /sys/class/net/$IFACE/statistics/tx_bytes 2>/dev/null || echo 0)
    
    if [ $prev_rx -gt 0 ]; then
        rx_rate=$(( (rx - prev_rx) / INTERVAL / 1024 ))
        tx_rate=$(( (tx - prev_tx) / INTERVAL / 1024 ))
        printf "%-12s %-12s %-12s\n" "$(date +%H:%M:%S)" "${rx_rate} KB/s" "${tx_rate} KB/s"
    fi
    
    prev_rx=$rx
    prev_tx=$tx
    sleep $INTERVAL
done
```

## Sample Exercises

1. **Command Mapping:** For each task, write the correct `ip` command: (a) show all interfaces, (b) add IP 10.0.0.5/24 to eth1, (c) add default route via 10.0.0.1, (d) show ARP table, (e) flush route cache.

2. **ss vs netstat:** What `ss` command would replace `netstat -tulnp`?

3. **Packet Capture Challenge:** Write a `tcpdump` one-liner that captures DNS queries to 8.8.8.8 and saves to a file called `dns_queries.pcap`.

4. **Troubleshooting:** `ping 192.168.1.1` works but `ping google.com` fails. What commands would you run and why?

5. **Script Challenge:** Write a bash script that alerts if any listening port above 1024 is opened by a root-owned process.

## Solutions

1. **ip command answers:**
   ```bash
   a) ip link show
   b) sudo ip addr add 10.0.0.5/24 dev eth1
   c) sudo ip route add default via 10.0.0.1
   d) ip neighbor show
   e) sudo ip route flush cache
   ```

2. **ss equivalent:** `ss -tulnp` (identical flags work the same way)

3. **tcpdump one-liner:**
   ```bash
   sudo tcpdump -i any -n host 8.8.8.8 and port 53 -w dns_queries.pcap
   ```

4. **Troubleshooting sequence:**
   ```bash
   ping 8.8.8.8         # Test if IP connectivity works (skip DNS)
   cat /etc/resolv.conf  # Check DNS server config
   nslookup google.com   # Test DNS resolution manually
   dig @8.8.8.8 google.com  # Test with specific DNS server
   # If IP ping works but DNS fails → DNS configuration issue
   ```

5. **Root-owned high port alert script:**
   ```bash
   #!/bin/bash
   ss -tlnp | grep LISTEN | awk '{print $4, $6}' | while read port proc; do
     portnum=$(echo $port | awk -F: '{print $NF}')
     if [ "$portnum" -gt 1024 ] 2>/dev/null && echo "$proc" | grep -q "root"; then
       echo "ALERT: Root process listening on high port $portnum: $proc"
     fi
   done
   ```

## Completion Checklist
- [ ] Use `ip addr`, `ip link`, `ip route`, `ip neigh` confidently
- [ ] Use `ss -tuln` and understand all output columns
- [ ] Capture and filter traffic with `tcpdump`
- [ ] Run targeted scans with `nmap`
- [ ] Configure interfaces and connections with `nmcli`
- [ ] Write diagnostic scripts combining multiple commands

## Key Takeaways
- `ip` is the modern replacement for `ifconfig`, `route`, and `arp`
- `ss` is faster and more feature-rich than `netstat`
- `tcpdump` is your primary packet-level debugging tool
- Combining these tools in scripts enables powerful automated diagnostics
- Understanding command output is as important as knowing the commands

## Next Steps
Proceed to [Day 13: DNS Concepts, Cloud DNS, Troubleshooting](../Day_13/notes_and_exercises.md) to master the internet's phone book.