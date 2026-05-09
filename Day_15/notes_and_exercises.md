# Day 15: Firewalls (iptables, nftables, ufw, Security Groups, Cloud Firewalls)

## Learning Objectives
By the end of Day 15, you will:
- Master iptables chains, tables, and rule syntax
- Use nftables and ufw for simpler firewall management
- Configure AWS Security Groups and NACLs
- Design defense-in-depth firewall strategies

**Estimated Time:** 3-4 hours

## Notes

### Firewall Fundamentals

A firewall **filters network traffic** based on rules. Types:
- **Stateless (packet filter):** Inspects each packet independently (iptables, NACLs)
- **Stateful:** Tracks connection state; allows return traffic automatically (most modern firewalls)
- **Application-layer (L7):** Inspects payload content (WAF, next-gen firewalls)
- **Host-based:** Runs on the host (iptables, ufw, Windows Firewall)
- **Network-based:** Dedicated appliance (Cisco ASA, Palo Alto, pfSense)

### iptables

#### Tables and Chains
```
Table: filter (default)          Table: nat              Table: mangle
  Chain: INPUT                    Chain: PREROUTING        Chain: PREROUTING
  Chain: FORWARD                  Chain: OUTPUT            Chain: OUTPUT
  Chain: OUTPUT                   Chain: POSTROUTING       Chain: INPUT/FORWARD

Packet flow:
PREROUTING → [FORWARD] → POSTROUTING
             [INPUT → local process → OUTPUT]
```

#### iptables Rule Syntax
```bash
iptables -[A|I|D|R] CHAIN -p PROTO --dport PORT -s SRC -d DEST -j TARGET

Targets:
  ACCEPT  - Allow
  DROP    - Discard silently
  REJECT  - Discard with error
  LOG     - Log to syslog
  DNAT    - Destination NAT
  SNAT    - Source NAT
  MASQUERADE - Dynamic SNAT
```

#### Common iptables Rules
```bash
# View all rules with line numbers
sudo iptables -L -n -v --line-numbers

# Default policy: drop all incoming
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow established/related (stateful)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH from specific IP
sudo iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Allow ICMP (ping)
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Rate limit SSH (brute force protection)
sudo iptables -A INPUT -p tcp --dport 22 -m limit --limit 5/min --limit-burst 10 -j ACCEPT

# Log dropped packets
sudo iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPT DROP: " --log-level 4
sudo iptables -A INPUT -j DROP

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6
```

### nftables (Modern Replacement for iptables)
```bash
# Install
sudo apt install nftables -y

# View current ruleset
sudo nft list ruleset

# Create a basic ruleset
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
sudo nft add chain inet filter forward { type filter hook forward priority 0 \; policy drop \; }
sudo nft add chain inet filter output { type filter hook output priority 0 \; policy accept \; }

# Allow established
sudo nft add rule inet filter input ct state established,related accept

# Allow loopback
sudo nft add rule inet filter input iifname lo accept

# Allow SSH
sudo nft add rule inet filter input tcp dport 22 accept

# Allow HTTP/HTTPS
sudo nft add rule inet filter input tcp dport { 80, 443 } accept

# List rules
sudo nft list ruleset

# nftables config file: /etc/nftables.conf
```

### ufw (Uncomplicated Firewall)
`ufw` wraps iptables with a simpler interface.

```bash
# Enable/disable
sudo ufw enable
sudo ufw disable
sudo ufw status verbose

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow/deny services
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 10.0.0.0/8 to any port 22

# Allow by application profile
sudo ufw allow "Nginx Full"
sudo ufw app list

# Delete rules
sudo ufw delete allow 80/tcp
sudo ufw delete 3         # by rule number

# Rate limiting
sudo ufw limit ssh

# Logging
sudo ufw logging on
```

### AWS Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance/ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Rule evaluation | All rules evaluated | Rules evaluated in order |
| Default | Deny all in, allow all out | Allow all |
| Scope | Per-instance | Per-subnet |

#### Security Group Rules
```json
{
  "GroupName": "web-servers",
  "Inbound": [
    {"Protocol": "tcp", "Port": 443, "Source": "0.0.0.0/0"},
    {"Protocol": "tcp", "Port": 80, "Source": "0.0.0.0/0"},
    {"Protocol": "tcp", "Port": 22, "Source": "10.0.0.0/8"}
  ],
  "Outbound": [
    {"Protocol": "-1", "Port": "All", "Destination": "0.0.0.0/0"}
  ]
}
```

