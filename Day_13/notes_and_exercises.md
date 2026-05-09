# Day 13: DNS Concepts, Cloud DNS, Troubleshooting

## Learning Objectives
By the end of Day 13, you will:
- Understand the DNS hierarchy and resolution process in depth
- Master all DNS record types and their use cases
- Configure local DNS with BIND or systemd-resolved
- Use cloud DNS services (Route 53, Cloud DNS, Azure DNS)
- Troubleshoot common DNS issues like NXDOMAIN, SERVFAIL, and propagation delays

**Estimated Time:** 3-4 hours

## Notes

### DNS Architecture

#### Hierarchical Structure
```
. (root)
├── com.
│   ├── google.com.
│   │   └── www.google.com.
│   └── example.com.
├── org.
│   └── wikipedia.org.
└── net.
    └── cloudflare.net.
```

#### DNS Server Types
| Type | Role | Example |
|------|------|---------|
| Root Nameserver | Knows TLD nameservers | a.root-servers.net |
| TLD Nameserver | Knows authoritative NS for domains | a.gtld-servers.net |
| Authoritative NS | Has actual DNS records | ns1.example.com |
| Recursive Resolver | Resolves on behalf of clients | 8.8.8.8, 1.1.1.1 |
| Caching Resolver | Caches responses locally | systemd-resolved |
| Forwarder | Forwards queries upstream | Pi-hole, Unbound |

#### Full Resolution Walk-through
```
Query: www.example.com → IP?

1. Client checks local cache → miss
2. Client checks /etc/hosts → miss
3. Client asks recursive resolver (8.8.8.8)
4. Resolver asks root: "who knows .com?"
   → Root: "Ask a.gtld-servers.net"
5. Resolver asks TLD: "who knows example.com?"
   → TLD: "Ask ns1.example.com"
6. Resolver asks authoritative: "www.example.com?"
   → Auth: "A 93.184.216.34" (TTL: 3600)
7. Resolver caches result and returns to client
8. Client connects to 93.184.216.34
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | `www → 93.184.216.34` |
| AAAA | IPv6 address | `www → 2606:2800::1` |
| CNAME | Alias | `blog → www.example.com` |
| MX | Mail server | `@ → 10 mail.example.com` |
| NS | Nameserver | `@ → ns1.example.com` |
| TXT | Text data | SPF, DKIM, verification |
| PTR | Reverse DNS | `34.216.184.93.in-addr.arpa → www.example.com` |
| SOA | Zone authority | Serial, refresh, retry, expire |
| SRV | Service location | `_http._tcp.example.com` |
| CAA | CA authorization | Limits which CAs can issue certs |

### TTL (Time to Live)
- TTL controls **how long resolvers cache a record**
- Low TTL (60s): Faster DNS changes, more queries
- High TTL (86400s): Fewer queries, slower propagation
- **Before major changes:** Lower TTL to 300s a day in advance
- **After propagation:** Raise TTL back to reduce load

### Zone File Example
```dns
$ORIGIN example.com.
$TTL 3600

@ IN SOA ns1.example.com. admin.example.com. (
    2024010101  ; Serial
    3600        ; Refresh
    900         ; Retry
    604800      ; Expire
    300 )       ; Minimum TTL

; Nameservers
@ IN NS ns1.example.com.
@ IN NS ns2.example.com.

; A Records
@ IN A 93.184.216.34
www IN A 93.184.216.34
mail IN A 93.184.216.100

; AAAA
www IN AAAA 2606:2800::1

; CNAME
blog IN CNAME www.example.com.
ftp IN CNAME www.example.com.

; MX
@ IN MX 10 mail.example.com.
@ IN MX 20 mail2.example.com.

