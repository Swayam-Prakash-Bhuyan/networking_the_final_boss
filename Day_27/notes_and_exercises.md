# Day 27: CDN, Anycast, Global Load Balancing, CloudFront, Cloud CDN

## Learning Objectives
By the end of Day 27, you will:
- Understand how CDNs reduce latency through edge caching
- Explain Anycast routing and its role in CDNs and DDoS mitigation
- Configure AWS CloudFront distributions with cache behaviors
- Compare CDN offerings across AWS, GCP, and Azure

**Estimated Time:** 3-4 hours

---

## Notes

### What is a CDN?

A **Content Delivery Network (CDN)** is a geographically distributed network of servers (PoPs — Points of Presence) that cache content close to end users, reducing latency and origin load.

**Without CDN:**
```
User (Mumbai) → origin server (us-east-1): ~200ms RTT
```
**With CDN:**
```
User (Mumbai) → PoP (Mumbai): ~5ms RTT
PoP cache hit → serve immediately
PoP cache miss → fetch from origin once, cache for all future requests
```

### CDN Concepts

#### Cache Hit vs Cache Miss
- **Cache HIT:** Content found at PoP → served immediately (fast, cheap)
- **Cache MISS:** Content not at PoP → fetched from origin, cached, returned

#### Cache-Control Headers
```http
Cache-Control: max-age=86400         → cache for 24 hours
Cache-Control: no-cache              → revalidate with origin every time
Cache-Control: no-store              → never cache (sensitive data)
Cache-Control: s-maxage=3600         → CDN-specific TTL (overrides max-age for CDN)
Cache-Control: public                → CDN can cache
Cache-Control: private               → only browser cache, not CDN
ETag: "abc123"                       → validator for conditional requests
Last-Modified: Mon, 10 May 2026 ...  → validator for conditional requests
```

#### Cache Invalidation
```bash
# AWS CloudFront invalidation
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"                      # all files
  --paths "/images/*" "/app.js"     # specific paths
```

#### Origin Types
- **S3 bucket** — static websites, media files
- **ALB / EC2** — dynamic content
- **Lambda@Edge / CloudFront Functions** — compute at edge
- **Custom origin** — any HTTPS endpoint (on-prem, other clouds)

### Anycast Routing

**Unicast:** One IP → one specific server
**Anycast:** Same IP advertised from multiple locations → routed to nearest

```
User in Tokyo   → 1.1.1.1 → Tokyo PoP
User in London  → 1.1.1.1 → London PoP
User in New York → 1.1.1.1 → New York PoP

All use the SAME IP (1.1.1.1), but BGP routes to the nearest node
```

**Used by:**
- CDNs (CloudFront, Cloudflare, Fastly)
- DNS resolvers (1.1.1.1, 8.8.8.8)
- DDoS mitigation (absorb attacks across all PoPs)
- AWS Global Accelerator

### Global Load Balancing (GLB)

Global load balancing routes users to the best origin/region based on:
- **Latency** — closest/fastest region
- **Geolocation** — serve from region closest to user's country
- **Health** — failover to next-best if primary is down
- **Weighted** — blue/green deployments, canary releases
- **Cost** — prefer cheaper regions when latency allows

#### DNS-based GLB (Route 53)
```
Route 53 Latency Routing:
  User in Tokyo   → api.example.com → ap-northeast-1 ALB (5ms)
  User in London  → api.example.com → eu-west-1 ALB (8ms)
  User in NY      → api.example.com → us-east-1 ALB (3ms)
```

#### Anycast-based GLB (AWS Global Accelerator)
```
Global Accelerator:
  Two static anycast IPs (e.g., 13.248.x.x and 76.223.x.x)
  User connects to nearest AWS edge PoP
  Traffic travels AWS backbone to origin (faster, more reliable than internet)
  Instant failover via anycast re-routing
```

### AWS CloudFront

CloudFront is AWS's global CDN with 450+ PoPs worldwide.

#### Distribution Types
- **Web distribution:** HTTP/HTTPS, static + dynamic content
- **RTMP:** Legacy streaming (deprecated)