#### NACL Rules
```
Rule# | Protocol | Port  | Source      | Action
100   | TCP      | 443   | 0.0.0.0/0   | ALLOW
110   | TCP      | 80    | 0.0.0.0/0   | ALLOW
120   | TCP      | 22    | 10.0.0.0/8  | ALLOW
130   | TCP      | 1024-65535 | 0.0.0.0/0 | ALLOW  ← return traffic!
*     | ALL      | ALL   | 0.0.0.0/0   | DENY
```

### Firewall Design Principles
1. **Least privilege:** Only allow what is explicitly needed
2. **Default deny:** Block everything, then open selectively
3. **Defense in depth:** Multiple layers (host + network + cloud)
4. **Stateful tracking:** Track connections to allow return traffic
5. **Logging:** Log drops for security auditing
6. **Review regularly:** Audit and remove stale rules

## Hands-On Labs

### Lab 1: iptables Firewall Build
```bash
#!/bin/bash
# Build a complete host firewall with iptables

echo "Configuring iptables firewall..."

# Flush all existing rules
sudo iptables -F
sudo iptables -X

# Default: drop incoming, allow outgoing
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow ICMP (ping)
sudo iptables -A INPUT -p icmp -j ACCEPT

# Allow SSH (rate limited)
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m limit --limit 3/min --limit-burst 5 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Log and drop everything else
sudo iptables -A INPUT -m limit --limit 10/min \
  -j LOG --log-prefix "FIREWALL DROP: " --log-level 7
sudo iptables -A INPUT -j DROP

echo "Firewall configured. Current rules:"
sudo iptables -L -n -v
```

### Lab 2: ufw Rapid Setup
```bash
# Reset and configure ufw
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Essential services
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Restrict access to sensitive port
sudo ufw allow from 192.168.0.0/16 to any port 8080

# Enable
sudo ufw --force enable
sudo ufw status numbered
```

### Lab 3: Firewall Audit Script
```bash
#!/bin/bash
# Audit current firewall configuration

echo "=== Firewall Audit Report ==="
echo "Date: $(date)"

echo -e "\n--- iptables Rules ---"
sudo iptables -L -n -v 2>/dev/null | head -30

echo -e "\n--- nftables Ruleset ---"
sudo nft list ruleset 2>/dev/null | head -20

echo -e "\n--- ufw Status ---"
sudo ufw status verbose 2>/dev/null

echo -e "\n--- Open Ports (listening) ---"
ss -tlnp | grep LISTEN

echo -e "\n--- Services That Should Be Firewalled ---"
for port in 3306 5432 6379 27017 11211 2181; do
    if ss -tlnp | grep -q ":$port "; then
        echo "  WARNING: Port $port is listening — ensure firewall rule exists"
    fi
done
```

## Practical Exercises

### Exercise 1: Security Group Simulator
```python
#!/usr/bin/env python3

class SecurityGroup:
    def __init__(self, name):
        self.name = name
        self.inbound = []
        self.outbound = []
    
    def add_inbound(self, protocol, port, source, description=""):
        self.inbound.append({"proto": protocol, "port": port, "src": source, "desc": description})
    
    def add_outbound(self, protocol, port, dest, description=""):
        self.outbound.append({"proto": protocol, "port": port, "dst": dest, "desc": description})
    
    def check_inbound(self, protocol, port, source_ip):
        import ipaddress
        for rule in self.inbound:
            proto_ok = rule["proto"] in [protocol, "-1", "all"]
            port_ok = str(rule["port"]) == str(port) or rule["port"] == "all"
            try:
                src_ok = ipaddress.IPv4Address(source_ip) in ipaddress.IPv4Network(rule["src"])
            except:
                src_ok = rule["src"] in ["0.0.0.0/0", source_ip]
            
            if proto_ok and port_ok and src_ok:
                return True, f"ALLOW by rule: {rule['desc']}"
        return False, "DENY (no matching rule)"
    
    def display(self):
        print(f"\nSecurity Group: {self.name}")
        print("Inbound rules:")
        for r in self.inbound:
            print(f"  {r['proto']:5} port {r['port']:6} from {r['src']:<20} [{r['desc']}]")

# Build web server security group
sg = SecurityGroup("web-sg")
sg.add_inbound("tcp", 443, "0.0.0.0/0", "HTTPS from internet")
sg.add_inbound("tcp", 80, "0.0.0.0/0", "HTTP from internet")
sg.add_inbound("tcp", 22, "10.0.0.0/8", "SSH from private network")
sg.add_inbound("icmp", "all", "10.0.0.0/8", "Ping from internal")

sg.display()

print("\nTraffic checks:")
tests = [
    ("tcp", 443, "203.0.113.5"),
    ("tcp", 22, "203.0.113.5"),
    ("tcp", 22, "10.5.1.100"),
    ("tcp", 3306, "0.0.0.0"),
]
for proto, port, src in tests:
    allowed, reason = sg.check_inbound(proto, port, src)
    status = "✓ ALLOW" if allowed else "✗ DENY"
    print(f"  {proto} port {port} from {src:<20} → {status} ({reason})")
```