; TXT (SPF)
@ IN TXT "v=spf1 include:_spf.google.com ~all"
```

### DNSSEC (DNS Security Extensions)
- Adds **cryptographic signatures** to DNS records
- Prevents DNS cache poisoning and MITM attacks
- Records: RRSIG, DNSKEY, DS, NSEC
- Root zone is DNSSEC-signed; propagates down the chain

### Cloud DNS Services

#### AWS Route 53
- **Public Hosted Zones:** DNS for internet-facing domains
- **Private Hosted Zones:** DNS within VPCs (internal names)
- **Routing Policies:** Simple, Weighted, Latency, Failover, Geolocation, Multivalue
- **Health Checks:** Integrated health monitoring for failover
- **Alias Records:** Route 53-specific; point to AWS resources (ELB, CloudFront, S3)

```bash
# Route 53 example with AWS CLI
aws route53 list-hosted-zones
aws route53 change-resource-record-sets --hosted-zone-id ZXXX \
  --change-batch file://record.json
```

#### Google Cloud DNS
- Anycast DNS with SLA
- Integrated with GCP IAM
- DNSSEC support
- 100% uptime SLA (no cold start)

#### Azure DNS
- Hosts DNS zones in Azure
- Supports private DNS zones for VNets
- RBAC integration
- Supports alias records pointing to Azure resources

### /etc/resolv.conf
```bash
# DNS resolver configuration
nameserver 8.8.8.8       # Primary DNS
nameserver 8.8.4.4       # Secondary DNS
search example.com       # Domain search list
options ndots:5          # Threshold for treating as FQDN
options timeout:2        # Query timeout in seconds
options attempts:3       # Retry attempts
```

### DNS Troubleshooting Tools

```bash
# Basic lookup
nslookup example.com
nslookup -type=MX example.com

# Detailed query with dig
dig example.com
dig example.com A
dig example.com MX
dig example.com +short
dig example.com +trace        # Full resolution path
dig @8.8.8.8 example.com      # Use specific resolver
dig -x 8.8.8.8                # Reverse lookup

# Check DNSSEC
dig example.com +dnssec

# Batch queries
dig +noall +answer example.com A AAAA MX

# Test zone transfer
dig axfr example.com @ns1.example.com

# systemd-resolve
resolvectl status
resolvectl query example.com
resolvectl flush-caches        # Clear local cache
```

## Hands-On Labs

### Lab 1: DNS Resolution Deep Dive
```bash
# Install dnsutils
sudo apt install dnsutils -y

# Trace full resolution
dig +trace www.google.com

# Check all record types
for type in A AAAA MX NS TXT SOA; do
    echo "=== $type ==="
    dig google.com $type +short
done

# Compare resolver responses
echo "=== Using 8.8.8.8 ==="
dig @8.8.8.8 google.com +short

echo "=== Using 1.1.1.1 ==="
dig @1.1.1.1 google.com +short

echo "=== Using local resolver ==="
dig google.com +short
```

### Lab 2: DNS Caching and TTL
```bash
# Check TTL of a record
dig google.com A | grep -A1 "ANSWER SECTION"

# Query twice to observe TTL countdown
dig @8.8.8.8 google.com A | grep "IN A"
sleep 5
dig @8.8.8.8 google.com A | grep "IN A"

# Flush systemd DNS cache
sudo resolvectl flush-caches
echo "Cache flushed"

# Check local cache status
resolvectl statistics 2>/dev/null || echo "resolvectl not available"
```

### Lab 3: Custom DNS Setup with hosts file
```bash
# View current hosts file
cat /etc/hosts

# Add a custom local DNS entry (temporary)
echo "127.0.0.1  myapp.local" | sudo tee -a /etc/hosts

# Test it
ping -c 2 myapp.local
nslookup myapp.local

# Remove it
sudo sed -i '/myapp.local/d' /etc/hosts
echo "Custom entry removed"
```

## Practical Exercises

### Exercise 1: DNS Audit Tool
```bash
#!/bin/bash
# DNS record auditor for a domain

