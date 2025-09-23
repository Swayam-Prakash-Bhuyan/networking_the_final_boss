# Day 02: Layer 1 - Physical Layer (Cables, Media, Signals, Cloud Realities)

## Learning Objectives
By the end of Day 2, you will:
- Understand physical transmission media types and their characteristics
- Learn about signal transmission, encoding, and impairments
- Explore cloud infrastructure physical realities and virtualization
- Master physical layer troubleshooting techniques and tools

**Estimated Time:** 2-3 hours

## Notes

### Physical Layer Fundamentals
- **Purpose:** Responsible for the actual transmission of raw bits over a physical medium
- **Functions:** Electrical signaling, optical signaling, radio frequency transmission
- **Key Concepts:** Bandwidth, attenuation, noise, signal encoding
- **Standards:** IEEE 802.3 (Ethernet), IEEE 802.11 (Wi-Fi), ITU-T (telecommunications)

### Transmission Media Types

#### Guided Media (Wired)

**Twisted Pair Cables:**
- **UTP (Unshielded Twisted Pair):** Cat5e (1 Gbps), Cat6 (10 Gbps), Cat6a (10 Gbps extended), Cat7 (40 Gbps)
- **STP (Shielded Twisted Pair):** Better electromagnetic interference (EMI) protection
- **Applications:** Ethernet networks, telephone systems, building infrastructure
- **Distance Limitations:** 100 meters for Ethernet applications

**Coaxial Cables:**
- **Structure:** Central conductor, insulation layer, metallic shield, outer jacket
- **Types:** RG-6 (cable TV), RG-59 (CCTV), RG-11 (long distance)
- **Applications:** Cable internet, television distribution, legacy Ethernet (10BASE2)
- **Advantages:** Better shielding than twisted pair, longer distances

**Fiber Optic Cables:**
- **Single-mode:** 9-micron core, laser light source, long distances (up to 100km)
- **Multi-mode:** 50/62.5-micron core, LED light source, shorter distances (up to 2km)
- **Advantages:** Immune to EMI, high bandwidth, secure transmission
- **Applications:** Internet backbone, data centers, high-speed networks

#### Unguided Media (Wireless)

**Radio Waves:**
- **Frequency Range:** 3 KHz to 1 GHz
- **Characteristics:** Omnidirectional, can penetrate walls
- **Applications:** AM/FM radio, Wi-Fi (2.4 GHz), Bluetooth, cellular networks

**Microwaves:**
- **Frequency Range:** 1 GHz to 300 GHz
- **Characteristics:** Line-of-sight transmission, directional
- **Applications:** Satellite communication, point-to-point links, cellular backhaul

**Infrared:**
- **Characteristics:** Line-of-sight, short range, blocked by obstacles
- **Applications:** Remote controls, IrDA data transfer, some wireless LANs

### Signal Transmission Concepts

#### Digital vs Analog Signals
- **Digital Signals:** Discrete voltage levels representing binary data (0s and 1s)
- **Analog Signals:** Continuous waveforms that vary smoothly over time
- **Conversion:** ADC (Analog-to-Digital), DAC (Digital-to-Analog)

#### Signal Encoding Schemes
- **NRZ (Non-Return-to-Zero):** Simple but lacks clock recovery
- **Manchester Encoding:** Self-synchronizing, used in 10 Mbps Ethernet
- **Differential Manchester:** Better noise immunity, used in Token Ring
- **4B/5B:** Maps 4 data bits to 5 code bits, used in Fast Ethernet

#### Signal Impairments
- **Attenuation:** Signal strength decreases over distance
- **Distortion:** Signal shape changes due to medium characteristics
- **Noise:** Unwanted electrical interference (thermal, crosstalk, impulse)
- **Jitter:** Timing variations in digital signals

### Cloud Infrastructure Physical Realities

#### Data Center Infrastructure
- **Power Systems:** Redundant power supplies, UPS systems, generators
- **Cooling Systems:** HVAC, liquid cooling, hot/cold aisle containment
- **Cabling Infrastructure:** Structured cabling, fiber backbone, copper access
- **Network Equipment:** Top-of-rack switches, spine-leaf architecture

#### Virtualized Physical Layer
- **SR-IOV (Single Root I/O Virtualization):** Hardware-level network virtualization
- **DPDK (Data Plane Development Kit):** Bypass kernel for high-performance networking
- **Virtual NICs:** Software-defined network interfaces in VMs
- **Network Function Virtualization (NFV):** Software-based network functions

