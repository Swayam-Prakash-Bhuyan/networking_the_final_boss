# Day 05: Layer 4 - Transport (TCP, UDP, Ports, Sockets, Troubleshooting)

## Learning Objectives
By the end of Day 5, you will:
- Master TCP and UDP protocol differences and use cases
- Understand port numbers, socket concepts, and connection management
- Learn transport layer troubleshooting techniques and tools
- Explore flow control, congestion control, and error recovery mechanisms
- Apply transport layer optimization strategies

**Estimated Time:** 3-4 hours

## Notes

### Transport Layer Functions
- **Segmentation:** Break application data into manageable segments
- **Reassembly:** Reconstruct original data from received segments
- **Connection Management:** Establish, maintain, and terminate connections
- **Flow Control:** Prevent overwhelming the receiver with data
- **Error Recovery:** Detect and recover from transmission errors
- **Multiplexing:** Allow multiple applications to use network simultaneously

### TCP (Transmission Control Protocol)

#### TCP Characteristics
- **Connection-oriented:** Three-way handshake before data transfer
- **Reliable:** Guarantees delivery and correct order
- **Flow Control:** Sliding window mechanism prevents buffer overflow
- **Error Detection:** Checksums and acknowledgments ensure data integrity
- **Congestion Control:** Adapts transmission rate to network conditions

#### TCP Header Structure (20 bytes minimum)
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### TCP Connection States
- **CLOSED:** No connection exists
- **LISTEN:** Server waiting for connection requests
- **SYN-SENT:** Client sent connection request
- **SYN-RECEIVED:** Server received connection request
- **ESTABLISHED:** Connection active and data can flow
- **FIN-WAIT-1:** Connection termination initiated
- **FIN-WAIT-2:** Waiting for remote termination
- **CLOSE-WAIT:** Remote end has shut down connection
- **LAST-ACK:** Waiting for final acknowledgment
- **TIME-WAIT:** Waiting to ensure remote received termination

#### TCP Three-Way Handshake
```
Client                    Server
  |                         |
  |-------SYN(seq=100)----->|  1. Client initiates connection
  |                         |
  |<--SYN-ACK(seq=200,------|  2. Server acknowledges and responds
  |         ack=101)        |
  |                         |
  |-------ACK(ack=201)----->|  3. Client acknowledges server's response
  |                         |
  |    Connection Established|
```

### UDP (User Datagram Protocol)

#### UDP Characteristics
- **Connectionless:** No connection establishment required
- **Unreliable:** No delivery guarantee or ordering
- **Fast:** Minimal overhead and processing
- **Simple:** Basic error detection only
- **Broadcast/Multicast:** Supports one-to-many communication

#### UDP Header Structure (8 bytes)
```
 0      7 8     15 16    23 24    31
+--------+--------+--------+--------+
|     Source      |   Destination   |
|      Port       |      Port       |
+--------+--------+--------+--------+
|                 |                 |
|     Length      |    Checksum     |
+--------+--------+--------+--------+
|                                   |
|              Data                 |
+-----------------------------------+
```

#### TCP vs UDP Comparison
| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Reliable delivery | Best effort |
| Ordering | Maintains order | No ordering guarantee |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Header Size | 20+ bytes | 8 bytes |
| Flow Control | Yes | No |
| Use Cases | Web, email, file transfer | DNS, streaming, gaming |

### Port Numbers and Sockets

#### Port Number Ranges
- **Well-known Ports (0-1023):** System services, require root privileges
- **Registered Ports (1024-49151):** User applications, assigned by IANA
- **Dynamic/Private Ports (49152-65535):** Temporary client connections

#### Common Well-Known Ports
| Port | Protocol | Service |
|------|----------|---------|
| 20/21 | TCP | FTP (data/control) |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| 443 | TCP | HTTPS |
| 993 | TCP | IMAPS |
| 995 | TCP | POP3S |

#### Socket Concept
A socket is an endpoint for network communication identified by:
- **IP Address:** Network location of the device
- **Port Number:** Application identifier on the device
- **Protocol:** TCP or UDP
- **Socket Notation:** IP:Port (e.g., 192.168.1.100:80)

## Hands-On Labs

### Lab 1: TCP Connection Analysis
```bash
# Monitor TCP connections and states
ss -tuln

# Show established connections only
ss -t state established

# Monitor specific port
ss -tlnp | grep :80

# Watch connection count in real-time
watch -n 1 'ss -t state established | wc -l'

# Capture TCP handshake with tcpdump
sudo tcpdump -i any -n port 80 and host google.com &
curl -I http://google.com
sudo pkill tcpdump
```

