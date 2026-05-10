# Day 24: Service Discovery & DNS in Cloud, Consul, CloudMap

## Learning Objectives
By the end of Day 24, you will:
- Understand why service discovery is essential in dynamic cloud environments
- Configure AWS Cloud Map for service registry and DNS-based discovery
- Deploy and use HashiCorp Consul for service mesh and health-aware discovery
- Compare DNS-based vs API-based service discovery patterns

**Estimated Time:** 3-4 hours

---

## Notes

### What is Service Discovery?

In traditional environments, servers had fixed IPs — you could hardcode them. In modern cloud and container environments:
- Instances are ephemeral (auto-scaling, spot, containers)
- IPs change constantly
- Services scale horizontally (multiple instances)
- Services move between hosts (Kubernetes scheduling)

**Service discovery** solves this by maintaining a registry of **service → healthy endpoint** mappings, updated in real time.

### Service Discovery Models

#### DNS-based Discovery
- Services register with a DNS name
- Clients resolve the name to get IPs
- Simple, works with existing libraries
- TTL can cause stale data
- No awareness of application health (only host health)

```
Client → DNS: "where is payment-service?"
DNS → Client: "10.0.1.10, 10.0.1.11, 10.0.1.12"
Client → 10.0.1.10: request
```

#### API-based Discovery (Client-side)
- Client queries a service registry API
- Gets list of healthy instances with metadata
- Client does load balancing
- Example: Consul, Eureka, etcd

```
Client → Consul API: GET /v1/health/service/payment-service
Consul → Client: [{address: "10.0.1.10", port: 8080, status: "passing"}, ...]
Client → 10.0.1.10:8080: request
```

#### Server-side Discovery (Proxy-based)
- Client sends request to load balancer/proxy
- Proxy queries registry and forwards to healthy instance
- Client is unaware of discovery
- Example: AWS ALB + ECS, Istio, Envoy

```
Client → Load Balancer → [queries registry] → healthy instance
```

### AWS Cloud Map

AWS Cloud Map is a **cloud resource discovery service** that lets you define custom names for application resources and maintains the location of those resources.

#### Cloud Map Concepts
- **Namespace:** A grouping, corresponds to a DNS domain (e.g., `production.local`)
- **Service:** A logical entity within a namespace (e.g., `payment`)
- **Service Instance:** A registered endpoint (IP:port, hostname, ARN)
- **Health checks:** Route 53 health checks or custom health check attributes

#### Cloud Map Registration
```bash
# Create a namespace (DNS-based)
aws servicediscovery create-private-dns-namespace \
  --name production.local \
  --vpc vpc-12345678 \
  --region us-east-1

# Create a service within the namespace
aws servicediscovery create-service \
  --name payment \
  --namespace-id ns-xxxxxxxxxxxxxxxx \
  --dns-config "NamespaceId=ns-xxx,DnsRecords=[{Type=A,TTL=30}]" \
  --health-check-custom-config FailureThreshold=1

# Register an instance
aws servicediscovery register-instance \
  --service-id srv-xxxxxxxxxxxxxxxx \
  --instance-id payment-instance-1 \
  --attributes "AWS_INSTANCE_IPV4=10.0.1.10,AWS_INSTANCE_PORT=8080"

# Discover instances (API call)
aws servicediscovery discover-instances \
  --namespace-name production.local \
  --service-name payment \
  --query-parameters "version=v2"

# DNS-based discovery (automatic after registration)
# dig payment.production.local
```

#### Cloud Map with ECS
When using ECS with Service Connect or Cloud Map integration:
```json
{
  "serviceRegistries": [{
    "registryArn": "arn:aws:servicediscovery:us-east-1:123:service/srv-xxx",
    "port": 8080
  }]
}
```
ECS automatically registers/deregisters tasks as they start/stop.

### HashiCorp Consul

Consul is a **full-featured service mesh** with service discovery, health checking, key-value store, and multi-datacenter support.

#### Consul Architecture
```
┌─────────────────────────────────────────┐
│              Consul Cluster              │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Server 1│  │ Server 2│  │ Server 3│ │  ← Raft consensus
│  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────┘
         ↑              ↑
    Consul Agent   Consul Agent       ← On every node
         |              |
    Service A      Service B          ← Registered services
```

