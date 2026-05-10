# Day 28: Network Performance, QoS, Bottlenecks, Cloud Monitoring

## Learning Objectives
By the end of Day 28, you will:
- Identify and resolve common network performance bottlenecks
- Understand QoS (Quality of Service) concepts and implementation
- Use cloud-native monitoring tools (CloudWatch, Cloud Monitoring, Azure Monitor)
- Build dashboards and alerts for network health

**Estimated Time:** 3-4 hours

---

## Notes

### Network Performance Metrics

| Metric | Definition | Good Value | Measurement Tool |
|--------|-----------|------------|-----------------|
| Latency | RTT delay | <10ms LAN, <100ms WAN | ping, mtr |
| Throughput | Actual data rate | Close to link speed | iperf3 |
| Bandwidth | Maximum capacity | Link spec | ethtool |
| Packet loss | % packets dropped | <0.1% | ping, mtr |
| Jitter | RTT variation | <10ms for VoIP | mtr, iperf3 |
| Retransmissions | TCP retransmit count | Low | ss -ti |
| Connection time | TCP handshake duration | <50ms | curl timing |
| DNS resolution | Name lookup time | <50ms | dig |

### Common Network Bottlenecks

#### 1. Bandwidth Saturation
```bash
# Detect: interface at 100% utilization
iftop -i eth0              # real-time per-connection
nethogs                    # per-process bandwidth
cat /proc/net/dev          # cumulative counters
sar -n DEV 1 10            # historical interface stats
```
**Fix:** Upgrade link, QoS prioritization, CDN offload, compression

#### 2. Packet Loss
```bash
# Detect
mtr --report -c 100 8.8.8.8   # path-level loss
ping -c 1000 gateway           # local loss test
ip -s link show eth0           # NIC error counters
ethtool -S eth0                # detailed NIC stats
```
**Fix:** Replace bad cables/hardware, fix duplex mismatch, fix buffer overflow

#### 3. High Latency
```bash
# Locate the slow hop
mtr -n google.com
traceroute -T -p 443 google.com
# DNS latency
time dig google.com
# Application latency
curl -w "@curl-format.txt" -o /dev/null https://api.example.com
```
**Fix:** Use CDN, closer region, DNS caching, connection pooling, keep-alive

#### 4. TCP Retransmissions
```bash
# Check retransmission count
ss -ti | grep retrans
netstat -s | grep retransmit
# Watch in real-time
watch -n1 'netstat -s | grep retransmit'
```
**Fix:** Fix packet loss upstream, tune TCP buffers, optimize window size

#### 5. DNS Performance
```bash
time dig google.com             # baseline
time dig @8.8.8.8 google.com   # test specific resolver
resolvectl statistics           # cache hit rate
```
**Fix:** Add local DNS cache (unbound/dnsmasq), tune ndots, use faster resolver

#### 6. MTU Issues (Black Hole)
```bash
ping -s 1472 -M do 8.8.8.8     # test standard MTU (1500)
tracepath google.com            # MTU-aware trace
```
**Fix:** Clamp MSS with iptables, reduce interface MTU, enable PMTUD

### QoS (Quality of Service)

QoS prioritizes network traffic to ensure critical applications get bandwidth first.

#### Traffic Classes (DSCP)
```
DSCP Value | Traffic Class | Example
EF (46)    | Expedited Forwarding (highest) | VoIP, video conferencing
AF41 (34)  | Assured Forwarding 4           | Video streaming
AF31 (26)  | Assured Forwarding 3           | Business-critical data
AF21 (18)  | Assured Forwarding 2           | Transactional data
BE (0)     | Best Effort (default)          | General web browsing
CS1 (8)    | Background/scavenger           | Backups, bulk transfers
```

