# Day 18: HTTP, HTTPS, TLS, Certificates, Reverse Proxy, API Gateways

## Learning Objectives
By the end of Day 18, you will:
- Master HTTP/1.1, HTTP/2, and HTTP/3 differences
- Understand the full TLS handshake and certificate chain
- Configure Nginx as a reverse proxy with TLS termination
- Use Let's Encrypt for free automated certificates
- Understand API Gateway patterns

**Estimated Time:** 3-4 hours

---

## Notes

### HTTP Protocol Evolution

| Version | Year | Key Features |
|---------|------|-------------|
| HTTP/0.9 | 1991 | GET only, no headers |
| HTTP/1.0 | 1996 | Headers, status codes, new connection per request |
| HTTP/1.1 | 1997 | Persistent connections, pipelining, chunked encoding |
| HTTP/2 | 2015 | Binary, multiplexing, header compression (HPACK), server push |
| HTTP/3 | 2022 | QUIC (UDP), 0-RTT, no head-of-line blocking |

### HTTP/1.1 Deep Dive

#### Request Structure
```
GET /api/users?page=1 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Accept: application/json
User-Agent: curl/7.81.0
Connection: keep-alive
```

#### Response Structure
```
HTTP/1.1 200 OK
Date: Mon, 10 May 2026 12:00:00 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 342
Cache-Control: max-age=3600
X-Request-ID: abc-123
X-RateLimit-Remaining: 99

{"users": [...]}
```

#### Important HTTP Headers

**Request headers:**
- `Host` — required in HTTP/1.1; identifies virtual host
- `Authorization` — credentials (Bearer, Basic, Digest)
- `Content-Type` — body MIME type (application/json)
- `Accept` — preferred response format
- `If-Modified-Since` / `ETag` — conditional requests for caching
- `X-Forwarded-For` — client IP when behind proxy

**Response headers:**
- `Cache-Control` — caching directives
- `Set-Cookie` — set browser cookies
- `Location` — redirect target (301/302)
- `Strict-Transport-Security` — force HTTPS
- `Access-Control-Allow-Origin` — CORS policy

### HTTPS and TLS

#### TLS 1.3 Handshake (Simplified)
```
Client                            Server
  |                                  |
  |── ClientHello ─────────────────> |   (TLS version, ciphers, client random)
  |                                  |
  |<─ ServerHello + Certificate ─── |   (chosen cipher, server random, cert)
  |<─ CertificateVerify ─────────── |   (signature proving private key ownership)
  |<─ Finished ─────────────────── |
  |                                  |
  |── Finished ─────────────────── > |   (handshake done — 1 RTT!)
  |                                  |
  |<══ Encrypted Application Data ══>|
```

TLS 1.3 improvements over 1.2:
- 1-RTT handshake (vs 2-RTT)
- 0-RTT session resumption (for repeat connections)
- Removed insecure algorithms (RSA key exchange, RC4, MD5, SHA-1)
- Forward secrecy mandatory

#### Certificate Chain
```
Root CA (self-signed, in OS/browser trust store)
    ↓ signs
Intermediate CA
    ↓ signs
End-Entity Certificate (your domain)
```

The server sends both the end-entity cert and the intermediate cert. The client validates by chaining up to a trusted root CA.

#### Certificate Fields
- **Subject:** CN=www.example.com (or SAN: DNS:example.com, DNS:www.example.com)
- **Issuer:** Let's Encrypt Authority X3
- **Valid:** Not before / Not after
- **Public key:** RSA 2048 or ECDSA P-256
- **SANs:** Subject Alternative Names — all covered hostnames
- **Key usage:** Digital signature, key encipherment

### Let's Encrypt and Certbot

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain certificate (Nginx plugin)
sudo certbot --nginx -d example.com -d www.example.com

# Standalone mode (stop Nginx first)
sudo certbot certonly --standalone -d example.com

# Manual DNS challenge (wildcard certs)
sudo certbot certonly --manual --preferred-challenges dns -d "*.example.com"

# Auto-renewal (cron or systemd timer)
sudo certbot renew --dry-run

# View certificate info
sudo certbot certificates
openssl x509 -in /etc/letsencrypt/live/example.com/cert.pem -text -noout
```

### Nginx Reverse Proxy with TLS

```nginx
# /etc/nginx/conf.d/app.conf

