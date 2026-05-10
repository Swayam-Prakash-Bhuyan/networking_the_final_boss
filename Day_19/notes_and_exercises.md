# Day 19: Network Monitoring & Troubleshooting (ping, traceroute, mtr, tcpdump, Wireshark)

## Learning Objectives
By the end of Day 19, you will:
- Build a systematic troubleshooting methodology
- Master diagnostic tools: ping, traceroute, mtr, tcpdump, ss, nmap, netcat
- Capture and analyze packets with tcpdump and Wireshark
- Identify common network problems and their root causes

**Estimated Time:** 3-4 hours

---

## Notes

### Troubleshooting Methodology

Always follow a **systematic approach** — don't guess randomly:

```
1. Define the problem clearly
   - Who is affected? When did it start? What changed?

2. Gather information
   - Logs, metrics, user reports, error messages

3. Isolate the layer (OSI bottom-up)
   - L1 Physical → L2 Data Link → L3 Network → L4 Transport → L7 Application

4. Form a hypothesis

5. Test the hypothesis

6. Implement fix

7. Verify and document
```

### Tool Selection by Layer

| Layer | Problem | Tool |
|-------|---------|------|
| L1 | Link down | `ip link show`, `ethtool` |
| L2 | ARP issues | `ip neighbor`, `arp`, `tcpdump arp` |
| L3 | Routing, no path | `ping`, `ip route`, `traceroute` |
| L4 | Port closed, timeout | `ss`, `nc`, `nmap`, `tcpdump` |
| L7 | App errors | `curl -v`, `openssl s_client`, app logs |

### ping
```bash
# Basic connectivity test
ping -c 4 8.8.8.8

# Set packet size (test MTU)
ping -s 1472 -M do 8.8.8.8   # 1472 + 28 = 1500 (standard MTU)
ping -s 8972 -M do 8.8.8.8   # Test jumbo frames (9000 MTU)

# Set interval and timeout
ping -c 10 -i 0.2 -W 1 8.8.8.8

# Flood ping (requires root) - stress test
sudo ping -f -c 1000 192.168.1.1

# IPv6 ping
ping6 -c 4 2001:4860:4860::8888
```

**Reading ping output:**
```
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=12.3 ms
                                    ^         ^
                                  hops       RTT

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 11.2/12.3/13.8/1.1 ms
                                          ^
                                        jitter
```

### traceroute / tracepath
```bash
# Standard traceroute (UDP by default on Linux)
traceroute google.com

# ICMP mode (sometimes passes firewalls better)
traceroute -I google.com

# TCP traceroute on port 443 (best for firewall traversal)
traceroute -T -p 443 google.com

# No DNS resolution (faster)
traceroute -n google.com

# Set max hops
traceroute -m 20 google.com

# tracepath — also shows MTU
tracepath google.com
```

**Interpreting traceroute:**
```
 1  192.168.1.1 (192.168.1.1)      1.2 ms   ← default gateway
 2  10.0.0.1 (10.0.0.1)           5.3 ms   ← ISP router
 3  * * *                                    ← firewall/ICMP blocked
 4  142.250.80.1 (142.250.80.1)  12.4 ms   ← Google's network
 5  216.239.49.3 (216.239.49.3)  13.1 ms
 6  8.8.8.8 (dns.google)          13.8 ms  ← destination
```

### mtr (My Traceroute)
Real-time combination of ping + traceroute — the go-to tool for intermittent packet loss.

```bash
# Interactive mode
mtr google.com

# Report mode (10 cycles, then exit)
mtr --report -n -c 10 google.com

# With hostnames and AS numbers
mtr --report --show-ips google.com

# TCP mode
mtr --tcp --port 443 google.com
```

**mtr columns:**
```
Host         Loss%  Snt  Last  Avg  Best  Wrst  StDev
1. gateway    0.0%   10   1.2   1.3   1.1   1.8   0.2
2. 10.0.0.1   0.0%   10   5.1   5.3   4.9   6.1   0.4
3. ???       100.0%  10   0.0   0.0   0.0   0.0   0.0  ← ICMP blocked (OK if next hops respond)
4. 8.8.8.8   0.0%   10  13.2  13.5  12.9  14.1   0.4
```

