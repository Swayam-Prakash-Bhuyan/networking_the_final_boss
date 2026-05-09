# Day 16: VPNs, Tunnels, SSH, WireGuard, OpenVPN, Cloud VPNs

## Learning Objectives
By the end of Day 16, you will:
- Understand VPN concepts, types, and protocols
- Configure SSH tunnels (local, remote, dynamic)
- Set up WireGuard and OpenVPN connections
- Use cloud VPN services (AWS VPN, Azure VPN Gateway, GCP Cloud VPN)

**Estimated Time:** 3-4 hours

## Notes

### VPN Fundamentals
A VPN (Virtual Private Network) creates an **encrypted tunnel** over a public network, allowing private communication as if devices were on the same local network.

**Key benefits:**
- Encrypt traffic over untrusted networks
- Extend private networks across the internet
- Site-to-site connectivity (branch offices, cloud)
- Remote access for employees

### VPN Types

#### Site-to-Site VPN
- Connects two or more entire networks
- Always-on tunnel between gateways
- Use case: HQ ↔ Branch office, On-prem ↔ Cloud

#### Remote Access VPN
- Connects individual users to a private network
- Client software on endpoint
- Use case: Work-from-home employees

#### Client-to-Site VPN
- Similar to Remote Access VPN
- Examples: OpenVPN, WireGuard, Cisco AnyConnect

### VPN Protocols

| Protocol | Encryption | Port | Speed | Use Case |
|----------|-----------|------|-------|----------|
| IPSec/IKEv2 | AES-256 | UDP 500, 4500 | Fast | Site-to-site, mobile |
| OpenVPN | AES-256 | TCP/UDP 1194 | Moderate | Remote access |
| WireGuard | ChaCha20 | UDP 51820 | Very fast | Modern VPN |
| L2TP/IPSec | AES-256 | UDP 1701 | Moderate | Legacy |
| PPTP | MPPE | TCP 1723 | Fast | Insecure, legacy |
| SSL/TLS VPN | AES-256 | TCP 443 | Moderate | Firewall-friendly |

### IPSec Architecture
```
IKE Phase 1: Authenticate peers, establish secure channel
  → Main mode or Aggressive mode
  → Diffie-Hellman key exchange

IKE Phase 2: Negotiate encryption for data tunnel
  → Quick mode
  → Define traffic selectors (which traffic to encrypt)

ESP (Encapsulating Security Payload): Data encryption
AH (Authentication Header): Integrity only (no encryption)

Modes:
  Transport mode: Encrypts payload only (host-to-host)
  Tunnel mode:    Encrypts entire packet (gateway-to-gateway)
```

### SSH Tunneling

SSH tunnels use an encrypted SSH connection to forward ports.

#### Local Port Forwarding
"I want to access a remote service through my local machine"
```bash
# Forward local port 8080 → remote_host:80 via ssh_server
ssh -L 8080:remote_host:80 user@ssh_server

# Example: Access internal web app via jump host
ssh -L 8080:internal-app.corp:80 user@jump.corp.com
# Now browse: http://localhost:8080
```

#### Remote Port Forwarding
"I want the remote server to be able to reach my local service"
```bash
# Forward ssh_server:9090 → localhost:3000
ssh -R 9090:localhost:3000 user@ssh_server
```

#### Dynamic Port Forwarding (SOCKS Proxy)
"I want to route all browser traffic through a remote server"
```bash
# Create SOCKS5 proxy on local port 1080
ssh -D 1080 user@ssh_server
# Configure browser to use SOCKS5 proxy: 127.0.0.1:1080
```

#### Jump Hosts / ProxyJump
```bash
# Connect to target through jump host
ssh -J user@jump_host user@target_host

# Multi-hop jump
ssh -J user@jump1,user@jump2 user@target

# In ~/.ssh/config:
Host target
    HostName 10.0.0.50
    User ubuntu
    ProxyJump bastion.example.com
```

### WireGuard

WireGuard is a modern, simple, high-performance VPN protocol built into the Linux kernel.