### Lab 2: UDP Traffic Analysis
```bash
# Monitor UDP sockets
ss -u -a

# Test UDP connectivity with netcat
nc -u google.com 53
# Type some data and press Enter

# Capture DNS queries (UDP port 53)
sudo tcpdump -i any -n port 53 &
nslookup google.com
sudo pkill tcpdump

# Send UDP test packets
echo "test message" | nc -u -w1 127.0.0.1 12345
```

### Lab 3: Port Scanning and Service Detection
```bash
# Install nmap for network scanning
sudo apt install nmap

# Basic TCP port scan
nmap -sT 192.168.1.1

# UDP port scan (slower)
nmap -sU 192.168.1.1

# Service version detection
nmap -sV 192.168.1.1

# Scan specific ports
nmap -p 22,80,443 192.168.1.1

# Check local listening ports
netstat -tlnp
# or modern equivalent
ss -tlnp
```

## Practical Exercises

### Exercise 1: TCP Connection Monitor
```python
#!/usr/bin/env python3
import subprocess
import time
import re

def get_tcp_connections():
    """Get current TCP connection statistics"""
    try:
        result = subprocess.run(['ss', '-t', 'state', 'all'], 
                              capture_output=True, text=True)
        lines = result.stdout.strip().split('\n')[1:]  # Skip header
        
        states = {}
        for line in lines:
            if 'ESTAB' in line:
                states['ESTABLISHED'] = states.get('ESTABLISHED', 0) + 1
            elif 'LISTEN' in line:
                states['LISTEN'] = states.get('LISTEN', 0) + 1
            elif 'TIME-WAIT' in line:
                states['TIME-WAIT'] = states.get('TIME-WAIT', 0) + 1
        
        return states
    except Exception as e:
        return {'ERROR': str(e)}

def monitor_connections():
    """Monitor TCP connections continuously"""
    print("TCP Connection Monitor")
    print("=" * 40)
    
    while True:
        states = get_tcp_connections()
        timestamp = time.strftime('%Y-%m-%d %H:%M:%S')
        
        print(f"\n{timestamp}")
        for state, count in states.items():
            print(f"  {state}: {count}")
        
        time.sleep(5)

if __name__ == "__main__":
    try:
        monitor_connections()
    except KeyboardInterrupt:
        print("\nMonitoring stopped.")
```

### Exercise 2: Simple Port Scanner
```bash
#!/bin/bash
# Simple TCP port scanner

TARGET=${1:-127.0.0.1}
START_PORT=${2:-1}
END_PORT=${3:-1000}
TIMEOUT=${4:-1}

if [ -z "$1" ]; then
    echo "Usage: $0 <target> [start_port] [end_port] [timeout]"
    echo "Example: $0 192.168.1.1 1 1000 1"
    exit 1
fi

echo "Scanning $TARGET from port $START_PORT to $END_PORT"
echo "Timeout: ${TIMEOUT}s"
echo "=" * 50

open_ports=()

for port in $(seq $START_PORT $END_PORT); do
    if timeout $TIMEOUT bash -c "echo >/dev/tcp/$TARGET/$port" 2>/dev/null; then
        echo "Port $port: OPEN"
        open_ports+=($port)
    fi
done

echo
echo "Summary:"
echo "Open ports: ${open_ports[*]}"
echo "Total open ports: ${#open_ports[@]}"
```

### Exercise 3: Socket Programming Example
```python
#!/usr/bin/env python3
# TCP Server Example
import socket
import threading

def handle_client(client_socket, address):
    """Handle individual client connections"""
    print(f"Connection from {address}")
    
    try:
        while True:
            data = client_socket.recv(1024).decode('utf-8')
            if not data:
                break
            
            print(f"Received from {address}: {data}")
            
            # Echo the data back
            response = f"Echo: {data}"
            client_socket.send(response.encode('utf-8'))
    
    except Exception as e:
        print(f"Error handling client {address}: {e}")
    finally:
        client_socket.close()
        print(f"Connection with {address} closed")

def tcp_server(host='localhost', port=12345):
    """Simple TCP server"""
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    
    try:
        server_socket.bind((host, port))
        server_socket.listen(5)
        print(f"TCP Server listening on {host}:{port}")
        
        while True:
            client_socket, address = server_socket.accept()
            client_thread = threading.Thread(
                target=handle_client, 
                args=(client_socket, address)
            )
            client_thread.daemon = True
            client_thread.start()
    
    except KeyboardInterrupt:
        print("\nServer shutting down...")
    finally:
        server_socket.close()

if __name__ == "__main__":
    tcp_server()
```