# Redirect HTTP → HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # TLS certificates
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Modern TLS settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    # Proxy to backend
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 60s;
    }

    # Static files (served by Nginx directly)
    location /static/ {
        root /var/www/app;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### API Gateway

An API Gateway is the **single entry point** for client-to-microservice communication.

#### Core Functions
- **Routing:** Forward requests to correct microservice
- **Authentication:** Validate JWT, API keys, OAuth tokens
- **Rate limiting:** Throttle requests per client
- **SSL termination:** Handle TLS, pass HTTP internally
- **Request transformation:** Modify headers, body, path
- **Response caching:** Reduce backend load
- **Logging & tracing:** Centralized observability

#### API Gateway Patterns

```
Client → API Gateway → Microservice A
                    → Microservice B
                    → Microservice C
```

**AWS API Gateway:**
- REST API, HTTP API, WebSocket API
- Lambda integration, HTTP proxy, AWS service integration
- Usage plans and API keys for rate limiting
- Stages: dev, staging, prod

**Kong (open source):**
```yaml
# kong.yml
services:
  - name: user-service
    url: http://user-service:3000
    routes:
      - paths: ["/api/users"]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
      - name: jwt
```

**Nginx as API Gateway:**
```nginx
upstream user_service   { server 10.0.1.10:3000; }
upstream order_service  { server 10.0.1.11:3001; }
upstream product_service{ server 10.0.1.12:3002; }

server {
    listen 443 ssl;
    server_name api.example.com;

    location /api/users/    { proxy_pass http://user_service; }
    location /api/orders/   { proxy_pass http://order_service; }
    location /api/products/ { proxy_pass http://product_service; }
}
```

---

## Hands-On Labs

### Lab 1: Inspect TLS Certificates
```bash
# Check a live certificate
echo | openssl s_client -connect google.com:443 -servername google.com 2>/dev/null \
  | openssl x509 -noout -text | grep -E "Subject:|Issuer:|Not After|DNS:"

# Check certificate expiry (days remaining)
echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -enddate

# Full chain view
echo | openssl s_client -connect example.com:443 -showcerts 2>/dev/null \
  | grep -E "^subject|^issuer"

# Check local cert file
openssl x509 -in /etc/ssl/certs/ca-certificates.crt -noout -text 2>/dev/null | head -20

# Test TLS versions supported
for version in ssl2 ssl3 tls1 tls1_1 tls1_2 tls1_3; do
    result=$(openssl s_client -connect google.com:443 -$version 2>&1 | grep -i "handshake")
    echo "$version: ${result:-unsupported}"
done 2>/dev/null
```

### Lab 2: Self-Signed Cert for Testing
```bash
# Generate self-signed certificate for testing
openssl req -x509 -newkey rsa:4096 -keyout /tmp/test.key -out /tmp/test.crt \
  -days 365 -nodes \
  -subj "/C=IN/ST=Karnataka/L=Bengaluru/O=TestOrg/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# View the certificate
openssl x509 -in /tmp/test.crt -noout -text | grep -E "Subject:|Issuer:|DNS:|Not After"

# Start a test HTTPS server with Python
python3 -c "
import ssl, http.server
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain('/tmp/test.crt', '/tmp/test.key')
httpd = http.server.HTTPServer(('localhost', 4443), http.server.SimpleHTTPRequestHandler)
httpd.socket = ctx.wrap_socket(httpd.socket, server_side=True)
print('HTTPS server at https://localhost:4443')
httpd.serve_forever()
" &
HTTPS_PID=$!
sleep 1
curl -k https://localhost:4443 > /dev/null && echo "HTTPS server working"
kill $HTTPS_PID 2>/dev/null
```

### Lab 3: HTTP Headers Analysis
```bash
#!/bin/bash
# Analyze HTTP security headers for a website

check_headers() {
    local url=$1
    echo "Security Header Analysis: $url"
    echo "=================================="

    headers=$(curl -sI "$url" 2>/dev/null)

    check_header() {
        local header=$1
        local name=$2
        if echo "$headers" | grep -qi "^$header"; then
            value=$(echo "$headers" | grep -i "^$header" | head -1 | cut -d: -f2- | xargs)
            echo "  ✓ $name: $value"
        else
            echo "  ✗ $name: MISSING"
        fi
    }

    check_header "strict-transport-security" "HSTS"
    check_header "x-frame-options" "X-Frame-Options"
    check_header "x-content-type-options" "X-Content-Type-Options"
    check_header "content-security-policy" "CSP"
    check_header "x-xss-protection" "XSS Protection"
    check_header "referrer-policy" "Referrer-Policy"
    check_header "permissions-policy" "Permissions-Policy"
}

check_headers "https://google.com"
```

---

## Practical Exercises

### Exercise 1: HTTP Request Parser
```python
#!/usr/bin/env python3
import urllib.request
import ssl

def inspect_http(url):
    ctx = ssl.create_default_context()
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE

    req = urllib.request.Request(url, headers={"User-Agent": "NetworkLearner/1.0"})
    try:
        with urllib.request.urlopen(req, context=ctx, timeout=5) as resp:
            print(f"URL: {url}")
            print(f"Status: {resp.status} {resp.reason}")
            print(f"HTTP Version: HTTP/1.x")
            print("\nResponse Headers:")
            for k, v in resp.headers.items():
                print(f"  {k}: {v}")
    except Exception as e:
        print(f"Error: {e}")

inspect_http("https://httpbin.org/get")
```

### Exercise 2: Certificate Expiry Monitor
```python
#!/usr/bin/env python3
import ssl, socket
from datetime import datetime

def check_cert_expiry(hostname, port=443):
    ctx = ssl.create_default_context()
    try:
        with socket.create_connection((hostname, port), timeout=5) as sock:
            with ctx.wrap_socket(sock, server_hostname=hostname) as ssock:
                cert = ssock.getpeercert()
                expiry_str = cert['notAfter']
                expiry = datetime.strptime(expiry_str, '%b %d %H:%M:%S %Y %Z')
                days_left = (expiry - datetime.utcnow()).days
                subject = dict(x[0] for x in cert['subject'])
                cn = subject.get('commonName', hostname)
                status = "✓ OK" if days_left > 30 else ("⚠ EXPIRING SOON" if days_left > 0 else "✗ EXPIRED")
                print(f"{hostname:<35} {status:20} {days_left:4}d left  CN={cn}")
    except Exception as e:
        print(f"{hostname:<35} ✗ ERROR: {e}")

hosts = ["google.com", "github.com", "example.com", "httpbin.org"]
print(f"{'Host':<35} {'Status':<20} {'Days'} {'Details'}")
print("-" * 80)
for h in hosts:
    check_cert_expiry(h)
```

### Exercise 3: Nginx Config Validator
```bash
#!/bin/bash
# Validate common Nginx reverse proxy configs

check_nginx_config() {
    if ! command -v nginx &>/dev/null; then
        echo "Nginx not installed"
        return
    fi
    sudo nginx -t 2>&1
    if [ $? -eq 0 ]; then
        echo "✓ Nginx config is valid"
    else
        echo "✗ Nginx config has errors"
    fi
}

show_nginx_status() {
    echo "Active virtual hosts:"
    sudo nginx -T 2>/dev/null | grep "server_name" | sort -u

    echo -e "\nListen ports:"
    sudo nginx -T 2>/dev/null | grep "listen" | sort -u

    echo -e "\nProxy upstreams:"
    sudo nginx -T 2>/dev/null | grep "proxy_pass" | sort -u
}

check_nginx_config
show_nginx_status
```

---

## Sample Exercises

1. **TLS Troubleshooting:** Users report "NET::ERR_CERT_AUTHORITY_INVALID". What are the possible causes?
2. **HTTP/2 Benefits:** List three specific ways HTTP/2 improves on HTTP/1.1 for a page with 50 assets.
3. **Reverse Proxy Config:** Write an Nginx config that proxies `/api/` to `localhost:3000` and serves `/static/` from `/var/www/app/static` directly.
4. **API Gateway:** Compare AWS API Gateway HTTP API vs REST API. When would you pick each?
5. **Certificate Chain:** Explain what happens if the intermediate certificate is missing from the server's TLS config.

## Solutions

1. **ERR_CERT_AUTHORITY_INVALID causes:** Self-signed cert not in trust store; missing intermediate certificate in server config; cert issued by untrusted CA; cert expired; hostname mismatch (CN/SAN doesn't match the URL).
2. **HTTP/2 improvements for 50-asset page:** (a) Multiplexing — all 50 assets sent over one connection simultaneously, no head-of-line blocking; (b) Header compression (HPACK) — repeated headers across 50 requests compressed significantly; (c) Server push — server can proactively send CSS/JS before browser requests them.
3. **Nginx config:** See `location /api/ { proxy_pass http://localhost:3000; ... }` and `location /static/ { root /var/www/app; }` patterns in the notes above.
4. **HTTP API vs REST API:** HTTP API — cheaper, lower latency, JWT/OIDC auth, ideal for simple proxying; REST API — more features (usage plans, API keys, request validation, caching, WAF, more transform options), higher cost. Choose REST API for complex enterprise APIs; HTTP API for simple, cost-sensitive use cases.
5. **Missing intermediate cert:** Browser may fail to validate the chain if it doesn't have the intermediate cert cached. Some browsers recover by fetching the cert via AIA (Authority Information Access) extension, but many will show a "certificate not trusted" error. Always include the full chain (`fullchain.pem` in Let's Encrypt).

## Completion Checklist
- [ ] Know HTTP/1.1 vs HTTP/2 vs HTTP/3 differences
- [ ] Explain TLS 1.3 handshake steps
- [ ] Obtain and renew certificates with certbot
- [ ] Configure Nginx as a TLS-terminating reverse proxy
- [ ] Understand API Gateway patterns and AWS API Gateway types
- [ ] Audit HTTP security headers for a website

## Key Takeaways
- HTTPS = HTTP + TLS; TLS handles encryption and authentication
- TLS 1.3 reduces handshake to 1 RTT and removes insecure algorithms
- Certificate chains must include intermediate certs; use fullchain.pem
- Nginx is a powerful, lightweight reverse proxy and TLS terminator
- API Gateways centralize cross-cutting concerns: auth, rate limiting, routing, observability

## Next Steps
Proceed to [Day 19: Network Monitoring & Troubleshooting](../Day_19/notes_and_exercises.md).