**Key features:**
- Only ~4000 lines of code (auditable)
- Uses modern cryptography (ChaCha20, Curve25519, BLAKE2)
- Roaming support (no connection drops when IP changes)
- Always UDP; no TCP mode

#### WireGuard Setup (Server)
```bash
# Install
sudo apt install wireguard -y

# Generate key pair
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key

# Create server config: /etc/wireguard/wg0.conf
cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
PrivateKey = $(cat /etc/wireguard/server_private.key)
Address = 10.100.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = CLIENT_PUBLIC_KEY_HERE
AllowedIPs = 10.100.0.2/32
EOF

# Enable and start
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0

# View status
sudo wg show
```

#### WireGuard Client Config
```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.100.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = server.example.com:51820
AllowedIPs = 0.0.0.0/0          # Route all traffic
# AllowedIPs = 10.100.0.0/24   # Split tunnel (VPN only for this range)
PersistentKeepalive = 25
```

### OpenVPN

OpenVPN is mature, flexible, and widely supported.

```bash
# Install
sudo apt install openvpn easy-rsa -y

# Server: Initialize PKI
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey secret ta.key

# Minimal server.conf
cat > /etc/openvpn/server.conf <<EOF
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 120
cipher AES-256-CBC
EOF

sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

### Cloud VPN Services

#### AWS Site-to-Site VPN
```
On-premises ←──IPSec Tunnel──→ AWS Virtual Private Gateway (VGW)
                                        ↓
                                   VPC Route Table
```
- Two tunnels per connection (redundancy)
- Supports BGP or static routing
- Up to 1.25 Gbps per tunnel
- AWS Transit Gateway supports scale-out to multiple VPCs

#### AWS Client VPN
- OpenVPN-compatible managed endpoint
- Mutual TLS authentication or AD/SAML
- Scales automatically

#### Azure VPN Gateway
- Route-based or policy-based VPN
- SKUs from Basic to VpnGw5 (100 Gbps)
- Supports S2S, P2S, and VNet-to-VNet

#### GCP Cloud VPN
- HA VPN (99.99% SLA) with two tunnels
- Dynamic routing via Cloud Router (BGP)
- Classic VPN for static/policy-based routing

## Hands-On Labs

### Lab 1: SSH Tunnel Practice
```bash
# Lab: SSH local port forward to a remote web server
# Requirement: SSH access to a jump host

# Start a simple local web server to simulate a target
python3 -m http.server 8888 &
SERVER_PID=$!
echo "Local test server started on port 8888 (PID: $SERVER_PID)"

# In another terminal, simulate local forwarding:
# ssh -L 8080:localhost:8888 user@remote_host
# Then access: curl http://localhost:8080

# Test without tunnel (direct)
curl -s http://localhost:8888 | head -5

# Cleanup
kill $SERVER_PID
```

### Lab 2: WireGuard Quick Demo
```bash
#!/bin/bash
# WireGuard key generation demo (no live VPN needed)

echo "WireGuard Key Management Demo"
echo "=============================="

if ! command -v wg &>/dev/null; then
    echo "Install WireGuard: sudo apt install wireguard"
    exit 0
fi

# Generate server keys
SERVER_PRIVATE=$(wg genkey)
SERVER_PUBLIC=$(echo "$SERVER_PRIVATE" | wg pubkey)

# Generate client keys
CLIENT_PRIVATE=$(wg genkey)
CLIENT_PUBLIC=$(echo "$CLIENT_PRIVATE" | wg pubkey)
CLIENT_PSK=$(wg genpsk)

echo "Server Private Key: ${SERVER_PRIVATE:0:10}..."
echo "Server Public Key:  $SERVER_PUBLIC"
echo ""
echo "Client Private Key: ${CLIENT_PRIVATE:0:10}..."
echo "Client Public Key:  $CLIENT_PUBLIC"
echo ""

# Generate server config
echo "=== Server Config (wg0.conf) ==="
cat <<EOF
[Interface]
PrivateKey = $SERVER_PRIVATE
Address = 10.100.0.1/24
ListenPort = 51820