#### CloudFront Configuration
```json
{
  "Origins": [{
    "DomainName": "mybucket.s3.amazonaws.com",
    "Id": "S3Origin",
    "S3OriginConfig": {"OriginAccessIdentity": "origin-access-identity/cloudfront/ABCDEF"}
  }],
  "DefaultCacheBehavior": {
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true,
    "AllowedMethods": ["GET", "HEAD"]
  },
  "CacheBehaviors": [{
    "PathPattern": "/api/*",
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",  // CachingDisabled
    "OriginRequestPolicyId": "b689b0a8-53d0-40ab-baf2-68738e2966ac",
    "ViewerProtocolPolicy": "https-only"
  }],
  "PriceClass": "PriceClass_All",
  "HttpVersion": "http2",
  "IsIPV6Enabled": true
}
```

#### Cache Behaviors (Path-based Routing)
```
/                → S3 bucket origin (cached, 24h TTL)
/api/*           → ALB origin (not cached, pass through)
/images/*        → S3 bucket origin (cached, 30 days TTL)
/admin/*         → ALB origin (not cached, require signed URL)
/static/*        → S3 bucket origin (cached, immutable, 1 year TTL)
```

#### Lambda@Edge and CloudFront Functions

**CloudFront Functions (lightweight, <1ms):**
- URL rewrites
- Header manipulation
- Simple A/B testing
- Request/response manipulation

```javascript
// CloudFront Function: Add security headers
function handler(event) {
  var response = event.response;
  var headers = response.headers;

  headers['strict-transport-security'] = {value: 'max-age=63072000'};
  headers['x-content-type-options'] = {value: 'nosniff'};
  headers['x-frame-options'] = {value: 'DENY'};
  headers['x-xss-protection'] = {value: '1; mode=block'};

  return response;
}
```

**Lambda@Edge (full Node.js/Python, ~1-5ms):**
- Authentication at the edge
- Dynamic content generation
- A/B testing with cookies
- Image resizing

#### CloudFront Security
- **OAC (Origin Access Control):** Only CloudFront can access S3 (bucket policy)
- **WAF integration:** Attach AWS WAF to inspect requests
- **Signed URLs/Cookies:** Time-limited access to private content
- **Geo-restriction:** Block or allow specific countries
- **Field-level encryption:** Encrypt specific POST fields at the edge

### Comparing CDN Services

| Feature | AWS CloudFront | GCP Cloud CDN | Azure CDN | Cloudflare |
|---------|---------------|---------------|-----------|------------|
| PoPs | 450+ | 160+ | 200+ | 300+ |
| Edge compute | Lambda@Edge, CF Functions | Cloud Run (edge) | Azure Functions | Workers |
| WAF | AWS WAF | Cloud Armor | Azure WAF | CF WAF |
| DDoS | AWS Shield | Cloud Armor | Azure DDoS | Unmetered |
| Real-time logs | Yes | Yes | Yes | Yes |
| Price model | Per GB + requests | Per GB | Per GB | Flat/per GB |
| Free tier | 1TB/month first year | No | Yes (some) | Yes (limited) |

### GCP Cloud CDN
```bash
# Enable Cloud CDN on a backend service
gcloud compute backend-services update my-backend \
  --enable-cdn \
  --global

# Set cache TTL
gcloud compute backend-services update my-backend \
  --cache-mode=CACHE_ALL_STATIC \
  --default-ttl=3600 \
  --max-ttl=86400 \
  --global

# Invalidate cache
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --path "/*" \
  --global
```

### Azure CDN
- **Azure CDN from Microsoft** (Verizon + Akamai integration)
- Integrated with Azure Front Door for global HTTPS + CDN
- Rules engine for URL rewriting and header modification

---

## Hands-On Labs