### Exercise 2: Port Security Scanner
```bash
#!/bin/bash
# Check which open ports have firewall rules

echo "Port Security Audit"
echo "==================="

# Get listening ports
while IFS= read -r line; do
    port=$(echo "$line" | awk '{print $4}' | awk -F: '{print $NF}')
    proc=$(echo "$line" | awk '{print $6}')
    
    if [ -n "$port" ] && [ "$port" -gt 0 ] 2>/dev/null; then
        # Check if iptables has a rule for this port
        if sudo iptables -L INPUT -n | grep -q "dpt:$port"; then
            echo "  ✓ Port $port: firewall rule exists ($proc)"
        else
            echo "  ⚠ Port $port: NO firewall rule found ($proc)"
        fi
    fi
done < <(ss -tlnp | grep LISTEN | grep -v "127.0.0.1")
```

## Sample Exercises

1. **Rule Writing:** Write iptables rules to: allow HTTP/HTTPS from anywhere, allow SSH only from 10.0.0.0/8, allow established traffic, drop everything else.

2. **Security Group Design:** Design Security Groups for a 3-tier app: ALB (public), App servers (private), RDS (private). Define which group talks to which.

3. **NACL vs SG:** A packet comes from 203.0.113.5 to port 443. The NACL explicitly denies it, but the Security Group allows it. Does it pass?

4. **Firewall Rule Conflict:** You have two rules: rule 100 allows TCP port 80, rule 50 denies all TCP. Which wins in iptables? In NACLs?

5. **Stateful vs Stateless:** Why do NACLs require explicit rules for return traffic (ports 1024-65535) while Security Groups don't?

## Solutions

1. **iptables Rules:**
   ```bash
   iptables -P INPUT DROP
   iptables -A INPUT -i lo -j ACCEPT
   iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
   iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT
   iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT
   ```

2. **3-Tier Security Groups:**
   ```
   alb-sg:   Inbound 80/443 from 0.0.0.0/0
   app-sg:   Inbound 8080 from alb-sg only; Inbound 22 from bastion-sg
   rds-sg:   Inbound 5432 from app-sg only
   All:      Outbound all allowed
   ```

3. **NACL vs SG:** The packet is **DENIED**. NACLs are evaluated first at the subnet level. If a NACL denies the traffic, it never reaches the instance or its Security Group.

4. **Rule Conflict:**
   - **iptables:** Rules evaluated in order; if rule 50 appears before rule 100, the DENY at rule 50 takes effect. First match wins.
   - **NACLs:** Same — evaluated in ascending rule number order; rule 50 (deny) fires before rule 100 (allow).

5. **Stateful vs Stateless:**
   - **Security Groups (stateful):** Track the TCP connection; when a request goes out on port 443, the SG remembers and automatically allows the return traffic regardless of its ephemeral port.
   - **NACLs (stateless):** Don't track state; each packet is evaluated independently. A request on port 443 is allowed by rule 100, but the server's reply comes from port 443 TO a random ephemeral port (1024-65535) on the client — this return packet must also be explicitly allowed.

## Completion Checklist
- [ ] Understand iptables tables, chains, and rule order
- [ ] Write iptables rules for common scenarios
- [ ] Use ufw for simplified firewall management
- [ ] Distinguish Security Groups from NACLs
- [ ] Design layered firewall strategies
- [ ] Audit a host's firewall configuration

## Key Takeaways
- Default-deny is the foundation of secure firewall design
- iptables evaluates rules top-to-bottom; first match wins
- Stateful firewalls automatically allow established session return traffic
- NACLs are stateless and require explicit rules for both directions
- Security Groups apply per-instance; NACLs apply per-subnet
- Defense in depth: combine host firewall + network ACL + cloud security group

## Next Steps
Proceed to [Day 16: VPNs, Tunnels, SSH, WireGuard, OpenVPN, Cloud VPNs](../Day_16/notes_and_exercises.md).