#### Cloud Provider Infrastructure
- **AWS:** Custom Nitro system, 100 Gbps networking, global fiber network
- **Azure:** SmartNIC technology, FPGA acceleration, ExpressRoute connections
- **GCP:** Jupiter network fabric, custom ASICs, dedicated interconnects

## Hands-On Labs

### Lab 1: Cable Testing and Analysis
```bash
# Install network testing tools
sudo apt update
sudo apt install ethtool net-tools

# Check physical interface details
sudo ethtool eth0 2>/dev/null || echo "ethtool not supported on this interface"

# Check interface statistics
ip -s link show eth0

# Test cable integrity (if supported)
sudo ethtool -t eth0 2>/dev/null || echo "Self-test not supported"

# Monitor interface statistics in real-time
watch -n 1 'cat /proc/net/dev | grep eth0'

# Check for interface errors
cat /sys/class/net/eth0/statistics/rx_errors 2>/dev/null || echo "0"
cat /sys/class/net/eth0/statistics/tx_errors 2>/dev/null || echo "0"
```

### Lab 2: Signal Quality Analysis
```bash
# Check for interface errors and statistics
ip -s link show eth0

# Monitor error counters
netstat -i | grep eth0

# Check for collisions (indicates half-duplex issues)
cat /sys/class/net/eth0/statistics/collisions 2>/dev/null || echo "No collision data"

# View detailed interface statistics
sudo ethtool -S eth0 2>/dev/null | head -20

# Check link speed and duplex
sudo ethtool eth0 2>/dev/null | grep -E "Speed|Duplex|Link"

# Monitor bandwidth utilization
sar -n DEV 1 5 2>/dev/null || echo "sar not available, install sysstat package"
```

### Lab 3: Wireless Physical Layer Analysis
```bash
# Install wireless tools
sudo apt install wireless-tools iw 2>/dev/null || echo "Wireless tools installation attempted"

# Check wireless interface capabilities
iw list 2>/dev/null || echo "No wireless interfaces found"

# Scan for wireless networks with signal strength
sudo iwlist scan 2>/dev/null | grep -E "ESSID|Signal level" | head -10

# Monitor wireless signal quality
watch -n 2 'cat /proc/net/wireless 2>/dev/null || echo "No wireless interfaces"'

# Check wireless interface details
iw dev 2>/dev/null || echo "No wireless devices found"

# Show wireless configuration
iwconfig 2>/dev/null || echo "iwconfig not available"
```

### Physical Layer Standards and Specifications

| Standard | Medium | Speed | Distance | Application |
|----------|--------|-------|----------|-------------|
| 10BASE-T | Cat3 UTP | 10 Mbps | 100m | Legacy Ethernet |
| 100BASE-TX | Cat5 UTP | 100 Mbps | 100m | Fast Ethernet |
| 1000BASE-T | Cat5e UTP | 1 Gbps | 100m | Gigabit Ethernet |
| 10GBASE-T | Cat6a UTP | 10 Gbps | 100m | 10 Gigabit Ethernet |
| 1000BASE-SX | MM Fiber | 1 Gbps | 550m | Gigabit Ethernet |
| 10GBASE-SR | MM Fiber | 10 Gbps | 300m | 10 Gigabit Ethernet |
| 10GBASE-LR | SM Fiber | 10 Gbps | 10km | Long Range 10GbE |

## Practical Exercises

### Exercise 1: Cable Performance Testing Script
```bash
#!/bin/bash
# Network performance tester for different scenarios

TARGET_SERVER=${1:-"8.8.8.8"}
TEST_DURATION=${2:-10}

echo "Network Performance Testing"
echo "========================="
echo "Target: $TARGET_SERVER"
echo "Duration: ${TEST_DURATION}s"
echo

# Test 1: Latency measurement
echo "1. Latency Test:"
ping -c 10 $TARGET_SERVER | tail -1
echo

# Test 2: Bandwidth test (if iperf3 available)
echo "2. Bandwidth Test:"
if command -v iperf3 &> /dev/null; then
    echo "Install iperf3 server on target for bandwidth testing"
    # iperf3 -c $TARGET_SERVER -t $TEST_DURATION
else
    echo "iperf3 not installed. Install with: sudo apt install iperf3"
fi
echo

# Test 3: Packet loss test
echo "3. Packet Loss Test:"
ping -c 100 -i 0.1 $TARGET_SERVER | grep "packet loss"
echo

# Test 4: Interface statistics before and after
echo "4. Interface Statistics:"
echo "Before test:"
cat /proc/net/dev | grep eth0
echo "Run network activity..."
sleep 2
echo "After test:"
cat /proc/net/dev | grep eth0
```