```python
#!/usr/bin/env python3
# TCP Client Example
import socket

def tcp_client(host='localhost', port=12345):
    """Simple TCP client"""
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    
    try:
        client_socket.connect((host, port))
        print(f"Connected to {host}:{port}")
        
        while True:
            message = input("Enter message (or 'quit' to exit): ")
            if message.lower() == 'quit':
                break
            
            client_socket.send(message.encode('utf-8'))
            response = client_socket.recv(1024).decode('utf-8')
            print(f"Server response: {response}")
    
    except Exception as e:
        print(f"Error: {e}")
    finally:
        client_socket.close()

if __name__ == "__main__":
    tcp_client()
```

## Sample Exercises

1. **Protocol Selection:** When would you choose TCP over UDP and vice versa? Provide specific examples.

2. **Connection Troubleshooting:** A web application is experiencing slow response times. How would you troubleshoot at the transport layer?

3. **Port Management:** Design a port allocation strategy for a multi-service application server.

4. **Socket Programming:** Explain the difference between blocking and non-blocking sockets.

5. **Performance Optimization:** What transport layer optimizations can improve application performance?

## Solutions

1. **Protocol Selection:**
   - **Use TCP when:**
     - Data integrity is critical (web browsing, file transfer, email)
     - Ordered delivery is required (database transactions)
     - Connection state is needed (persistent sessions)
   - **Use UDP when:**
     - Speed is more important than reliability (gaming, streaming)
     - Simple request/response (DNS queries)
     - Broadcast/multicast needed (DHCP, network discovery)
     - Real-time applications (VoIP, video conferencing)

2. **Connection Troubleshooting Steps:**
   - Check connection states: `ss -t state established`
   - Monitor retransmissions: `ss -i | grep retrans`
   - Check TCP window sizes: `ss -i | grep cwnd`
   - Analyze connection timeouts: `ss -o state fin-wait-1`
   - Review system TCP settings: `/proc/sys/net/ipv4/tcp_*`
   - Use packet capture for detailed analysis

3. **Port Allocation Strategy:**
   ```
   Service Tier Allocation:
   - Web services: 8000-8099
   - API services: 8100-8199  
   - Database services: 8200-8299
   - Monitoring services: 8300-8399
   - Internal services: 8400-8499
   
   Benefits:
   - Organized and predictable
   - Easy firewall rule management
   - Clear service identification
   ```

4. **Blocking vs Non-blocking Sockets:**
   - **Blocking Sockets:**
     - Operations wait until completion
     - Simpler programming model
     - One thread per connection typically needed
   - **Non-blocking Sockets:**
     - Operations return immediately
     - More complex programming (select/poll/epoll)
     - Can handle many connections with fewer threads
     - Better scalability for high-concurrency applications

5. **Transport Layer Optimizations:**
   - **TCP Window Scaling:** Increase buffer sizes for high-bandwidth networks
   - **TCP Congestion Control:** Tune algorithms (cubic, bbr)
   - **Connection Pooling:** Reuse connections to reduce handshake overhead
   - **Keep-alive Settings:** Optimize for application patterns
   - **Nagle Algorithm:** Disable for low-latency applications
   - **Buffer Tuning:** Adjust send/receive buffer sizes

## Completion Checklist
- [ ] Understand TCP and UDP characteristics and use cases
- [ ] Know port number ranges and common service ports
- [ ] Can analyze TCP connection states and troubleshoot issues
- [ ] Understand socket programming concepts
- [ ] Familiar with transport layer performance optimization
- [ ] Can use tools like ss, netstat, and tcpdump for analysis

## Key Takeaways
- TCP provides reliable, ordered delivery with connection management
- UDP offers fast, lightweight communication for specific use cases
- Port numbers enable multiple applications to share network resources
- Socket programming requires understanding of blocking vs non-blocking operations
- Transport layer troubleshooting focuses on connection states and performance metrics
- Proper protocol selection is crucial for application performance and reliability

## Next Steps
Proceed to [Day 6: Layer 5 - Session Layer](../Day_06/notes_and_exercises.md) to learn about session management, authentication, and real-world session implementations.