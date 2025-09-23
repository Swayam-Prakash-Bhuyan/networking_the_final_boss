# Day 08: Layer 7 - Application Layer (HTTP, DNS, SMTP, REST, APIs, Cloud Apps)

## Learning Objectives
By the end of Day 8, you will:
- Master HTTP protocol and web communication
- Understand DNS resolution and domain name system
- Learn email protocols (SMTP, POP3, IMAP)
- Explore REST APIs and modern web services
- Apply application layer concepts to cloud applications

**Estimated Time:** 3-4 hours

## Notes

### Application Layer Overview
- **Purpose**: Provides network services directly to applications and end-users
- **Functions**: User interface, network process to application, data formatting
- **Protocols**: HTTP/HTTPS, DNS, SMTP, FTP, SSH, SNMP, DHCP
- **Services**: Web browsing, email, file transfer, remote access

### HTTP Protocol

#### HTTP Request/Response Cycle
```
Client (Browser)                    Server
       |                              |
       | 1. HTTP Request               |
       |------------------------------>|
       |   GET /index.html HTTP/1.1    |
       |   Host: www.example.com       |
       |   User-Agent: Mozilla/5.0     |
       |                              |
       | 2. HTTP Response              |
       |<------------------------------|
       |   HTTP/1.1 200 OK             |
       |   Content-Type: text/html     |
       |   Content-Length: 1234        |
       |   <html>...</html>            |
```

#### HTTP Methods
- **GET**: Retrieve data from server
- **POST**: Send data to server (create)
- **PUT**: Update existing resource
- **DELETE**: Remove resource
- **PATCH**: Partial update
- **HEAD**: Get headers only
- **OPTIONS**: Check allowed methods

#### HTTP Status Codes
- **1xx**: Informational (100 Continue)
- **2xx**: Success (200 OK, 201 Created)
- **3xx**: Redirection (301 Moved, 302 Found)
- **4xx**: Client Error (400 Bad Request, 404 Not Found)
- **5xx**: Server Error (500 Internal Error, 503 Unavailable)

### DNS (Domain Name System)

#### DNS Resolution Process
```
User Types: www.example.com
     |
     v
[Local DNS Cache] ──(miss)──> [Local DNS Resolver]
     |                              |
     v                              v
[Hosts File] ──(miss)──> [Root DNS Server (.)]
                              |
                              v
                         [TLD DNS Server (.com)]
                              |
                              v
                         [Authoritative DNS Server]
                              |
                              v
                         Returns IP: 93.184.216.34
```

#### DNS Record Types
- **A**: IPv4 address mapping
- **AAAA**: IPv6 address mapping
- **CNAME**: Canonical name (alias)
- **MX**: Mail exchange server
- **NS**: Name server
- **TXT**: Text records (SPF, DKIM)
- **PTR**: Reverse DNS lookup
- **SOA**: Start of authority

### Email Protocols

#### SMTP (Simple Mail Transfer Protocol)
```
Email Flow:
[User] ──> [Mail Client] ──SMTP──> [Outgoing Mail Server]
                                         |
                                         v
                                   [Internet]
                                         |
                                         v
                              [Receiving Mail Server] <──SMTP── [Other Mail Servers]
                                         |
                                         v
                              [User's Mailbox]
                                         ^
                                         |
                              POP3/IMAP [Mail Client] <── [User]
```

#### Email Protocol Comparison
| Protocol | Port | Purpose | Connection |
|----------|------|---------|------------|
| SMTP | 25/587 | Send email | Outgoing |
| POP3 | 110/995 | Download email | Incoming |
| IMAP | 143/993 | Sync email | Incoming |

### REST APIs

#### REST Principles
- **Stateless**: Each request independent
- **Resource-based**: URLs represent resources
- **HTTP Methods**: Use appropriate verbs
- **Representations**: JSON, XML data formats
- **HATEOAS**: Hypermedia as engine of application state