### tcpdump
```bash
# Capture all traffic on an interface
sudo tcpdump -i eth0

# Write to file (for Wireshark analysis)
sudo tcpdump -i any -w /tmp/capture.pcap

# Read from file
tcpdump -r /tmp/capture.pcap

# No DNS resolution, verbose
sudo tcpdump -i any -nn -v

# Filter by host
sudo tcpdump -i any host 8.8.8.8

# Filter by port
sudo tcpdump -i any port 80

# Filter by protocol
sudo tcpdump -i any icmp
sudo tcpdump -i any tcp
sudo tcpdump -i any udp

# Complex filters (BPF syntax)
sudo tcpdump -i any 'tcp port 443 and host 8.8.8.8'
sudo tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0'  # SYN packets only
sudo tcpdump -i any 'icmp[icmptype] = icmp-echoreply'

# Capture limited packets
sudo tcpdump -i any -c 100 port 80 -w http.pcap

# Show packet content (hex + ASCII)
sudo tcpdump -i any -X port 80
sudo tcpdump -i any -A port 80  # ASCII only
```

### ss (Socket Statistics)
```bash
# All TCP connections with process names
ss -tnp

# Listening ports
ss -tlnp

# Connections in TIME_WAIT (common after high traffic)
ss -t state time-wait | wc -l

# Connections to a specific port
ss -tn dst :443

# Show socket details including retransmissions
ss -ti

# UDP sockets
ss -unp
```

### netcat (nc) — Network Swiss Army Knife
```bash
# Test TCP port connectivity
nc -zv google.com 443
nc -zv -w 3 192.168.1.1 22   # 3 second timeout

# Test UDP port
nc -zuv 8.8.8.8 53

# Scan a range of ports
nc -zv 192.168.1.1 20-100

# Create a simple listener (receive data)
nc -l 9999

# Send data to listener
echo "hello" | nc localhost 9999

# File transfer with nc
# Receiver:  nc -l 9999 > received_file
# Sender:    nc receiver_ip 9999 < file_to_send

# Banner grabbing
echo "" | nc -w 1 192.168.1.10 22
```

### nmap
```bash
# Host discovery (which hosts are up)
nmap -sn 192.168.1.0/24

# TCP port scan
nmap -sT 192.168.1.10

# SYN scan (faster, less noisy)
sudo nmap -sS 192.168.1.10

# Service version detection
nmap -sV 192.168.1.10

# OS detection
sudo nmap -O 192.168.1.10

# Comprehensive scan
sudo nmap -A 192.168.1.10

# Specific ports
nmap -p 22,80,443,8080,3306 192.168.1.10

# UDP scan (slower)
sudo nmap -sU -p 53,161,123 192.168.1.10

# Output to file
nmap -A 192.168.1.10 -oN scan.txt
```

### Common Problems and Diagnostics

#### Problem: "No route to host"
```bash
ip route get <dest>          # Does a route exist?
ping <gateway>               # Is gateway reachable?
ip route show                # Check routing table
traceroute <dest>            # Where does it fail?
```

#### Problem: Intermittent packet loss
```bash
mtr --report -c 50 <dest>    # Identify which hop has loss
ping -c 100 <dest>           # Measure loss percentage
sudo tcpdump -i any icmp     # Capture ICMP to see patterns
```

#### Problem: DNS not resolving
```bash
cat /etc/resolv.conf          # Check configured DNS servers
dig google.com                # Test DNS resolution
dig @8.8.8.8 google.com       # Test with public DNS
nc -zv 8.8.8.8 53             # Is DNS port reachable?
```

#### Problem: Service unreachable on port
```bash
ss -tlnp | grep :PORT         # Is service listening?
nc -zv <host> <port>          # Can we connect?
sudo iptables -L -n | grep PORT # Firewall blocking?
curl -v http://<host>:<port>  # Application-level response
```

---

## Hands-On Labs