### Exercise 2: Physical Layer Troubleshooting Guide
```bash
#!/bin/bash
# Physical Layer Troubleshooting Script

echo "Physical Layer Troubleshooting Guide"
echo "===================================="

# Check 1: Interface Status
echo "1. Interface Status Check:"
for interface in $(ls /sys/class/net/ | grep -v lo); do
    status=$(cat /sys/class/net/$interface/operstate 2>/dev/null)
    carrier=$(cat /sys/class/net/$interface/carrier 2>/dev/null)
    echo "  $interface: Status=$status, Carrier=$carrier"
done
echo

# Check 2: Cable Connection
echo "2. Cable Connection Check:"
for interface in $(ls /sys/class/net/ | grep -E "eth|ens|enp"); do
    if [ -f "/sys/class/net/$interface/carrier" ]; then
        carrier=$(cat /sys/class/net/$interface/carrier 2>/dev/null)
        if [ "$carrier" = "1" ]; then
            echo "  ✓ $interface: Cable connected"
        else
            echo "  ✗ $interface: No cable or bad connection"
        fi
    fi
done
echo

# Check 3: Interface Errors
echo "3. Interface Error Check:"
for interface in $(ls /sys/class/net/ | grep -E "eth|ens|enp"); do
    rx_errors=$(cat /sys/class/net/$interface/statistics/rx_errors 2>/dev/null || echo "0")
    tx_errors=$(cat /sys/class/net/$interface/statistics/tx_errors 2>/dev/null || echo "0")
    echo "  $interface: RX errors=$rx_errors, TX errors=$tx_errors"
done
echo

# Check 4: Speed and Duplex
echo "4. Speed and Duplex Check:"
for interface in $(ls /sys/class/net/ | grep -E "eth|ens|enp"); do
    speed=$(cat /sys/class/net/$interface/speed 2>/dev/null || echo "unknown")
    duplex=$(cat /sys/class/net/$interface/duplex 2>/dev/null || echo "unknown")
    echo "  $interface: Speed=${speed}Mbps, Duplex=$duplex"
done
```

### Exercise 3: Cloud Network Performance Tester
```python
#!/usr/bin/env python3
import subprocess
import time
import json
import statistics

def ping_test(host, count=10):
    """Perform ping test and return statistics"""
    try:
        result = subprocess.run(
            ['ping', '-c', str(count), host],
            capture_output=True, text=True, timeout=30
        )
        
        # Parse ping results
        lines = result.stdout.split('\n')
        times = []
        
        for line in lines:
            if 'time=' in line:
                time_str = line.split('time=')[1].split()[0]
                times.append(float(time_str))
        
        if times:
            return {
                'min': min(times),
                'max': max(times),
                'avg': statistics.mean(times),
                'count': len(times)
            }
    except Exception as e:
        return {'error': str(e)}
    
    return {'error': 'No data'}

def bandwidth_estimate(host, port=80):
    """Estimate bandwidth using curl"""
    try:
        start_time = time.time()
        result = subprocess.run(
            ['curl', '-s', '-o', '/dev/null', '-w', '%{speed_download}', f'http://{host}'],
            capture_output=True, text=True, timeout=10
        )
        end_time = time.time()
        
        if result.returncode == 0 and result.stdout:
            speed_bps = float(result.stdout)
            speed_mbps = (speed_bps * 8) / (1024 * 1024)  # Convert to Mbps
            return {
                'speed_mbps': round(speed_mbps, 2),
                'duration': round(end_time - start_time, 2)
            }
    except Exception as e:
        return {'error': str(e)}
    
    return {'error': 'Test failed'}

def network_performance_test():
    """Run comprehensive network performance test"""
    test_hosts = ['8.8.8.8', 'google.com', '1.1.1.1']
    results = {}
    
    print("Network Performance Test")
    print("=" * 40)
    
    for host in test_hosts:
        print(f"\nTesting {host}...")
        
        # Ping test
        ping_result = ping_test(host)
        
        # Bandwidth estimate (only for domain names)
        bandwidth_result = {}
        if '.' in host and not host.replace('.', '').isdigit():
            bandwidth_result = bandwidth_estimate(host)
        
        results[host] = {
            'ping': ping_result,
            'bandwidth': bandwidth_result
        }
        
        # Display results
        if 'error' not in ping_result:
            print(f"  Ping: {ping_result['avg']:.2f}ms avg ({ping_result['count']} packets)")
        else:
            print(f"  Ping: {ping_result['error']}")
        
        if bandwidth_result and 'error' not in bandwidth_result:
            print(f"  Bandwidth estimate: {bandwidth_result['speed_mbps']} Mbps")
    
    return results

if __name__ == "__main__":
    results = network_performance_test()
    
    # Save results to file
    with open('network_performance.json', 'w') as f:
        json.dump(results, f, indent=2)
    
    print("\nResults saved to network_performance.json")
```