#### Linux Traffic Control (tc)
```bash
# Create a queueing discipline (HTB - Hierarchical Token Bucket)
sudo tc qdisc add dev eth0 root handle 1: htb default 30

# Create classes with bandwidth guarantees
sudo tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit burst 15k

# High priority: 30 Mbps minimum, 100 Mbps ceiling
sudo tc class add dev eth0 parent 1:1 classid 1:10 htb rate 30mbit ceil 100mbit burst 15k

# Normal: 60 Mbps
sudo tc class add dev eth0 parent 1:1 classid 1:20 htb rate 60mbit ceil 100mbit burst 15k

# Background: 10 Mbps
sudo tc class add dev eth0 parent 1:1 classid 1:30 htb rate 10mbit ceil 100mbit burst 15k

# Add fair queuing to each class
sudo tc qdisc add dev eth0 parent 1:10 handle 10: sfq perturb 10
sudo tc qdisc add dev eth0 parent 1:20 handle 20: sfq perturb 10
sudo tc qdisc add dev eth0 parent 1:30 handle 30: sfq perturb 10

# Classify traffic (SSH gets high priority)
sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
  match ip dport 22 0xffff flowid 1:10

# View current qdisc
tc -s qdisc show dev eth0
tc -s class show dev eth0
```

#### AWS VPC Traffic Mirroring
Capture and inspect traffic from ENIs without installing agents:
```bash
# Create traffic mirror filter (what to capture)
FILTER_ID=$(aws ec2 create-traffic-mirror-filter \
  --query 'TrafficMirrorFilter.TrafficMirrorFilterId' --output text)

# Add rule: capture all TCP traffic
aws ec2 create-traffic-mirror-filter-rule \
  --traffic-mirror-filter-id $FILTER_ID \
  --traffic-direction ingress \
  --rule-number 100 \
  --rule-action accept \
  --protocol 6  # TCP

# Create mirror session (source → target)
aws ec2 create-traffic-mirror-session \
  --network-interface-id eni-source \
  --traffic-mirror-target-id tmt-target \
  --traffic-mirror-filter-id $FILTER_ID \
  --session-number 1
```

### Cloud Network Monitoring

#### AWS CloudWatch for Networking

**EC2 Network Metrics:**
```bash
# AWS CLI: Get EC2 network metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkIn \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2026-05-10T00:00:00Z \
  --end-time 2026-05-10T12:00:00Z \
  --period 300 \
  --statistics Average,Maximum
```

**VPC Flow Log Metrics (CloudWatch Logs Insights):**
```
# Top talkers by bytes
fields srcAddr, dstAddr, bytes
| filter action = "ACCEPT"
| stats sum(bytes) as totalBytes by srcAddr
| sort totalBytes desc
| limit 10

# Rejected connections by destination port
fields dstPort, srcAddr
| filter action = "REJECT"
| stats count() as rejectCount by dstPort
| sort rejectCount desc
| limit 20

# Network utilization over time
fields bytes, start, end
| filter interfaceId = "eni-12345678"
| stats sum(bytes) by bin(5m)
```

**CloudWatch Alarms for Network:**
```json
{
  "AlarmName": "HighNetworkIn",
  "MetricName": "NetworkIn",
  "Namespace": "AWS/EC2",
  "Statistic": "Average",
  "Period": 300,
  "EvaluationPeriods": 3,
  "Threshold": 100000000,
  "ComparisonOperator": "GreaterThanThreshold",
  "AlarmActions": ["arn:aws:sns:us-east-1:123:network-alerts"]
}
```

#### GCP Cloud Monitoring (Network)
```bash
# GCP: List network metrics
gcloud monitoring metrics list --filter="metric.type:networking"

# Create uptime check (availability monitoring)
gcloud monitoring uptime-check-configs create \
  --display-name="API Health Check" \
  --http-check-path="/health" \
  --http-check-port=443 \
  --monitored-resource-type=uptime_url \
  --checked-resource-uri="https://api.example.com/health" \
  --period=60
```

#### Azure Monitor Network Insights
```bash
# Azure CLI: Get VM network metrics
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm} \
  --metric "Network In Total" "Network Out Total" \
  --interval PT5M \
  --start-time 2026-05-10T00:00:00Z \
  --end-time 2026-05-10T12:00:00Z
```

