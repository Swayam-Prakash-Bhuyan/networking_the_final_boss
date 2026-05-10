# Day 17: Load Balancing (L4/L7, HAProxy, Nginx, Cloud Load Balancers)

## Learning Objectives
By the end of Day 17, you will:
- Understand L4 vs L7 load balancing and when to use each
- Configure HAProxy for TCP and HTTP load balancing
- Set up Nginx as a reverse proxy and load balancer
- Use AWS ALB, NLB, CLB and equivalent cloud services

**Estimated Time:** 3-4 hours

---

## Notes

### What is Load Balancing?
Load balancing distributes incoming traffic across multiple backend servers to:
- **Increase availability** — if one server fails, traffic goes to others
- **Scale horizontally** — add servers to handle more load
- **Reduce latency** — route to the least-loaded or nearest server
- **Enable maintenance** — drain a server without downtime

### L4 vs L7 Load Balancing

| Feature | L4 (Transport) | L7 (Application) |
|---------|---------------|-----------------|
| Works at | TCP/UDP | HTTP/HTTPS |
| Sees | IP, Port | URL, Headers, Cookies |
| Speed | Faster | Slower (parses payload) |
| Routing | IP:Port based | URL/path/host/cookie |
| SSL | Passthrough | Termination |
| Examples | AWS NLB, HAProxy TCP | AWS ALB, Nginx, HAProxy HTTP |

### Load Balancing Algorithms

| Algorithm | Description | Best for |
|-----------|-------------|---------|
| Round Robin | Requests distributed in order | Equal capacity servers |
| Least Connections | Route to server with fewest active connections | Long-lived sessions |
| Weighted RR | Assign more traffic to powerful servers | Mixed capacity |
| IP Hash | Same client always hits same server | Sticky sessions |
| Random | Random selection | Simple stateless apps |
| Resource Based | Route based on CPU/RAM | Adaptive scaling |

### Health Checks
Load balancers continuously check backend health:
- **Active:** LB sends probe (HTTP GET /health, TCP connect)
- **Passive:** LB observes real traffic errors
- **Thresholds:** Unhealthy after N failures; healthy after M successes

### HAProxy

HAProxy is a high-performance, open-source L4/L7 load balancer.

#### HAProxy Config Structure
```
global       → process-wide settings
defaults     → default values for all frontends/backends
frontend     → defines listening address and rules
backend      → defines server pool and algorithm
```

#### Example: HTTP Load Balancer
```
global
    log /dev/log local0
    maxconn 50000
    user haproxy
    group haproxy
    daemon

defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend http_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/example.pem
    redirect scheme https if !{ ssl_fc }
    default_backend web_servers
    acl is_api path_beg /api/
    use_backend api_servers if is_api

backend web_servers
    balance roundrobin
    option httpchk GET /health HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200
    server web1 10.0.1.10:8080 check weight 1
    server web2 10.0.1.11:8080 check weight 1
    server web3 10.0.1.12:8080 check weight 2

backend api_servers
    balance leastconn
    server api1 10.0.2.10:9000 check
    server api2 10.0.2.11:9000 check
    server api3 10.0.2.12:9000 check backup
```

#### HAProxy Stats Page
```
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:secret
```

### Nginx Load Balancing