## Sample Exercises

1. **Cable Selection:** You need to connect two buildings 500 meters apart with a 10 Gbps connection. What type of cable would you choose and why?

2. **Signal Analysis:** Explain why Manchester encoding is preferred over NRZ for Ethernet transmission.

3. **Troubleshooting Scenario:** A network connection is experiencing intermittent connectivity issues. List the physical layer checks you would perform.

4. **Cloud Infrastructure:** Compare the physical networking differences between on-premises data centers and cloud environments.

5. **Performance Calculation:** Calculate the maximum theoretical throughput for a Cat6 cable over 90 meters vs 100 meters.

## Solutions

1. **Cable Selection (500m, 10 Gbps):**
   - **Answer:** Single-mode fiber optic cable with 10GBASE-LR transceivers
   - **Reasoning:** 
     - Copper cables (Cat6a) limited to 100m for 10 Gbps
     - Multi-mode fiber limited to ~300m for 10GBASE-SR
     - Single-mode fiber can support 10 Gbps up to 10km
     - Cost-effective for this distance requirement

2. **Manchester Encoding Benefits:**
   - **Self-synchronizing:** Clock information embedded in signal
   - **DC balance:** Equal high and low periods prevent DC drift
   - **Error detection:** Invalid transitions indicate transmission errors
   - **Noise immunity:** Differential encoding reduces noise impact
   - **Simple implementation:** Easy to encode/decode in hardware

3. **Physical Layer Troubleshooting Checklist:**
   - **Visual inspection:** Check for damaged cables, loose connections
   - **Link lights:** Verify interface LEDs show proper status
   - **Cable testing:** Use cable tester to check continuity and wiring
   - **Signal quality:** Check for excessive errors, CRC failures
   - **Environmental factors:** Temperature, humidity, electromagnetic interference
   - **Power levels:** Verify adequate power for PoE devices
   - **Distance limitations:** Ensure cables within specification limits

4. **On-Premises vs Cloud Physical Networking:**
   - **On-Premises:**
     - Direct control over physical infrastructure
     - Custom cabling and equipment selection
     - Physical security under your control
     - Capital expenditure for equipment
   - **Cloud:**
     - Abstracted physical layer (virtualized interfaces)
     - Shared infrastructure with isolation
     - Provider manages physical maintenance
     - Operational expenditure model
     - Global reach through provider networks

5. **Cat6 Performance Calculation:**
   - **90 meters:** Full 10 Gbps capability (within specification)
   - **100 meters:** Still 10 Gbps but at specification limit
   - **Beyond 100m:** Performance degrades, may drop to 1 Gbps
   - **Factors:** Signal attenuation increases with distance, crosstalk effects
   - **Real-world:** Always test actual performance with network tools

## Completion Checklist
- [ ] Understand different types of transmission media and their characteristics
- [ ] Know the differences between guided and unguided media
- [ ] Understand signal encoding and transmission concepts
- [ ] Can identify appropriate cables for different scenarios
- [ ] Familiar with physical layer troubleshooting techniques
- [ ] Understand cloud infrastructure physical realities

## Key Takeaways
- Physical layer forms the foundation of all network communication
- Cable selection depends on distance, speed, and environmental requirements
- Signal quality directly impacts network performance and reliability
- Cloud environments abstract but don't eliminate physical layer considerations
- Proper physical layer design is crucial for network scalability and performance

## Next Steps
Proceed to [Day 3: Layer 2 - Data Link](../Day_03/notes_and_exercises.md) to learn about Ethernet, VLANs, MAC addressing, switching, and ARP protocols.