### Network Performance Tuning

#### Linux Kernel TCP Tuning
```bash
# Increase TCP buffer sizes (for high-bandwidth, high-latency paths)
sysctl -w net.core.rmem_max=134217728      # 128 MB max receive buffer
sysctl -w net.core.wmem_max=134217728      # 128 MB max send buffer
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# Enable window scaling
sysctl -w net.ipv4.tcp_window_scaling=1

# Use BBR congestion control (better for cloud)
sysctl -w net.ipv4.tcp_congestion_control=bbr

# Increase connection queue
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# Reduce TIME_WAIT
sysctl -w net.ipv4.tcp_fin_timeout=15
sysctl -w net.ipv4.tcp_tw_reuse=1

# Make persistent
cat >> /etc/sysctl.conf << 'EOF'
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.ipv4.tcp_congestion_control = bbr
net.core.somaxconn = 65535
EOF
sysctl -p
```

---

## Hands-On Labs

### Lab 1: Network Performance Baseline
```bash
#!/bin/bash
echo "Network Performance Baseline"
echo "============================="
DATE=$(date '+%Y-%m-%d %H:%M:%S')
echo "Timestamp: $DATE"

# Interface stats
echo -e "\n[Interface Statistics]"
for iface in $(ls /sys/class/net | grep -v lo); do
    rx=$(cat /sys/class/net/$iface/statistics/rx_bytes 2>/dev/null || echo 0)
    tx=$(cat /sys/class/net/$iface/statistics/tx_bytes 2>/dev/null || echo 0)
    rx_err=$(cat /sys/class/net/$iface/statistics/rx_errors 2>/dev/null || echo 0)
    tx_err=$(cat /sys/class/net/$iface/statistics/tx_errors 2>/dev/null || echo 0)
    drops=$(cat /sys/class/net/$iface/statistics/rx_dropped 2>/dev/null || echo 0)
    printf "  %-10s RX: %-12s TX: %-12s Errors: %s/%s Drops: %s\n" \
        "$iface" "${rx}B" "${tx}B" "$rx_err" "$tx_err" "$drops"
done

# Latency tests
echo -e "\n[Latency Tests]"
for host in "8.8.8.8" "1.1.1.1" "$(ip route | awk '/default/{print $3}' | head -1)"; do
    [ -z "$host" ] && continue
    result=$(ping -c 5 -W 2 $host 2>/dev/null | tail -1 | awk -F'/' '{printf "avg=%.1fms min=%.1fms max=%.1fms", $5, $4, $6}')
    echo "  $host: ${result:-timeout}"
done

# TCP stats
echo -e "\n[TCP Statistics]"
echo "  Established: $(ss -t state established | wc -l)"
echo "  Time-wait:   $(ss -t state time-wait | wc -l)"
echo "  Listen:      $(ss -l | wc -l)"
retrans=$(netstat -s 2>/dev/null | grep "segments retransmited" | awk '{print $1}' || echo "N/A")
echo "  Retransmissions: $retrans"

# DNS performance
echo -e "\n[DNS Performance]"
start=$(date +%s%N)
dig +short google.com > /dev/null 2>&1
end=$(date +%s%N)
dns_ms=$(( (end - start) / 1000000 ))
echo "  Resolution time: ${dns_ms}ms"
```