#### Consul Agent Modes
- **Server:** Participates in Raft consensus, stores data, 3 or 5 per datacenter
- **Client:** Lightweight, forwards requests to servers, runs on every node

#### Installing and Running Consul
```bash
# Install Consul
wget https://releases.hashicorp.com/consul/1.17.0/consul_1.17.0_linux_amd64.zip
unzip consul_1.17.0_linux_amd64.zip
sudo mv consul /usr/local/bin/

# Start development server (single node, non-production)
consul agent -dev -ui -client=0.0.0.0 &

# Verify
consul members
consul info
```

#### Registering Services with Consul
```bash
# Method 1: Service definition file
cat > /etc/consul.d/payment.json <<'EOF'
{
  "service": {
    "name": "payment",
    "tags": ["v2", "production"],
    "port": 8080,
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s",
      "timeout": "3s"
    }
  }
}
EOF

consul reload

# Method 2: HTTP API
curl --request PUT \
  --data '{
    "Name": "payment",
    "Port": 8080,
    "Check": {
      "HTTP": "http://localhost:8080/health",
      "Interval": "10s"
    }
  }' \
  http://localhost:8500/v1/agent/service/register
```

#### Querying Consul
```bash
# List all services
curl http://localhost:8500/v1/catalog/services | jq

# Get healthy instances of a service
curl http://localhost:8500/v1/health/service/payment?passing=true | jq

# DNS-based discovery (Consul acts as DNS server on port 8600)
dig @127.0.0.1 -p 8600 payment.service.consul

# DNS with tag filter
dig @127.0.0.1 -p 8600 v2.payment.service.consul

# KV store (for config)
consul kv put config/payment/db-url "postgres://10.0.2.10:5432/payments"
consul kv get config/payment/db-url
```

#### Consul Service Mesh (Connect)
Consul Connect provides **mTLS-based service-to-service encryption** using sidecar proxies (Envoy):
```
Service A → [Envoy Sidecar] ──mTLS──→ [Envoy Sidecar] → Service B

Benefits:
- Zero-trust networking
- Automatic certificate rotation
- Traffic intentions (allow/deny rules)
- Observability
```

```bash
# Define intention: payment can call database, others cannot
consul intention create payment database
consul intention create -deny "*" database
```

### Kubernetes DNS (CoreDNS)
In Kubernetes, CoreDNS provides built-in service discovery:

```
Service: payment-svc in namespace: production

DNS name: payment-svc.production.svc.cluster.local
Short form (within namespace): payment-svc
Cross-namespace: payment-svc.production

Headless service DNS → returns pod IPs directly (not VIP)
```

### Comparing Discovery Solutions

| Feature | Cloud Map | Consul | CoreDNS (K8s) | Route 53 |
|---------|-----------|--------|---------------|----------|
| Cloud-native | AWS only | Multi-cloud | K8s only | AWS only |
| Health checks | Yes | Yes (deep) | Yes (readiness) | Yes |
| Service mesh | No | Yes | Via Istio | No |
| KV store | No | Yes | No | No |
| Multi-datacenter | No | Yes | No | Yes |
| Cost | Pay per query | Open source | Free (K8s) | Pay per query |

---

## Hands-On Labs

### Lab 1: Consul Local Setup
```bash
#!/bin/bash
# Set up a local Consul dev environment

if ! command -v consul &>/dev/null; then
    echo "Installing Consul..."
    CONSUL_VERSION="1.17.0"
    wget -q https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip
    unzip -q consul_${CONSUL_VERSION}_linux_amd64.zip
    sudo mv consul /usr/local/bin/
    rm consul_${CONSUL_VERSION}_linux_amd64.zip
fi

# Create config directory
mkdir -p /tmp/consul-data /tmp/consul.d

# Start dev agent
consul agent -dev -client=0.0.0.0 -data-dir=/tmp/consul-data \
  -config-dir=/tmp/consul.d > /tmp/consul.log 2>&1 &
CONSUL_PID=$!
sleep 2

echo "Consul started (PID: $CONSUL_PID)"
consul members

# Register a test service
cat > /tmp/consul.d/web.json <<'EOF'
{
  "service": {
    "name": "web",
    "tags": ["nginx", "v1"],
    "port": 80,
    "check": {
      "http": "http://localhost:80/",
      "interval": "10s",
      "timeout": "2s"
    }
  }
}
EOF

consul reload 2>/dev/null
sleep 1

echo -e "\nRegistered services:"
curl -s http://localhost:8500/v1/catalog/services | python3 -m json.tool

echo -e "\nDNS lookup for web service:"
dig @127.0.0.1 -p 8600 web.service.consul +short 2>/dev/null || echo "DNS lookup (install dig to test)"

echo -e "\nStoping Consul..."
kill $CONSUL_PID 2>/dev/null
```