### Lab 1: Test CDN Performance
```bash
#!/bin/bash
# Compare performance with and without CDN

echo "CDN Performance Analysis"
echo "========================"

# Test multiple CDN endpoints
URLS=(
    "https://www.cloudflare.com/"
    "https://d111111abcdef8.cloudfront.net/"  # CloudFront example
    "https://httpbin.org/get"
)

for url in "${URLS[@]}"; do
    echo -e "\nTesting: $url"
    
    # First request (likely cache miss)
    result1=$(curl -o /dev/null -s -w \
        "  DNS: %{time_namelookup}s  Connect: %{time_connect}s  TTFB: %{time_starttransfer}s  Total: %{time_total}s  Size: %{size_download}B  Cache: %{http_code}" \
        -H "Cache-Control: no-cache" \
        --max-time 10 "$url" 2>/dev/null)
    echo "  Request 1: $result1"
    
    # Second request (likely cache hit)
    result2=$(curl -o /dev/null -s -w \
        "  DNS: %{time_namelookup}s  Connect: %{time_connect}s  TTFB: %{time_starttransfer}s  Total: %{time_total}s  Cache: %header{x-cache}" \
        --max-time 10 "$url" 2>/dev/null)
    echo "  Request 2: $result2"
done
```

### Lab 2: CloudFront Cache Header Analysis
```bash
#!/bin/bash
# Analyze CDN cache headers from a CloudFront distribution

analyze_cdn_headers() {
    local url=$1
    echo "CDN Header Analysis: $url"
    echo "----------------------------"
    
    headers=$(curl -sI "$url" 2>/dev/null)
    
    # CloudFront specific
    x_cache=$(echo "$headers" | grep -i "x-cache:" | head -1)
    x_amz=$(echo "$headers" | grep -i "x-amz-cf-pop:" | head -1)
    x_req_id=$(echo "$headers" | grep -i "x-amz-cf-id:" | head -1)
    
    # Standard cache headers
    cache_ctrl=$(echo "$headers" | grep -i "cache-control:" | head -1)
    age=$(echo "$headers" | grep -i "^age:" | head -1)
    etag=$(echo "$headers" | grep -i "^etag:" | head -1)
    
    echo "  ${x_cache:-X-Cache: not present}"
    echo "  ${x_amz:-X-Amz-CF-Pop: not present (not CloudFront)}"
    echo "  ${cache_ctrl:-Cache-Control: not present}"
    echo "  ${age:-Age: not present}"
    echo "  ${etag:-ETag: not present}"
    
    # Determine cache status
    if echo "$x_cache" | grep -qi "Hit"; then
        echo "  → CACHE HIT ✓"
    elif echo "$x_cache" | grep -qi "Miss"; then
        echo "  → CACHE MISS ✗"
    elif echo "$x_cache" | grep -qi "RefreshHit"; then
        echo "  → REFRESH HIT (stale content revalidated)"
    fi
}

for url in "https://aws.amazon.com" "https://docs.aws.amazon.com" "https://d111111abcdef8.cloudfront.net/"; do
    analyze_cdn_headers "$url"
    echo ""
done
```

### Lab 3: Global Latency Tester
```python
#!/usr/bin/env python3
import subprocess, time

def measure_latency(host, count=5):
    try:
        result = subprocess.run(
            ['ping', '-c', str(count), '-W', '3', host],
            capture_output=True, text=True, timeout=30
        )
        for line in result.stdout.split('\n'):
            if 'rtt min' in line or 'round-trip' in line:
                parts = line.split('/')
                if len(parts) >= 5:
                    return float(parts[4])
    except:
        pass
    return None

# CDN edge endpoints (anycast IPs resolve to nearest PoP)
endpoints = {
    "Cloudflare DNS (1.1.1.1)": "1.1.1.1",
    "Google DNS (8.8.8.8)": "8.8.8.8",
    "Quad9 DNS (9.9.9.9)": "9.9.9.9",
    "AWS DNS (205.251.196.1)": "205.251.196.1",
    "Cloudflare CDN": "www.cloudflare.com",
    "AWS CloudFront": "d111111abcdef8.cloudfront.net",
    "Google CDN": "www.gstatic.com",
}

print("Global CDN/DNS Latency Test (Anycast demonstration)")
print("=" * 55)
print(f"{'Service':<30} {'Latency':>10}")
print("-" * 45)

results = {}
for name, host in endpoints.items():
    latency = measure_latency(host, count=3)
    results[name] = latency
    if latency:
        bar = "█" * int(latency / 5)
        print(f"{name:<30} {latency:>8.1f}ms  {bar}")
    else:
        print(f"{name:<30}   timeout")

if results:
    best = min((k for k, v in results.items() if v), key=lambda k: results[k])
    print(f"\nBest latency: {best} ({results[best]:.1f}ms)")
    print("(Anycast routes each query to nearest PoP)")
```