DOMAIN=${1:-google.com}
echo "DNS Audit for: $DOMAIN"
echo "========================"

for rtype in A AAAA MX NS TXT SOA CAA; do
    result=$(dig $DOMAIN $rtype +short 2>/dev/null)
    if [ -n "$result" ]; then
        echo "[$rtype]"
        echo "$result" | sed 's/^/  /'
    fi
done

echo -e "\n[PTR / Reverse DNS for A records]"
for ip in $(dig $DOMAIN A +short); do
    ptr=$(dig -x $ip +short)
    echo "  $ip → ${ptr:-no PTR record}"
done

echo -e "\n[DNSSEC]"
dig $DOMAIN +dnssec +short | grep -q "RRSIG" && echo "  ✓ DNSSEC enabled" || echo "  ✗ DNSSEC not detected"
```

### Exercise 2: DNS Propagation Checker
```python
#!/usr/bin/env python3
import subprocess

def query_dns(domain, record_type, server):
    """Query a specific DNS server for a record"""
    try:
        result = subprocess.run(
            ['dig', f'@{server}', domain, record_type, '+short', '+time=3'],
            capture_output=True, text=True, timeout=10
        )
        return result.stdout.strip() or "(no response)"
    except:
        return "timeout"

domain = "google.com"
record_type = "A"

resolvers = {
    "Google (8.8.8.8)": "8.8.8.8",
    "Cloudflare (1.1.1.1)": "1.1.1.1",
    "OpenDNS (208.67.222.222)": "208.67.222.222",
    "Quad9 (9.9.9.9)": "9.9.9.9",
    "Comodo (8.26.56.26)": "8.26.56.26",
}

print(f"DNS Propagation Check: {domain} ({record_type})")
print("=" * 55)
results = {}
for name, server in resolvers.items():
    answer = query_dns(domain, record_type, server)
    results[name] = answer
    status = "✓" if answer and answer != "timeout" else "✗"
    print(f"  {status} {name:<30}: {answer}")

# Check consistency
unique = set(results.values())
if len(unique) == 1:
    print("\n✓ All resolvers agree — propagation complete")
else:
    print(f"\n⚠ {len(unique)} different answers — propagation in progress")
```

### Exercise 3: DNS Troubleshooter
```bash
#!/bin/bash
# Automated DNS troubleshooting

DOMAIN=${1:-google.com}
echo "DNS Troubleshooter for: $DOMAIN"
echo "================================"

# Step 1: Local resolution
echo -e "\n1. Local resolver:"
LOCAL=$(dig $DOMAIN A +short | head -1)
[ -n "$LOCAL" ] && echo "  ✓ Resolves to $LOCAL" || echo "  ✗ FAILED"

# Step 2: Try public DNS
echo -e "\n2. Public DNS (8.8.8.8):"
PUBLIC=$(dig @8.8.8.8 $DOMAIN A +short | head -1)
[ -n "$PUBLIC" ] && echo "  ✓ Resolves to $PUBLIC" || echo "  ✗ FAILED"

# Step 3: Check /etc/resolv.conf
echo -e "\n3. Configured nameservers:"
grep nameserver /etc/resolv.conf | awk '{print "  "$0}'

# Step 4: Authorititative check
echo -e "\n4. Authoritative nameservers:"
dig $DOMAIN NS +short | sed 's/^/  /'

# Step 5: Check connectivity to resolver
RESOLVER=$(grep nameserver /etc/resolv.conf | head -1 | awk '{print $2}')
echo -e "\n5. Resolver reachability ($RESOLVER):"
nc -zw2 $RESOLVER 53 2>/dev/null && echo "  ✓ TCP/53 reachable" || echo "  ✗ TCP/53 unreachable"