### Lab 2: Service Discovery API Simulator
```python
#!/usr/bin/env python3
import random, time, json
from http.server import HTTPServer, BaseHTTPRequestHandler
import threading

# Simple in-memory service registry
registry = {
    "payment": [
        {"id": "pay-1", "ip": "10.0.1.10", "port": 8080, "healthy": True, "version": "v2"},
        {"id": "pay-2", "ip": "10.0.1.11", "port": 8080, "healthy": True, "version": "v2"},
        {"id": "pay-3", "ip": "10.0.1.12", "port": 8080, "healthy": False, "version": "v1"},
    ],
    "inventory": [
        {"id": "inv-1", "ip": "10.0.2.10", "port": 9000, "healthy": True, "version": "v1"},
        {"id": "inv-2", "ip": "10.0.2.11", "port": 9000, "healthy": True, "version": "v1"},
    ]
}

class DiscoveryHandler(BaseHTTPRequestHandler):
    def log_message(self, format, *args): pass  # silence logs

    def do_GET(self):
        parts = self.path.strip('/').split('/')
        # GET /health/service/<name>
        if len(parts) == 3 and parts[0] == 'health' and parts[1] == 'service':
            svc = parts[2].split('?')[0]
            passing_only = 'passing=true' in self.path
            instances = registry.get(svc, [])
            if passing_only:
                instances = [i for i in instances if i['healthy']]
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps(instances).encode())
        # GET /catalog/services
        elif self.path == '/catalog/services' or self.path == '/v1/catalog/services':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            self.wfile.write(json.dumps({k: [] for k in registry}).encode())
        else:
            self.send_response(404)
            self.end_headers()

# Test the registry
print("Service Registry Demo (in-memory)")
print("=" * 45)

for svc, instances in registry.items():
    healthy = [i for i in instances if i['healthy']]
    print(f"\nService: {svc}")
    print(f"  Total instances: {len(instances)}, Healthy: {len(healthy)}")
    for inst in healthy:
        print(f"  → {inst['ip']}:{inst['port']} [{inst['version']}]")

# Simulate a client using service discovery
print("\n--- Client Discovery Simulation ---")
def discover(service_name, passing_only=True):
    instances = registry.get(service_name, [])
    if passing_only:
        instances = [i for i in instances if i['healthy']]
    if instances:
        chosen = random.choice(instances)
        return f"{chosen['ip']}:{chosen['port']}"
    return None

for i in range(6):
    endpoint = discover("payment")
    print(f"Request {i+1} → {endpoint}")
```

### Lab 3: DNS-based Service Discovery
```bash
#!/bin/bash
# Simulate DNS-based service discovery using /etc/hosts

echo "DNS-based Service Discovery Simulation"
echo "======================================="

# Register services in /etc/hosts (simulating DNS)
SERVICES=(
    "10.0.1.10 payment-v1.service.local"
    "10.0.1.11 payment-v1.service.local"
    "10.0.1.20 payment-v2.service.local"
    "10.0.2.10 inventory.service.local"
    "10.0.3.10 notification.service.local"
)

echo "Service Registry (/etc/hosts entries):"
for entry in "${SERVICES[@]}"; do
    echo "  $entry"
done

echo -e "\nSimulating DNS resolution:"
for svc in "payment-v1.service.local" "payment-v2.service.local" "inventory.service.local"; do
    # In real DNS, multiple A records returned for same name
    echo "  Query: $svc"
    for entry in "${SERVICES[@]}"; do
        if echo "$entry" | grep -q "$svc"; then
            ip=$(echo $entry | awk '{print $1}')
            echo "    → $ip"
        fi
    done
done

echo -e "\nDNS TTL consideration:"
echo "  TTL = 30s → stale entries cached up to 30 seconds after unhealthy"
echo "  TTL = 5s  → more responsive but higher DNS query load"
echo "  Recommendation: TTL 5-30s for dynamic services"
```