---

## Practical Exercises

### Exercise 1: CDN Cache Simulator
```python
#!/usr/bin/env python3
import hashlib, time, random

class CDNCache:
    def __init__(self, max_size=100, default_ttl=3600):
        self.cache = {}
        self.max_size = max_size
        self.default_ttl = default_ttl
        self.hits = 0
        self.misses = 0

    def _key(self, url): return hashlib.md5(url.encode()).hexdigest()

    def get(self, url):
        key = self._key(url)
        if key in self.cache:
            entry = self.cache[key]
            if time.time() < entry['expires']:
                self.hits += 1
                entry['age'] = int(time.time() - entry['cached_at'])
                return entry
            else:
                del self.cache[key]  # expired
        self.misses += 1
        return None

    def put(self, url, content, ttl=None, cache_control="public"):
        if "no-store" in cache_control or "private" in cache_control:
            return False
        key = self._key(url)
        ttl = ttl or self.default_ttl
        self.cache[key] = {
            "url": url, "content": content,
            "cached_at": time.time(),
            "expires": time.time() + ttl,
            "ttl": ttl, "age": 0
        }
        return True

    def stats(self):
        total = self.hits + self.misses
        hit_rate = (self.hits / total * 100) if total else 0
        return {"hits": self.hits, "misses": self.misses, "hit_rate": f"{hit_rate:.1f}%"}

# Simulate CDN traffic
cdn = CDNCache(default_ttl=60)

# Pre-populate common assets
assets = ["/index.html", "/style.css", "/app.js", "/logo.png"]
for asset in assets:
    cdn.put(f"https://example.com{asset}", f"content of {asset}", ttl=3600)

# Simulate 50 requests (80% to cached content, 20% to uncached)
urls = [f"https://example.com{random.choice(assets)}" for _ in range(40)]
urls += [f"https://example.com/dynamic/{i}" for i in range(10)]
random.shuffle(urls)

print("CDN Traffic Simulation (50 requests)")
print("=" * 45)
for url in urls[:10]:  # show first 10
    entry = cdn.get(url)
    if entry:
        print(f"  HIT  {url[-30:]:<32} age={entry['age']}s")
    else:
        content = f"fetched content for {url}"
        cached = cdn.put(url, content, ttl=300)
        print(f"  MISS {url[-30:]:<32} {'(cached)' if cached else '(not cacheable)'}")

stats = cdn.stats()
print(f"\nStats: {stats}")
```

### Exercise 2: CloudFront Config Generator
```python
#!/usr/bin/env python3
import json

def generate_cloudfront_config(domain, s3_bucket, alb_domain, env="prod"):
    return {
        "Comment": f"CloudFront distribution for {domain}",
        "Origins": [
            {
                "Id": "S3Origin",
                "DomainName": f"{s3_bucket}.s3.amazonaws.com",
                "S3OriginConfig": {"OriginAccessIdentity": ""}
            },
            {
                "Id": "ALBOrigin",
                "DomainName": alb_domain,
                "CustomOriginConfig": {
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only"
                }
            }
        ],
        "DefaultCacheBehavior": {
            "TargetOriginId": "S3Origin",
            "ViewerProtocolPolicy": "redirect-to-https",
            "Compress": True,
            "AllowedMethods": ["GET", "HEAD"],
            "CachedMethods": ["GET", "HEAD"],
            "DefaultTTL": 86400,
            "MaxTTL": 31536000
        },
        "CacheBehaviors": [
            {
                "PathPattern": "/api/*",
                "TargetOriginId": "ALBOrigin",
                "ViewerProtocolPolicy": "https-only",
                "DefaultTTL": 0,
                "MaxTTL": 0,
                "AllowedMethods": ["DELETE","GET","HEAD","OPTIONS","PATCH","POST","PUT"]
            },
            {
                "PathPattern": "/static/*",
                "TargetOriginId": "S3Origin",
                "ViewerProtocolPolicy": "redirect-to-https",
                "DefaultTTL": 2592000,
                "MaxTTL": 31536000,
                "Compress": True
            }
        ],
        "Aliases": [domain, f"www.{domain}"],
        "HttpVersion": "http2and3",
        "IsIPV6Enabled": True,
        "PriceClass": "PriceClass_All" if env == "prod" else "PriceClass_100"
    }

config = generate_cloudfront_config(
    domain="example.com",
    s3_bucket="example-static",
    alb_domain="app-alb-123.us-east-1.elb.amazonaws.com"
)
print(json.dumps(config, indent=2))
```