#### RESTful API Design
```
Resource: Users
GET    /api/users      - List all users
GET    /api/users/123  - Get specific user
POST   /api/users      - Create new user
PUT    /api/users/123  - Update user
DELETE /api/users/123  - Delete user
```

### Cloud Application Protocols

#### Modern Web Technologies
- **WebSockets**: Real-time bidirectional communication
- **GraphQL**: Query language for APIs
- **gRPC**: High-performance RPC framework
- **Server-Sent Events**: Server-to-client streaming
- **WebRTC**: Peer-to-peer communication

## Hands-On Labs

### Lab 1: HTTP Analysis
```bash
# Basic HTTP request with curl
curl -v http://httpbin.org/get

# HTTP POST request
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"test","value":"data"}' \
     http://httpbin.org/post

# Check HTTP headers
curl -I https://google.com

# Follow redirects
curl -L https://google.com

# Test different HTTP methods
curl -X GET http://httpbin.org/get
curl -X POST http://httpbin.org/post
curl -X PUT http://httpbin.org/put
curl -X DELETE http://httpbin.org/delete
```

### Lab 2: DNS Resolution Testing
```bash
# Basic DNS lookup
nslookup google.com

# Detailed DNS query
dig google.com

# Query specific record types
dig google.com A
dig google.com AAAA
dig google.com MX
dig google.com TXT

# Reverse DNS lookup
dig -x 8.8.8.8

# Trace DNS resolution path
dig +trace google.com

# Check local DNS cache (varies by system)
systemd-resolve --status 2>/dev/null || echo "systemd-resolve not available"
```

### Lab 3: Email Protocol Testing
```bash
# Test SMTP connection
telnet smtp.gmail.com 587
# Commands to try in telnet:
# EHLO localhost
# STARTTLS
# QUIT

# Test POP3 connection (if available)
# telnet pop.gmail.com 995

# Check MX records for domain
dig example.com MX

# Verify email server connectivity
nc -zv smtp.gmail.com 587
nc -zv imap.gmail.com 993
```

## Practical Exercises

### Exercise 1: Simple HTTP Server
```python
#!/usr/bin/env python3
# Simple HTTP server for testing

from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class SimpleHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        
        response = {
            'method': 'GET',
            'path': self.path,
            'headers': dict(self.headers)
        }
        
        self.wfile.write(json.dumps(response, indent=2).encode())
    
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        
        response = {
            'method': 'POST',
            'path': self.path,
            'data': post_data.decode('utf-8')
        }
        
        self.wfile.write(json.dumps(response, indent=2).encode())

if __name__ == '__main__':
    server = HTTPServer(('localhost', 8080), SimpleHandler)
    print("Server running on http://localhost:8080")
    server.serve_forever()
```

### Exercise 2: DNS Query Tool
```bash
#!/bin/bash
# DNS query tool

query_dns() {
    local domain=$1
    local record_type=${2:-A}
    
    echo "Querying $domain for $record_type records:"
    echo "----------------------------------------"
    
    # Use dig if available, otherwise nslookup
    if command -v dig &> /dev/null; then
        dig $domain $record_type +short
    else
        nslookup -type=$record_type $domain
    fi
}

# Query multiple record types
DOMAIN=${1:-google.com}

echo "DNS Analysis for: $DOMAIN"
echo "========================="

query_dns $DOMAIN A
echo
query_dns $DOMAIN AAAA
echo
query_dns $DOMAIN MX
echo
query_dns $DOMAIN TXT
```