---

## Practical Exercises

### Exercise 1: Service Registry with Health Tracking
```python
#!/usr/bin/env python3
import time, random, threading

class ServiceRegistry:
    def __init__(self):
        self.services = {}
        self.lock = threading.Lock()

    def register(self, name, instance_id, host, port, tags=None):
        with self.lock:
            if name not in self.services:
                self.services[name] = {}
            self.services[name][instance_id] = {
                "host": host, "port": port,
                "tags": tags or [],
                "healthy": True,
                "registered_at": time.time(),
                "last_check": time.time()
            }
        print(f"[REGISTER] {name}/{instance_id} @ {host}:{port}")

    def deregister(self, name, instance_id):
        with self.lock:
            if name in self.services and instance_id in self.services[name]:
                del self.services[name][instance_id]
        print(f"[DEREGISTER] {name}/{instance_id}")

    def update_health(self, name, instance_id, healthy):
        with self.lock:
            if name in self.services and instance_id in self.services[name]:
                old = self.services[name][instance_id]["healthy"]
                self.services[name][instance_id]["healthy"] = healthy
                self.services[name][instance_id]["last_check"] = time.time()
                if old != healthy:
                    status = "HEALTHY" if healthy else "UNHEALTHY"
                    print(f"[HEALTH] {name}/{instance_id} → {status}")

    def discover(self, name, tag=None):
        with self.lock:
            instances = self.services.get(name, {}).values()
            result = [i for i in instances if i["healthy"]]
            if tag:
                result = [i for i in result if tag in i["tags"]]
            return result

    def status(self):
        print(f"\n{'Service':<20}{'Instance':<20}{'Endpoint':<22}{'Healthy'}")
        print("-" * 70)
        for name, instances in self.services.items():
            for iid, info in instances.items():
                ep = f"{info['host']}:{info['port']}"
                h = "✓" if info["healthy"] else "✗"
                tags = ",".join(info["tags"]) if info["tags"] else ""
                print(f"  {name:<18}{iid:<20}{ep:<22}{h}  {tags}")

# Demo
reg = ServiceRegistry()

# Register services
for i in range(1, 4):
    reg.register("payment", f"pay-{i}", f"10.0.1.{9+i}", 8080, tags=["v2"])
for i in range(1, 3):
    reg.register("inventory", f"inv-{i}", f"10.0.2.{9+i}", 9000, tags=["v1"])

reg.status()

# Simulate health changes
reg.update_health("payment", "pay-2", False)
reg.update_health("inventory", "inv-1", False)

print("\nHealthy payment instances:")
for inst in reg.discover("payment"):
    print(f"  → {inst['host']}:{inst['port']}")

# Re-register after recovery
reg.update_health("payment", "pay-2", True)
print("\nAfter recovery:")
for inst in reg.discover("payment"):
    print(f"  → {inst['host']}:{inst['port']}")
```

### Exercise 2: Consul KV Config Manager
```bash
#!/bin/bash
# Consul KV store for service configuration

CONSUL_ADDR=${CONSUL_HTTP_ADDR:-"http://localhost:8500"}

# Check if Consul is running
if ! curl -sf $CONSUL_ADDR/v1/status/leader > /dev/null 2>&1; then
    echo "Consul not running. Start with: consul agent -dev"
    exit 0
fi

echo "Consul KV Configuration Manager"
echo "================================"

# Store config values
echo -e "\n[Writing config]"
configs=(
    "config/payment/db-url=postgres://10.0.2.10:5432/payments"
    "config/payment/redis-url=redis://10.0.3.10:6379"
    "config/payment/timeout=30"
    "config/inventory/db-url=postgres://10.0.2.11:5432/inventory"
    "config/global/log-level=INFO"
)

for kv in "${configs[@]}"; do
    key=$(echo $kv | cut -d= -f1)
    val=$(echo $kv | cut -d= -f2-)
    curl -sf -X PUT -d "$val" "$CONSUL_ADDR/v1/kv/$key" > /dev/null
    echo "  Set: $key = $val"
done

# Read config
echo -e "\n[Reading config for payment service]"
for key in config/payment/db-url config/payment/redis-url config/payment/timeout; do
    value=$(curl -sf "$CONSUL_ADDR/v1/kv/$key?raw" 2>/dev/null)
    echo "  $key = $value"
done

# List all keys
echo -e "\n[All config keys]"
curl -sf "$CONSUL_ADDR/v1/kv/?keys" 2>/dev/null | python3 -m json.tool
```