---

## Sample Exercises

1. **Cache-Control strategy:** You have: a homepage (updated daily), product images (updated rarely), user cart data (per-user, never shared), and API responses (real-time). Write the Cache-Control header for each.
2. **Anycast vs Unicast:** Explain why DNS servers like 8.8.8.8 can be reached with ultra-low latency from anywhere in the world despite being a single IP.
3. **CloudFront behavior:** A user requests `/api/user/profile`. Your CloudFront has two behaviors: `/*` cached 24h, `/api/*` not cached. Which behavior applies? Why?
4. **Cache invalidation cost:** You update your entire website (1000 files). Compared to using versioned filenames (e.g., `app.v2.js`), why is cache invalidation a worse strategy?
5. **CDN for dynamic content:** A common misconception is that CDNs only work for static files. Describe two ways CDNs accelerate dynamic content.

## Solutions

1. **Cache-Control headers:**
   - Homepage (updated daily): `Cache-Control: public, max-age=3600, s-maxage=86400` (browser 1h, CDN 24h)
   - Product images (rarely updated): `Cache-Control: public, max-age=31536000, immutable` (1 year, immutable — use versioned URLs)
   - User cart data: `Cache-Control: private, no-store` (never cache anywhere)
   - API responses: `Cache-Control: no-cache` (always revalidate with origin, or `max-age=0`)

2. **Anycast magic:** 8.8.8.8 is advertised from hundreds of Google PoPs worldwide via BGP. Each ISP's router selects the route to 8.8.8.8 with the shortest AS path — which is the nearest Google PoP. Your DNS query travels only to the nearest Google edge node, not to a single server in the USA. This is BGP anycast in action.

3. **CloudFront behavior matching:** `/api/user/profile` — CloudFront matches the **most specific** path pattern first. `/api/*` is more specific than `/*`, so the `/api/*` behavior (not cached) applies. Most specific path pattern always wins.

4. **Invalidation vs versioned filenames:**
   - Invalidation costs: AWS charges $0.005 per path per 1000 (first 1000 free). 1000 files = multiple requests. More importantly, there's a propagation delay (5-10 min) during which users may get stale content.
   - Versioned filenames (`app.v2.js`): The old URL `app.v1.js` stays cached forever — no invalidation needed. The new URL is immediately fresh since it was never cached. Instant, free, zero propagation delay. Far superior strategy.

5. **CDN for dynamic content:**
   - **Edge compute (Lambda@Edge, Cloudflare Workers):** Run business logic at the PoP — personalize content, authenticate, A/B test, transform responses — without the round trip to origin.
   - **TCP connection optimization:** CDN maintains persistent connections to origin over the backbone network. The user's TCP handshake terminates at the nearest PoP (5ms), then the PoP reuses a pre-warmed connection to origin — eliminating the full RTT for connection setup on every request.

## Completion Checklist
- [ ] Explain cache hit/miss and cache-control header directives
- [ ] Understand Anycast routing and its CDN applications
- [ ] Configure CloudFront with multiple origins and cache behaviors
- [ ] Implement cache invalidation strategies
- [ ] Compare CloudFront, Cloud CDN, and Azure CDN
- [ ] Design a global load balancing strategy using Route 53 or Global Accelerator

## Key Takeaways
- CDNs reduce latency by serving cached content from PoPs near users
- Anycast routes the same IP to the geographically nearest node using BGP
- Cache-Control headers control what is cached, where, and for how long
- Use versioned filenames instead of cache invalidation for static assets
- CloudFront supports both static (S3) and dynamic (ALB) origins with different cache behaviors
- Global Accelerator uses Anycast to accelerate non-HTTP traffic and provides static IPs

## Next Steps
Proceed to [Day 28: Network Performance, QoS, Bottlenecks, Cloud Monitoring](../Day_28/notes_and_exercises.md).