```nginx
# /etc/nginx/nginx.conf or /etc/nginx/conf.d/lb.conf

upstream web_backend {
    least_conn;                         # algorithm
    server 10.0.1.10:8080 weight=3;
    server 10.0.1.11:8080 weight=1;
    server 10.0.1.12:8080 backup;      # only used when others fail
    keepalive 32;                       # keep persistent connections
}

upstream api_backend {
    ip_hash;                            # sticky sessions by client IP
    server 10.0.2.10:9000;
    server 10.0.2.11:9000;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.crt;
    ssl_certificate_key /etc/ssl/private/example.key;

    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Health check endpoint (served by Nginx itself)
    location /nginx-health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### AWS Load Balancers

| Type | Layer | Protocol | Use Case |
|------|-------|----------|---------|
| ALB (Application) | L7 | HTTP, HTTPS, gRPC | Web apps, microservices, path/host routing |
| NLB (Network) | L4 | TCP, UDP, TLS | High performance, gaming, IoT, static IP |
| CLB (Classic) | L4/L7 | TCP, HTTP | Legacy; use ALB/NLB instead |
| GWLB (Gateway) | L3+L4 | IP | Security appliance insertion |

#### ALB Key Features
- **Path-based routing:** `/api/*` → one target group, `/static/*` → another
- **Host-based routing:** `app.example.com` → app servers, `api.example.com` → API servers
- **Weighted target groups:** canary deployments, blue/green
- **Lambda target:** Serverless backends
- **Sticky sessions:** Cookie-based (AWSALB cookie)
- **WAF integration:** Layer 7 filtering
- **Access logs:** Every request logged to S3

#### ALB Target Groups
```
Target Group: web-tg
  Protocol: HTTP, Port: 8080
  Health check: GET /health, expect 200
  Targets:
    i-abc123 (10.0.1.10) → healthy
    i-def456 (10.0.1.11) → healthy
    i-ghi789 (10.0.1.12) → unhealthy (draining)
```

#### NLB Key Features
- Static IP per AZ (useful for firewall whitelisting)
- Handles millions of requests/sec with ultra-low latency
- Preserves client source IP
- TLS offloading supported
- No security groups on NLB itself

---

## Hands-On Labs

### Lab 1: Install and Test HAProxy
```bash
sudo apt update && sudo apt install haproxy -y

# Write minimal config
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
global
    log /dev/log local0
    daemon

defaults
    mode http
    log global
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend http_front
    bind *:8080
    default_backend servers

backend servers
    balance roundrobin
    option httpchk GET /
    server s1 127.0.0.1:8001 check
    server s2 127.0.0.1:8002 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats auth admin:admin
EOF

# Start two simple backends
python3 -m http.server 8001 --directory /tmp &
python3 -m http.server 8002 --directory /tmp &

sudo systemctl restart haproxy
echo "Test: curl http://localhost:8080"
echo "Stats: http://localhost:8404/stats"
```

### Lab 2: Nginx Upstream Config
```bash
sudo apt install nginx -y

sudo tee /etc/nginx/conf.d/loadbalancer.conf > /dev/null <<'EOF'
upstream backend {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}

server {
    listen 9090;
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Real-IP $remote_addr;
    }
    location /health {
        return 200 "ok\n";
    }
}
EOF

sudo nginx -t && sudo systemctl reload nginx
echo "Test: curl http://localhost:9090"
```

### Lab 3: Load Balancer Health Check Script
```bash
#!/bin/bash
BACKENDS=("127.0.0.1:8001" "127.0.0.1:8002" "127.0.0.1:8003")

echo "Backend Health Check"
echo "===================="
for backend in "${BACKENDS[@]}"; do
    host=$(echo $backend | cut -d: -f1)
    port=$(echo $backend | cut -d: -f2)
    if curl -sf --max-time 2 "http://$backend/" > /dev/null 2>&1; then
        echo "  ✓ $backend — HEALTHY"
    else
        echo "  ✗ $backend — UNHEALTHY"
    fi
done
```

---

## Practical Exercises

### Exercise 1: Round-Robin Simulator
```python
#!/usr/bin/env python3
import itertools

class LoadBalancer:
    def __init__(self, servers, algorithm="round_robin"):
        self.servers = servers
        self.algorithm = algorithm
        self._rr = itertools.cycle(servers)
        self.connections = {s: 0 for s in servers}
        self.healthy = {s: True for s in servers}

    def get_server(self):
        healthy = [s for s in self.servers if self.healthy[s]]
        if not healthy:
            return None
        if self.algorithm == "round_robin":
            for _ in range(len(self.servers)):
                s = next(self._rr)
                if self.healthy[s]:
                    return s
        elif self.algorithm == "least_conn":
            return min(healthy, key=lambda s: self.connections[s])

    def send_request(self, request_id):
        server = self.get_server()
        if server:
            self.connections[server] += 1
            print(f"Request {request_id:3} → {server}  (conns: {self.connections[server]})")
        else:
            print(f"Request {request_id:3} → NO HEALTHY SERVER")

lb = LoadBalancer(["10.0.1.10:8080", "10.0.1.11:8080", "10.0.1.12:8080"])
for i in range(1, 10):
    lb.send_request(i)

print("\n--- Simulating server failure ---")
lb.healthy["10.0.1.11:8080"] = False
for i in range(10, 16):
    lb.send_request(i)
```

### Exercise 2: HAProxy Config Generator
```python
#!/usr/bin/env python3

def generate_haproxy_config(frontends, backends):
    lines = [
        "global",
        "    daemon",
        "    log /dev/log local0",
        "",
        "defaults",
        "    mode http",
        "    timeout connect 5s",
        "    timeout client 30s",
        "    timeout server 30s",
        "",
    ]
    for fe in frontends:
        lines += [
            f"frontend {fe['name']}",
            f"    bind *:{fe['port']}",
            f"    default_backend {fe['default_backend']}",
            "",
        ]
    for be in backends:
        lines += [
            f"backend {be['name']}",
            f"    balance {be.get('algorithm','roundrobin')}",
            f"    option httpchk GET {be.get('health_path','/health')}",
        ]
        for i, srv in enumerate(be['servers']):
            lines.append(f"    server s{i+1} {srv} check")
        lines.append("")
    return "\n".join(lines)

config = generate_haproxy_config(
    frontends=[{"name": "http_front", "port": 80, "default_backend": "web_pool"}],
    backends=[{
        "name": "web_pool",
        "algorithm": "leastconn",
        "health_path": "/health",
        "servers": ["10.0.1.10:8080", "10.0.1.11:8080", "10.0.1.12:8080"]
    }]
)
print(config)
```

---

## Sample Exercises

1. A web app has uneven server capacities (Server A: 8 CPU, Server B: 4 CPU, Server C: 4 CPU). Which algorithm and weights would you configure?
2. You need sticky sessions without application changes. How do you configure this in HAProxy? In Nginx?
3. Design an ALB routing scheme for a monorepo with `/` → frontend, `/api/` → backend, `/admin/` → internal-only service.
4. An NLB is returning 502 errors for some requests. What are the likely causes?
5. Explain the difference between ALB and NLB in terms of target health checks.

## Solutions

1. **Weighted Round Robin:** `server a 10.0.1.10:80 weight 2`, `server b 10.0.1.11:80 weight 1`, `server c 10.0.1.12:80 weight 1` — server A gets 2x traffic.
2. **Sticky sessions:** HAProxy: `cookie SERVERID insert indirect nocache` per server; Nginx: `sticky cookie srv_id expires=1h;` in upstream block.
3. **ALB routing:** Listener rules: rule 1 path `/api/*` → api-tg, rule 2 path `/admin/*` → admin-tg (with IP condition for internal), default → frontend-tg.
4. **NLB 502 causes:** Backend health check failing (port not open), security group blocking NLB probes, backend returning unexpected responses, connection timeout too short.
5. **ALB vs NLB health checks:** ALB uses HTTP/HTTPS GET to a specific path, expects a status code; NLB supports TCP (just connect), HTTP, or HTTPS — TCP is simplest and fastest.

## Completion Checklist
- [ ] Explain L4 vs L7 load balancing differences
- [ ] Write a functional HAProxy config for HTTP
- [ ] Configure Nginx upstream block with health checks
- [ ] Know AWS ALB vs NLB use cases
- [ ] Implement path-based and host-based routing
- [ ] Understand sticky sessions and their trade-offs

## Key Takeaways
- L7 load balancers provide routing intelligence at the cost of slightly higher overhead
- HAProxy is the gold standard for performance-critical TCP and HTTP balancing
- Nginx is excellent as a combined web server, reverse proxy, and load balancer
- AWS ALB handles most web workloads; NLB excels at raw TCP performance and static IPs
- Health checks are the backbone of high availability — configure them carefully

## Next Steps
Proceed to [Day 18: HTTP, HTTPS, TLS, Certificates, Reverse Proxy, API Gateways](../Day_18/notes_and_exercises.md).