---

## Sample Exercises

1. **Discovery choice:** You have 200 microservices in ECS on AWS. Should you use AWS Cloud Map or self-managed Consul? Justify.
2. **DNS TTL impact:** A service has a 60-second DNS TTL and 3 instances. One instance becomes unhealthy. What is the worst-case time a client continues sending traffic to the unhealthy instance?
3. **Consul health check types:** List the 4 types of health checks Consul supports and give a use case for each.
4. **Service mesh vs discovery:** What additional capabilities does Consul Connect (service mesh) provide over basic service discovery?
5. **Cloud Map vs Route 53:** When would you use Cloud Map instead of a simple Route 53 private hosted zone for service discovery?

## Solutions

1. **Cloud Map vs Consul for 200 ECS services:** **AWS Cloud Map** — it integrates natively with ECS (auto-registration on task start/stop), works with Route 53 for DNS-based discovery, no extra infrastructure to manage, and AWS supports the health check integration. Consul makes more sense when you're multi-cloud, need service mesh (mTLS), have multi-datacenter requirements, or need a KV store for config management.

2. **DNS TTL worst case:** With TTL=60s, a client could cache the unhealthy instance's IP for up to **60 seconds** after it becomes unhealthy. Combined with the health check detection interval (e.g., 30s), total worst case = 30s (detection) + 60s (TTL drain) = **90 seconds** of traffic to an unhealthy instance.

3. **Consul health check types:**
   - **HTTP:** `GET /health` — expects 2xx; for web services
   - **TCP:** connects to port; for any TCP service, databases
   - **Script/Command:** runs a shell command; for custom checks
   - **TTL:** service must call Consul to say it's alive; for custom push-based health
   - **gRPC:** gRPC health protocol; for gRPC services

4. **Service mesh vs discovery:** Consul Connect adds: (a) **mTLS** — encrypted, authenticated service-to-service traffic; (b) **Intentions** — access control (payment can call database, frontend cannot); (c) **Observability** — metrics per service pair (latency, error rate); (d) **Traffic management** — L7 routing, canary deployments; (e) **Certificate management** — automatic rotation of service certificates.

5. **Cloud Map vs Route 53 private zone:** Use Cloud Map when: services need metadata attributes (version, environment) beyond just IP; you need API-based discovery (not just DNS); you need health-check-aware routing; ECS/EKS automatic deregistration; you need to filter instances by attributes. Use Route 53 private zone when: you just need a stable internal DNS name for a fixed endpoint (e.g., an internal NLB); no instance-level health awareness needed; simplest possible setup.

## Completion Checklist
- [ ] Explain DNS-based vs API-based service discovery
- [ ] Register and discover services using AWS Cloud Map
- [ ] Install Consul, register services, and query via DNS and HTTP API
- [ ] Implement health-check-aware service discovery
- [ ] Use Consul KV store for distributed configuration
- [ ] Compare Cloud Map, Consul, and CoreDNS for Kubernetes

## Key Takeaways
- Service discovery is essential when instance IPs change dynamically
- DNS-based discovery is simple but TTL causes staleness; API-based is more responsive
- AWS Cloud Map integrates natively with ECS and Route 53
- Consul adds KV store, service mesh, multi-datacenter support, and deep health checking
- Kubernetes CoreDNS provides built-in service discovery for pod-to-pod communication
- Always pair service discovery with health checks to avoid routing to unhealthy instances

## Next Steps
Proceed to [Day 25: Network Automation (Ansible, Terraform, IaC, NetDevOps)](../Day_25/notes_and_exercises.md).