### Lab 2: Bandwidth Test with iperf3
```bash
#!/bin/bash
# iperf3 bandwidth testing

if ! command -v iperf3 &>/dev/null; then
    echo "Installing iperf3..."
    sudo apt-get install -y iperf3 2>/dev/null
fi

echo "iperf3 Network Bandwidth Test"
echo "=============================="

# Local loopback test (baseline without network)
echo -e "\n[Local Loopback Baseline]"
iperf3 -s -D --logfile /tmp/iperf3.log 2>/dev/null
sleep 0.5
iperf3 -c 127.0.0.1 -t 5 -J 2>/dev/null | python3 -c "
import json, sys
try:
    data = json.load(sys.stdin)
    bps = data['end']['sum_received']['bits_per_second']
    mbps = bps / 1_000_000
    retrans = data['end']['sum_sent']['retransmits']
    print(f'  Throughput: {mbps:.1f} Mbps')
    print(f'  Retransmits: {retrans}')
except:
    print('  Could not parse iperf3 output')
"
pkill iperf3 2>/dev/null

# Test to a public iperf3 server
echo -e "\n[Public iperf3 Server Test (iperf.he.net)]"
timeout 15 iperf3 -c iperf.he.net -t 5 2>/dev/null | grep -E "sender|receiver" | head -4 || \
    echo "  Public server test failed or timed out"

echo -e "\n[UDP Jitter Test (loopback)]"
iperf3 -s -D --logfile /tmp/iperf3.log 2>/dev/null
sleep 0.5
iperf3 -c 127.0.0.1 -u -b 100M -t 5 2>/dev/null | grep -E "jitter|loss" | head -3
pkill iperf3 2>/dev/null
```

### Lab 3: Network Monitoring Dashboard
```python
#!/usr/bin/env python3
import time, subprocess, json, os

def get_interface_stats(iface):
    stats = {}
    base = f"/sys/class/net/{iface}/statistics"
    for metric in ['rx_bytes', 'tx_bytes', 'rx_errors', 'tx_errors', 'rx_dropped', 'tx_dropped']:
        try:
            with open(f"{base}/{metric}") as f:
                stats[metric] = int(f.read().strip())
        except:
            stats[metric] = 0
    return stats

def get_connections():
    result = subprocess.run(['ss', '-s'], capture_output=True, text=True)
    return result.stdout

def get_top_connections(n=5):
    result = subprocess.run(['ss', '-tn', 'state', 'established'], capture_output=True, text=True)
    lines = result.stdout.strip().split('\n')[1:]  # skip header
    return lines[:n]

def bytes_to_human(b):
    for unit in ['B', 'KB', 'MB', 'GB']:
        if b < 1024:
            return f"{b:.1f}{unit}"
        b /= 1024
    return f"{b:.1f}TB"

def monitor(interval=5, iterations=6):
    interfaces = [i for i in os.listdir('/sys/class/net') if i != 'lo']
    prev = {iface: get_interface_stats(iface) for iface in interfaces}

    print("Network Performance Monitor")
    print("=" * 60)
    print(f"{'Interface':<12}{'RX Rate':>12}{'TX Rate':>12}{'Errors':>8}{'Drops':>8}")

    for _ in range(iterations):
        time.sleep(interval)
        print(f"\n[{time.strftime('%H:%M:%S')}]")
        for iface in interfaces:
            curr = get_interface_stats(iface)
            rx_rate = (curr['rx_bytes'] - prev[iface]['rx_bytes']) / interval
            tx_rate = (curr['tx_bytes'] - prev[iface]['tx_bytes']) / interval
            errors = curr['rx_errors'] + curr['tx_errors']
            drops = curr['rx_dropped'] + curr['tx_dropped']
            print(f"  {iface:<10}{bytes_to_human(rx_rate)+'/s':>12}{bytes_to_human(tx_rate)+'/s':>12}{errors:>8}{drops:>8}")
            prev[iface] = curr

        conns = get_top_connections()
        if conns:
            print(f"  Top connections: {len(conns)}")
            for conn in conns[:3]:
                print(f"    {conn}")

monitor(interval=3, iterations=4)
```

---

## Sample Exercises

1. **Bottleneck identification:** CPU is at 5%, memory at 30%, but response times are high. `ss -ti` shows high retransmission counts. What is the root cause and how do you investigate?
2. **QoS design:** Design a QoS policy for a branch office with: 10 Mbps uplink, VoIP (requires 1 Mbps, <20ms jitter), video conferencing (3 Mbps), business app (3 Mbps), internet browsing (remaining).
3. **CloudWatch alarm:** Write the parameters for a CloudWatch alarm that triggers when average NetworkIn across an Auto Scaling Group exceeds 80% of capacity for 10 minutes.
4. **TCP BBR:** What is TCP BBR and why is it preferred over the default CUBIC in cloud environments?
5. **MTU black hole:** Users report that large file downloads work but small HTTP requests fail randomly. Could this be an MTU issue? How would you diagnose?