### Lab 1: Full Network Diagnostic
```bash
#!/bin/bash
TARGET=${1:-8.8.8.8}
echo "=== Full Network Diagnostic for: $TARGET ==="
echo ""

echo "[1] Interface status:"
ip -brief link show | grep -v "^lo"

echo -e "\n[2] IP and Gateway:"
ip -brief addr show | grep -v "^lo"
ip route | grep default

echo -e "\n[3] DNS:"
grep nameserver /etc/resolv.conf

echo -e "\n[4] Ping (5 packets):"
ping -c 5 -W 2 $TARGET | tail -2

echo -e "\n[5] Traceroute (first 10 hops):"
traceroute -n -m 10 $TARGET 2>/dev/null | head -12

echo -e "\n[6] Listening services:"
ss -tlnp | grep LISTEN | awk '{print "  "$4, $6}' | head -10

echo -e "\n[7] Connection count:"
echo "  Established: $(ss -t state established | wc -l)"
echo "  Time-wait:   $(ss -t state time-wait | wc -l)"
echo "  Listening:   $(ss -l | wc -l)"
```

### Lab 2: Packet Capture and Analysis
```bash
# Capture DNS traffic for 10 seconds
sudo timeout 10 tcpdump -i any -nn port 53 -w /tmp/dns.pcap 2>/dev/null &
# Generate some DNS traffic
for host in google.com github.com stackoverflow.com; do
    dig $host +short > /dev/null
done
wait

# Analyze the capture
echo "DNS Queries Captured:"
tcpdump -r /tmp/dns.pcap -nn 2>/dev/null | grep -o "A? [^ ]*" | sort | uniq -c | sort -rn

# Capture HTTP traffic
sudo timeout 5 tcpdump -i any -nn -A port 80 -c 20 2>/dev/null &
curl -s http://httpbin.org/get > /dev/null
wait
```

### Lab 3: MTU Discovery
```bash
#!/bin/bash
# Find the path MTU to a destination

TARGET=${1:-8.8.8.8}
echo "MTU Discovery to $TARGET"
echo "========================="

for size in 1500 1472 1400 1300 1200; do
    payload=$((size - 28))  # 20 IP + 8 ICMP header
    if ping -c 1 -W 1 -s $payload -M do $TARGET &>/dev/null; then
        echo "  Size $size: ✓ PASS"
        LAST_OK=$size
    else
        echo "  Size $size: ✗ FRAGMENTED/BLOCKED"
    fi
done
echo "Detected MTU: ~${LAST_OK:-unknown}"
```

---

## Practical Exercises

### Exercise 1: Network Troubleshooter Script
```bash
#!/bin/bash
# Automated network troubleshooter

run_check() {
    local desc=$1
    local cmd=$2
    local expected=$3
    result=$(eval "$cmd" 2>/dev/null)
    if echo "$result" | grep -q "$expected"; then
        echo "  ✓ $desc"
    else
        echo "  ✗ $desc → $result"
    fi
}

echo "Network Troubleshooter"
echo "======================"

echo -e "\nLayer 1 - Physical:"
run_check "Interface UP" "ip link show | grep -v lo | grep -c 'state UP'" "1"

echo -e "\nLayer 3 - Network:"
run_check "Default route exists" "ip route | grep -c 'default'" "1"
run_check "Gateway reachable" "ping -c1 -W2 $(ip route | awk '/default/{print $3}' | head -1)" "1 received"
run_check "Internet reachable (IP)" "ping -c1 -W3 8.8.8.8" "1 received"

echo -e "\nLayer 7 - Application (DNS):"
run_check "DNS resolution" "nslookup google.com" "Address"
run_check "HTTPS reachable" "curl -sk --max-time 5 https://google.com" ""

echo -e "\nSummary complete."
```

### Exercise 2: Packet Loss Analyzer
```python
#!/usr/bin/env python3
import subprocess, re

def run_mtr(host, count=20):
    try:
        result = subprocess.run(
            ['mtr', '--report', '--report-cycles', str(count), '-n', host],
            capture_output=True, text=True, timeout=60
        )
        return result.stdout
    except FileNotFoundError:
        return None

def parse_mtr(output):
    hops = []
    for line in output.split('\n'):
        m = re.match(r'\s+\d+\.\s+(\S+)\s+([\d.]+)%\s+(\d+)\s+([\d.]+)\s+([\d.]+)', line)
        if m:
            hops.append({
                'host': m.group(1),
                'loss': float(m.group(2)),
                'avg': float(m.group(5))
            })
    return hops

output = run_mtr('8.8.8.8')
if output:
    hops = parse_mtr(output)
    print(f"{'Hop':<5}{'Host':<20}{'Loss%':>7}{'Avg ms':>8}")
    print('-' * 42)
    for i, hop in enumerate(hops, 1):
        flag = ' ⚠ LOSS' if hop['loss'] > 5 else ''
        print(f"{i:<5}{hop['host']:<20}{hop['loss']:>6.1f}%{hop['avg']:>7.1f}{flag}")
else:
    print("mtr not available. Install with: sudo apt install mtr")
```