# Step 6: Compare local vs authoritative
echo -e "\n6. Consistency check:"
AUTH_NS=$(dig $DOMAIN NS +short | head -1)
if [ -n "$AUTH_NS" ]; then
    AUTH_IP=$(dig @$AUTH_NS $DOMAIN A +short | head -1)
    if [ "$LOCAL" = "$AUTH_IP" ]; then
        echo "  ✓ Local and authoritative match ($LOCAL)"
    else
        echo "  ⚠ Mismatch: local=$LOCAL auth=$AUTH_IP (cache staleness?)"
    fi
fi
```

## Sample Exercises

1. **Record Design:** Design the DNS zone for a company with: web server, mail server, two nameservers, and SPF records.

2. **TTL Strategy:** A company is migrating to a new IP. Outline the TTL change strategy and timing.

3. **Troubleshooting:** Users report they can't reach `app.example.com`. The IP is correct in Route 53. What DNS-level issues would you investigate?

4. **Split-Horizon DNS:** Explain split-horizon DNS and when it's used. How does AWS Route 53 private hosted zones implement this?

5. **DNSSEC Chain of Trust:** Explain how DNSSEC validates a response from `www.example.com` all the way back to the root.

## Solutions

1. **Zone Design:**
   ```
   @ A 203.0.113.10          ; web server
   www CNAME @
   mail A 203.0.113.20
   @ MX 10 mail.example.com.
   @ NS ns1.example.com.
   @ NS ns2.example.com.
   ns1 A 203.0.113.1
   ns2 A 203.0.113.2
   @ TXT "v=spf1 a:mail.example.com ~all"
   ```

2. **TTL Migration Strategy:**
   - T-48h: Lower TTL to 300s (clients will cache for 5 min max)
   - T-0: Change A record to new IP
   - T+5min: Old IP traffic should be near zero
   - T+24h: Raise TTL back to 3600s
   - T+48h: Decommission old IP

3. **Troubleshooting Checklist:**
   - `dig app.example.com` — does it resolve?
   - `dig @ns1.example.com app.example.com` — is the auth server serving it?
   - Check TTL — is a cached negative response (NXDOMAIN) stuck?
   - Check CNAME chains — infinite loop or missing end record?
   - Check ACLs — is the DNS response being filtered?
   - Check /etc/hosts on client — overriding DNS?

4. **Split-Horizon DNS:**
   - Same domain resolves to different IPs based on source (internal vs external)
   - Example: `app.example.com` → `10.0.0.50` internally, `203.0.113.50` externally
   - AWS Route 53: Private hosted zone resolves within VPC; public zone resolves from internet
   - Use cases: Internal load balancers, private services, security

5. **DNSSEC Chain of Trust:**
   - Root zone signed with RRSIG; root DNSKEY in trust anchors (DNSSEC validators know this)
   - Root DS record points to .com zone's DNSKEY
   - .com DNSKEY signs example.com's DS record
   - example.com DNSKEY signs all records (A, MX, etc.)
   - Validator follows chain: root → .com → example.com → validates RRSIG on answer

## Completion Checklist
- [ ] Explain the full DNS resolution process
- [ ] Know all major DNS record types and their purpose
- [ ] Can troubleshoot NXDOMAIN, SERVFAIL, and propagation issues
- [ ] Use `dig` and `nslookup` confidently for DNS analysis
- [ ] Understand cloud DNS offerings (Route 53, Cloud DNS, Azure DNS)
- [ ] Know how TTL affects caching and change propagation

## Key Takeaways
- DNS is a distributed, hierarchical, cached database
- TTL controls caching; plan changes by lowering TTL first
- Always use `dig` over `nslookup` for scripting and detailed analysis
- Cloud DNS adds routing intelligence (latency, failover, geo) on top of basic DNS
- DNSSEC adds cryptographic validation to prevent poisoning attacks

## Next Steps
Proceed to [Day 14: DHCP, NAT, PAT, Private/Public IPs, Cloud IP Management](../Day_14/notes_and_exercises.md) to learn how IPs are assigned and translated in networks.