## Solutions

1. **High retransmissions root cause:** Retransmissions indicate **packet loss**. High retransmissions without CPU/RAM pressure suggest network-layer issue. Investigate: `mtr --report -c 100 <remote_host>` — identify which hop has loss; check NIC errors with `ip -s link show eth0`; check for duplex mismatch with `ethtool eth0`; inspect if NAT table is full; check if buffer overflow is causing drops (`ip -s link show` rx_dropped counter).

2. **QoS policy (10 Mbps uplink):**
   ```
   Class 1 (EF/High): VoIP — guaranteed 1.5 Mbps, ceil 2 Mbps, priority 1
   Class 2 (AF41): Video conferencing — guaranteed 3 Mbps, ceil 6 Mbps, priority 2
   Class 3 (AF31): Business app — guaranteed 3 Mbps, ceil 8 Mbps, priority 3
   Class 4 (BE): Web browsing — best effort, remaining bandwidth
   Total guaranteed: 7.5 Mbps (headroom for bursting)
   ```

3. **CloudWatch alarm parameters:**
   - Namespace: `AWS/EC2`
   - Metric: `NetworkIn`
   - Dimensions: `AutoScalingGroupName`
   - Statistic: `Average`
   - Period: `300` (5 minutes)
   - EvaluationPeriods: `2` (2 × 5min = 10 min)
   - Threshold: `(link_speed_bps × 0.8)` — e.g., 80% of 10 Gbps = 8,000,000,000 bytes/s ... adjust to your instance type
   - ComparisonOperator: `GreaterThanThreshold`

4. **TCP BBR vs CUBIC:** CUBIC (default) uses packet loss as a signal of congestion — it backs off when it sees drops. In cloud environments with shallow buffers and high bandwidth-delay products, CUBIC can underutilize bandwidth. **BBR (Bottleneck Bandwidth and RTT)** models the actual bottleneck bandwidth using RTT measurements rather than loss — it maintains higher throughput, is more efficient in cloud networks, and is less aggressive in competing with other flows. Enable with `sysctl net.ipv4.tcp_congestion_control=bbr`.

5. **MTU black hole diagnosis:** Yes, this is classic MTU black hole behavior — small requests (small packets) succeed; large file downloads (large packets) that hit the MTU limit fail or hang. Diagnose with: `ping -s 1472 -M do <remote>` — if 1472-byte packets fail but 1000-byte succeed, the MTU is too small somewhere in the path. Use `tracepath <remote>` to see where the MTU drops. Fix: `iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu` or manually set MTU lower on the interface.

## Completion Checklist
- [ ] Identify the 6 common network bottleneck types and their tools
- [ ] Configure Linux tc for traffic shaping and QoS
- [ ] Use CloudWatch, Cloud Monitoring, and Azure Monitor for network metrics
- [ ] Tune Linux TCP stack parameters for performance
- [ ] Write CloudWatch Logs Insights queries for VPC Flow Logs
- [ ] Build a continuous network monitoring script

## Key Takeaways
- Always establish a performance baseline before optimizing
- Packet loss and retransmissions are the most common causes of poor TCP performance
- QoS ensures critical traffic (VoIP, video) gets bandwidth even under congestion
- Linux tc provides powerful traffic shaping but requires careful tuning
- Cloud-native monitoring (VPC Flow Logs, CloudWatch) provides deep visibility without agents
- TCP BBR improves cloud network utilization significantly over CUBIC

## Next Steps
Proceed to [Day 29: Real-World Scenarios & Interview Questions](../Day_29/notes_and_exercises.md).