### Exercise 3: Port Scanner with Service ID
```python
#!/usr/bin/env python3
import socket, concurrent.futures

SERVICES = {
    21: "FTP", 22: "SSH", 23: "Telnet", 25: "SMTP", 53: "DNS",
    80: "HTTP", 110: "POP3", 143: "IMAP", 443: "HTTPS",
    3306: "MySQL", 5432: "PostgreSQL", 6379: "Redis",
    8080: "HTTP-Alt", 8443: "HTTPS-Alt", 27017: "MongoDB"
}

def scan(host, port, timeout=1):
    try:
        s = socket.socket()
        s.settimeout(timeout)
        s.connect((host, port))
        # Try banner grab
        try:
            s.send(b'\r\n')
            banner = s.recv(256).decode(errors='ignore').strip()[:50]
        except:
            banner = ""
        s.close()
        return port, True, banner
    except:
        return port, False, ""

host = "localhost"
print(f"Scanning {host}...")
print(f"{'Port':<7}{'State':<10}{'Service':<15}{'Banner'}")
print("-" * 55)
with concurrent.futures.ThreadPoolExecutor(max_workers=30) as ex:
    futures = [ex.submit(scan, host, p) for p in SERVICES]
    for f in concurrent.futures.as_completed(futures):
        port, open_, banner = f.result()
        if open_:
            svc = SERVICES.get(port, "unknown")
            print(f"{port:<7}{'OPEN':<10}{svc:<15}{banner}")
```

---

## Sample Exercises

1. **Packet loss:** mtr shows 30% loss at hop 3 but 0% loss at hops 4–10. Is there a real problem? Explain.
2. **Firewall vs no route:** Explain the difference between `ping` returning "Destination host unreachable" vs "Request timeout".
3. **tcpdump filter:** Write a tcpdump filter to capture only TCP SYN packets to port 80 or 443.
4. **MTU issue:** Users report slow speeds and occasional disconnections when sending large files. How would you diagnose an MTU black hole?
5. **Retransmissions:** `ss -ti` shows a high retrans count. What does this indicate and how do you investigate?

## Solutions

1. **Packet loss at middle hop:** Not a real problem. ICMP rate-limiting at intermediate routers causes apparent loss; the traffic to the final destination passes fine. Real loss would appear at hop 3 AND all subsequent hops.
2. **Unreachable vs timeout:** "Destination host unreachable" = router explicitly sends back ICMP unreachable (router knows there's no path). "Request timeout" = ICMP packets are silently dropped (firewall, or the target simply doesn't respond).
3. **tcpdump SYN filter:** `sudo tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0 and (port 80 or port 443)'`
4. **MTU black hole diagnosis:** `ping -s 1472 -M do <dest>` — if 1472 fails but smaller sizes succeed, MTU is the issue. Solution: Set interface MTU lower (`ip link set eth0 mtu 1400`), or configure MSS clamping with iptables: `iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`.
5. **High retransmissions:** Indicates packet loss or congestion causing TCP to retransmit segments. Investigate with: `mtr <remote_host>` for path loss; check NIC errors with `ip -s link show`; review server CPU and memory; check for bandwidth saturation with `iftop`.

## Completion Checklist
- [ ] Follow the OSI-layered troubleshooting methodology
- [ ] Use ping, traceroute, and mtr correctly for path analysis
- [ ] Write tcpdump BPF filters for specific traffic
- [ ] Use ss to analyze socket states and connection counts
- [ ] Scan ports and detect services with nmap and nc
- [ ] Diagnose MTU, packet loss, and DNS issues systematically

## Key Takeaways
- Layer-by-layer troubleshooting prevents wasted time chasing the wrong problem
- mtr is the best single tool for diagnosing path-level issues
- tcpdump is irreplaceable for deep protocol-level debugging
- Packet loss at an intermediate hop doesn't always mean a real problem
- MTU mismatches cause mysterious slow speeds and disconnections

## Next Steps
Proceed to [Day 20: Cloud Networking Basics (VPC, Subnets, Gateways, Peering)](../Day_20/notes_and_exercises.md).