### Exercise 3: REST API Tester
```bash
#!/bin/bash
# REST API testing tool

API_BASE="https://jsonplaceholder.typicode.com"

test_get() {
    echo "Testing GET request:"
    curl -s "$API_BASE/posts/1" | jq '.' 2>/dev/null || curl -s "$API_BASE/posts/1"
    echo
}

test_post() {
    echo "Testing POST request:"
    curl -s -X POST "$API_BASE/posts" \
         -H "Content-Type: application/json" \
         -d '{"title":"Test","body":"Test body","userId":1}' | \
         jq '.' 2>/dev/null || echo "Posted successfully"
    echo
}

test_put() {
    echo "Testing PUT request:"
    curl -s -X PUT "$API_BASE/posts/1" \
         -H "Content-Type: application/json" \
         -d '{"id":1,"title":"Updated","body":"Updated body","userId":1}' | \
         jq '.' 2>/dev/null || echo "Updated successfully"
    echo
}

test_delete() {
    echo "Testing DELETE request:"
    curl -s -X DELETE "$API_BASE/posts/1"
    echo "Delete request sent"
    echo
}

echo "REST API Testing"
echo "================"
test_get
test_post
test_put
test_delete
```

## Sample Exercises

1. **HTTP Header Analysis**: Analyze HTTP headers from different websites and explain their security implications.

2. **DNS Troubleshooting**: Diagnose and resolve common DNS issues in a network environment.

3. **Email Server Configuration**: Set up and configure email server components (SMTP, IMAP).

4. **REST API Design**: Design a RESTful API for a simple application with proper resource modeling.

5. **Web Performance Optimization**: Analyze and optimize web application performance at the application layer.

## Solutions

1. **HTTP Header Analysis:**
   ```
   Security Headers to Check:
   - Strict-Transport-Security: Enforces HTTPS
   - X-Frame-Options: Prevents clickjacking
   - X-Content-Type-Options: Prevents MIME sniffing
   - Content-Security-Policy: Prevents XSS attacks
   - X-XSS-Protection: Browser XSS filtering
   ```

2. **DNS Troubleshooting Steps:**
   ```
   1. Check local DNS cache: systemctl flush-dns
   2. Test with different DNS servers: dig @8.8.8.8 domain.com
   3. Verify DNS server connectivity: nc -zv 8.8.8.8 53
   4. Check hosts file: cat /etc/hosts
   5. Trace DNS resolution: dig +trace domain.com
   ```

3. **Email Server Components:**
   ```
   SMTP Server (Outgoing):
   - Postfix, Sendmail, Exim
   - Port 25 (standard), 587 (submission)
   
   IMAP/POP3 Server (Incoming):
   - Dovecot, Courier
   - IMAP: 143/993, POP3: 110/995
   
   Webmail Interface:
   - Roundcube, SquirrelMail
   ```

4. **REST API Design Example:**
   ```
   Blog API:
   GET    /api/posts           - List posts
   GET    /api/posts/{id}      - Get post
   POST   /api/posts           - Create post
   PUT    /api/posts/{id}      - Update post
   DELETE /api/posts/{id}      - Delete post
   GET    /api/posts/{id}/comments - Get comments
   POST   /api/posts/{id}/comments - Add comment
   ```

5. **Web Performance Optimization:**
   ```
   Application Layer Optimizations:
   - Enable compression (gzip, brotli)
   - Use CDN for static content
   - Implement caching headers
   - Minimize HTTP requests
   - Optimize images and assets
   - Use HTTP/2 or HTTP/3
   ```

## Completion Checklist
- [ ] Understand HTTP protocol and methods
- [ ] Know DNS resolution process and record types
- [ ] Familiar with email protocols (SMTP, POP3, IMAP)
- [ ] Can design and test REST APIs
- [ ] Understand modern web technologies and protocols
- [ ] Can troubleshoot application layer issues

## Key Takeaways
- Application layer provides direct interface between network and applications
- HTTP is the foundation of web communication with various methods and status codes
- DNS translates human-readable names to IP addresses through hierarchical resolution
- Email systems use multiple protocols for sending and receiving messages
- REST APIs provide standardized way for applications to communicate
- Modern web applications use advanced protocols for real-time and efficient communication

## Next Steps
Proceed to [Day 9: TCP/IP Model & OSI Comparison](../Day_09/notes_and_exercises.md) to understand the relationship between theoretical models and practical implementation.