[Peer]
PublicKey = $CLIENT_PUBLIC
PresharedKey = $CLIENT_PSK
AllowedIPs = 10.100.0.2/32
EOF

echo ""
echo "=== Client Config (client.conf) ==="
cat <<EOF
[Interface]
PrivateKey = $CLIENT_PRIVATE
Address = 10.100.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = $SERVER_PUBLIC
PresharedKey = $CLIENT_PSK
Endpoint = YOUR_SERVER_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF
```

### Lab 3: VPN Connectivity Checker
```bash
#!/bin/bash
# Check VPN interface and connectivity

echo "VPN Status Check"
echo "================"

# Check for WireGuard interfaces
if ip link show | grep -q "wg"; then
    echo "WireGuard interfaces:"
    ip addr show | grep -A3 "wg"
    sudo wg show 2>/dev/null
fi

# Check for tun/tap (OpenVPN)
if ip link show | grep -q "tun"; then
    echo "OpenVPN/TUN interfaces:"
    ip addr show | grep -A3 "tun"
fi

# Check VPN-related ports
echo -e "\nVPN Ports Listening:"
ss -ulnp | grep -E "51820|1194|500|4500"

# Check routing for VPN subnets
echo -e "\nVPN Routes:"
ip route show | grep -E "10\.|172\.|192\.168\." | grep -v "src"
```

## Practical Exercises

### Exercise 1: SSH Config for Jump Hosts
```bash
# Create ~/.ssh/config for multi-hop tunneling

mkdir -p ~/.ssh
cat > ~/.ssh/config <<'EOF'
# Bastion / jump host
Host bastion
    HostName bastion.example.com
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60

# Internal server via bastion
Host internal-web
    HostName 10.0.1.50
    User ubuntu
    IdentityFile ~/.ssh/id_rsa
    ProxyJump bastion

# Dev environment with local port forward
Host dev-db
    HostName 10.0.2.100
    User ubuntu
    ProxyJump bastion
    LocalForward 5432 localhost:5432  # Forward remote Postgres locally

# SOCKS proxy for browsing internal sites
Host socks-proxy
    HostName bastion.example.com
    User ubuntu
    DynamicForward 1080
    ServerAliveInterval 60
EOF

echo "SSH config created. Usage:"
echo "  ssh internal-web   → direct hop to internal server"
echo "  ssh dev-db         → with local Postgres forwarded on :5432"
echo "  ssh socks-proxy    → SOCKS5 proxy on localhost:1080"
```

### Exercise 2: VPN Latency Comparison
```python
#!/usr/bin/env python3
import subprocess
import time

def measure_latency(host, count=5):
    """Measure round-trip latency"""
    try:
        result = subprocess.run(
            ['ping', '-c', str(count), '-W', '2', host],
            capture_output=True, text=True, timeout=30
        )
        lines = result.stdout.split('\n')
        for line in lines:
            if 'rtt min' in line or 'round-trip' in line:
                return line.strip()
        return "No stats available"
    except Exception as e:
        return f"Error: {e}"

# Compare latency for different scenarios
print("VPN Performance Analysis")
print("=" * 45)

targets = {
    "Local gateway (192.168.1.1)": "192.168.1.1",
    "Google DNS (direct)": "8.8.8.8",
    "Cloudflare DNS": "1.1.1.1",
}

for name, host in targets.items():
    print(f"\n{name}:")
    result = measure_latency(host)
    print(f"  {result}")
    
print("\nNote: Compare these before/after VPN to measure VPN overhead")
```

### Exercise 3: Tunnel Health Monitor
```bash
#!/bin/bash
# Monitor VPN tunnel health

TUNNEL_HOST=${1:-"10.100.0.1"}   # VPN gateway IP
CHECK_INTERVAL=${2:-30}

echo "VPN Tunnel Health Monitor"
echo "Target: $TUNNEL_HOST"
echo "Interval: ${CHECK_INTERVAL}s"
echo "=========================="

fail_count=0
while true; do
    timestamp=$(date '+%H:%M:%S')
    
    if ping -c 1 -W 2 $TUNNEL_HOST >/dev/null 2>&1; then
        latency=$(ping -c 3 -W 2 $TUNNEL_HOST | tail -1 | awk -F '/' '{print $5}')
        echo "[$timestamp] ✓ Tunnel UP  | Latency: ${latency}ms"
        fail_count=0
    else
        fail_count=$((fail_count + 1))
        echo "[$timestamp] ✗ Tunnel DOWN | Consecutive failures: $fail_count"
        
        if [ $fail_count -ge 3 ]; then
            echo "[$timestamp] ALERT: 3+ consecutive failures — tunnel may be down!"
        fi
    fi
    
    sleep $CHECK_INTERVAL
done
```

## Sample Exercises

1. **Protocol Choice:** You need a VPN for 500 remote workers and a site-to-site tunnel to AWS. Which protocols would you choose and why?

2. **SSH Tunnel Design:** A developer needs access to a database on a private subnet (10.0.2.100:5432) through a bastion host (bastion.corp.com). Write the exact SSH command.

3. **WireGuard vs OpenVPN:** List three situations where WireGuard is preferred over OpenVPN and three where OpenVPN is preferred.

4. **Cloud VPN:** Design an AWS site-to-site VPN connection with redundancy. What components are needed?

5. **Split Tunneling:** Explain split tunneling. When would you enable it and when would you disable it?

## Solutions

1. **Protocol Choices:**
   - **Remote workers (500):** AWS Client VPN (managed, scales) with OpenVPN-compatible clients or WireGuard-based solution. WireGuard is ideal for performance; OpenVPN for legacy client support.
   - **Site-to-site to AWS:** AWS Site-to-Site VPN (IPSec/IKEv2) with BGP via Virtual Private Gateway or Transit Gateway. Two tunnels for redundancy.

2. **SSH Tunnel Command:**
   ```bash
   ssh -L 5432:10.0.2.100:5432 user@bastion.corp.com
   # Then connect locally: psql -h localhost -p 5432 -U dbuser mydb
   ```

3. **WireGuard preferred:**
   - Performance-critical (gaming, video, high-bandwidth)
   - Mobile/roaming clients (handles IP changes gracefully)
   - Simple peer-to-peer mesh (no PKI required)
   
   **OpenVPN preferred:**
   - TCP mode needed (to traverse firewalls on port 443)
   - Large existing PKI/certificate infrastructure
   - Broad legacy client support required

4. **AWS Site-to-Site VPN Components:**
   - Customer Gateway (CGW): Represents your on-prem VPN device
   - Virtual Private Gateway (VGW): AWS-side endpoint (or Transit Gateway)
   - Site-to-Site VPN Connection: Two IPSec tunnels (automatic redundancy)
   - Route tables: Updated to route on-prem CIDRs via VGW
   - BGP (optional): Dynamic route exchange for failover

5. **Split Tunneling:**
   - **Definition:** Only traffic to specific private subnets goes through the VPN; internet traffic goes directly
   - **Enable when:** Performance matters; you trust the user's internet connection; VPN has limited bandwidth
   - **Disable (full tunnel) when:** Compliance/DLP requires all traffic inspected; users work with sensitive data; company security policy mandates it

## Completion Checklist
- [ ] Understand site-to-site vs remote access VPN
- [ ] Configure SSH local, remote, and dynamic port forwarding
- [ ] Generate WireGuard keys and write server/client configs
- [ ] Know the difference between WireGuard and OpenVPN
- [ ] Understand cloud VPN options (AWS, Azure, GCP)
- [ ] Implement jump host configurations in ~/.ssh/config

## Key Takeaways
- VPNs encrypt traffic and extend private networks over public infrastructure
- SSH tunnels are powerful for quick, ad-hoc port forwarding without VPN software
- WireGuard is modern, fast, and simpler than OpenVPN — prefer it for new deployments
- Cloud VPN services abstract infrastructure while providing managed redundancy
- Split tunneling balances performance and security requirements

## Next Steps
Proceed to [Day 17: Load Balancing (L4/L7, HAProxy, Nginx, Cloud Load Balancers)](../Day_17/notes_and_exercises.md).