# Networking Fundamentals

## Table of Contents
1. [Network Interface Management](#network-interface-management)
2. [IP Addressing and Subnetting](#ip-addressing-and-subnetting)
3. [Network Configuration](#network-configuration)
4. [Network Tools and Commands](#network-tools-and-commands)
5. [Remote Access Tools](#remote-access-tools)
6. [Network Troubleshooting](#network-troubleshooting)
7. [Practical Scenarios](#practical-scenarios)

---

## Network Interface Management

### Understanding Network Interfaces

Linux network interfaces are the connection points between your system and the network. Common types include:

- **eth0, eth1**: Ethernet interfaces
- **wlan0, wifi0**: Wireless interfaces  
- **lo**: Loopback interface (127.0.0.1)
- **docker0, br0**: Bridge interfaces
- **tun0, tap0**: VPN tunnel interfaces

### Viewing Network Interfaces

```bash
# Modern command (preferred)
ip addr show
ip a  # Short form

# Legacy command (deprecated but still common)
ifconfig

# Show only active interfaces
ip link show up

# Show specific interface
ip addr show eth0

# Show interface statistics
ip -s link show eth0
```

### Interface Status and Control

```bash
# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Legacy method
sudo ifconfig eth0 up
sudo ifconfig eth0 down

# Check interface status
ip link show eth0 | grep -o "state [A-Z]*"
```

---

## IP Addressing and Subnetting

### IPv4 Address Classes

| Class | Range | Default Subnet Mask | CIDR | Usage |
|-------|-------|-------------------|------|--------|
| A | 1.0.0.0 - 126.255.255.255 | 255.0.0.0 | /8 | Large networks |
| B | 128.0.0.0 - 191.255.255.255 | 255.255.0.0 | /16 | Medium networks |
| C | 192.0.0.0 - 223.255.255.255 | 255.255.255.0 | /24 | Small networks |

### Private IP Ranges (RFC 1918)
- **Class A**: 10.0.0.0/8 (10.0.0.0 - 10.255.255.255)
- **Class B**: 172.16.0.0/12 (172.16.0.0 - 172.31.255.255)
- **Class C**: 192.168.0.0/16 (192.168.0.0 - 192.168.255.255)

### CIDR Notation Examples

```bash
# Common subnet masks and their CIDR equivalents
255.255.255.0   = /24  (256 addresses, 254 hosts)
255.255.255.128 = /25  (128 addresses, 126 hosts)
255.255.255.192 = /26  (64 addresses, 62 hosts)
255.255.255.224 = /27  (32 addresses, 30 hosts)
255.255.255.240 = /28  (16 addresses, 14 hosts)
255.255.255.248 = /29  (8 addresses, 6 hosts)
255.255.255.252 = /30  (4 addresses, 2 hosts)
```

### Subnetting Calculator Script

```bash
#!/bin/bash
# subnet_calculator.sh

calculate_subnet() {
    local ip_cidr="$1"
    local ip="${ip_cidr%/*}"
    local cidr="${ip_cidr#*/}"
    
    if [ -z "$cidr" ]; then
        echo "Usage: provide IP/CIDR (e.g., 192.168.1.0/24)"
        return 1
    fi
    
    # Convert CIDR to subnet mask
    local mask=""
    local full_octets=$((cidr / 8))
    local partial_octet=$((cidr % 8))
    
    for ((i=0; i<4; i++)); do
        if [ $i -lt $full_octets ]; then
            mask="$mask.255"
        elif [ $i -eq $full_octets ] && [ $partial_octet -gt 0 ]; then
            local partial_mask=$((256 - 2**(8-partial_octet)))
            mask="$mask.$partial_mask"
        else
            mask="$mask.0"
        fi
    done
    mask=${mask#.}
    
    # Calculate network details
    local host_bits=$((32 - cidr))
    local total_addresses=$((2**host_bits))
    local usable_hosts=$((total_addresses - 2))
    
    echo "Network: $ip/$cidr"
    echo "Subnet Mask: $mask"
    echo "Total Addresses: $total_addresses"
    echo "Usable Host Addresses: $usable_hosts"
    echo "Host Bits: $host_bits"
    echo "Network Bits: $cidr"
}

# Example usage
calculate_subnet "192.168.1.0/24"
```

---

## Network Configuration

### Temporary IP Configuration

```bash
# Add IP address to interface
sudo ip addr add 192.168.1.100/24 dev eth0

# Remove IP address
sudo ip addr del 192.168.1.100/24 dev eth0

# Legacy method
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0

# Add multiple IPs to same interface
sudo ip addr add 192.168.1.101/24 dev eth0
sudo ip addr add 192.168.1.102/24 dev eth0
```

### Default Gateway Configuration

```bash
# View current routing table
ip route show
route -n  # Legacy command

# Add default gateway
sudo ip route add default via 192.168.1.1

# Add specific route
sudo ip route add 10.0.0.0/8 via 192.168.1.2

# Delete route
sudo ip route del default via 192.168.1.1

# Legacy method
sudo route add default gw 192.168.1.1
sudo route del default gw 192.168.1.1
```

### DNS Configuration

#### /etc/resolv.conf
```bash
# DNS configuration file
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1
search company.local
options timeout:1 attempts:2
```

#### systemd-resolved (Ubuntu 18.04+)
```bash
# Check DNS status
systemd-resolve --status

# Query specific DNS
systemd-resolve google.com 8.8.8.8

# Flush DNS cache
sudo systemd-resolve --flush-caches
```

### Persistent Network Configuration

#### Ubuntu/Debian (Netplan)
```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - company.local
```

Apply configuration:
```bash
sudo netplan apply
sudo netplan try  # Test configuration for 120 seconds
```

#### CentOS/RHEL (NetworkManager)
```bash
# /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=eth0
UUID=12345678-1234-1234-1234-123456789012
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```

Restart networking:
```bash
sudo systemctl restart NetworkManager
sudo nmcli connection reload
```

---

## Network Tools and Commands

### Connectivity Testing

```bash
# Ping - Test connectivity
ping -c 4 google.com
ping6 -c 4 google.com  # IPv6

# Ping with specific interval and size
ping -c 10 -i 0.5 -s 1000 192.168.1.1

# Traceroute - Show network path
traceroute google.com
tracepath google.com  # Alternative

# MTU Path Discovery
tracepath -n google.com
```

### Network Scanning

```bash
# Nmap - Network exploration and security scanning
nmap 192.168.1.1-254          # Scan IP range
nmap -sP 192.168.1.0/24       # Ping scan
nmap -sS -O 192.168.1.100     # TCP SYN scan with OS detection
nmap -sU -p 53,123,161 192.168.1.1  # UDP scan specific ports

# Port scanning with netcat
nc -zv google.com 80          # Check if port 80 is open
nc -zv -u 192.168.1.1 53     # UDP port check

# ARP table
arp -a                        # View ARP table
ip neigh show                 # Modern alternative
```

### Network Statistics

```bash
# Netstat - Network connections and statistics
netstat -tuln                 # Show listening TCP/UDP ports
netstat -rn                   # Show routing table
netstat -i                    # Interface statistics
netstat -s                    # Protocol statistics

# ss - Modern replacement for netstat
ss -tuln                      # Listening ports
ss -t state established       # Established TCP connections
ss -p                         # Show processes

# Network interface statistics
cat /proc/net/dev
iostat -n                     # Network I/O statistics
```

### Bandwidth and Performance Testing

```bash
# iperf3 - Network performance testing
# On server:
iperf3 -s

# On client:
iperf3 -c server_ip

# UDP test
iperf3 -c server_ip -u -b 100M

# Speedtest using curl
curl -s https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py | python3

# Monitor bandwidth usage
sudo iftop                    # Real-time bandwidth usage
sudo nethogs                  # Per-process network usage
nload                         # Simple bandwidth monitor
```

---

## Remote Access Tools

### SSH (Secure Shell)

#### Basic SSH Usage
```bash
# Connect to remote server
ssh username@hostname
ssh -p 2222 username@hostname    # Custom port

# SSH with key authentication
ssh -i ~/.ssh/private_key username@hostname

# X11 forwarding
ssh -X username@hostname

# Port forwarding (local)
ssh -L 8080:localhost:80 username@hostname

# Port forwarding (remote)
ssh -R 9090:localhost:22 username@hostname

# SSH tunnel (SOCKS proxy)
ssh -D 1080 username@hostname
```

#### SSH Configuration
```bash
# Client configuration: ~/.ssh/config
Host webserver
    HostName 192.168.1.100
    User admin
    Port 2222
    IdentityFile ~/.ssh/webserver_key
    
Host *.company.com
    User john
    IdentityFile ~/.ssh/company_key
    ProxyJump jumphost.company.com
```

#### SSH Server Configuration
```bash
# /etc/ssh/sshd_config (common security settings)
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin john
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2

# Restart SSH service
sudo systemctl restart sshd
```

### SCP (Secure Copy)

```bash
# Copy file to remote server
scp file.txt username@hostname:/path/to/destination/

# Copy file from remote server
scp username@hostname:/path/to/file.txt ./

# Copy directory recursively
scp -r /local/directory/ username@hostname:/remote/path/

# Copy with specific SSH key
scp -i ~/.ssh/private_key file.txt username@hostname:~/

# Copy with compression
scp -C large_file.zip username@hostname:~/

# Copy through jump host
scp -o ProxyJump=jumphost file.txt user@finalhost:~/
```

### RSYNC (Remote Sync)

```bash
# Basic rsync usage
rsync -avz /local/path/ username@hostname:/remote/path/

# Common options:
# -a: archive mode (preserves permissions, timestamps, etc.)
# -v: verbose
# -z: compression
# -h: human-readable
# -P: progress and partial transfer
# --delete: delete files not in source

# Sync with progress and exclude files
rsync -avzh --progress --exclude='*.log' --exclude='.git/' \
    /local/project/ username@hostname:/opt/project/

# Dry run (test without making changes)
rsync -avzn /source/ /destination/

# Sync over SSH with specific port
rsync -avz -e "ssh -p 2222" /local/ user@host:/remote/
```

### FTP/SFTP

#### SFTP Usage
```bash
# Connect to SFTP server
sftp username@hostname

# SFTP commands:
get remote_file.txt           # Download file
put local_file.txt            # Upload file
ls                           # List remote directory
lls                          # List local directory
cd /remote/path              # Change remote directory
lcd /local/path              # Change local directory
mkdir remote_dir             # Create remote directory
rm remote_file.txt           # Delete remote file
exit                         # Close connection
```

#### FTP Client (Legacy)
```bash
# Connect to FTP server
ftp hostname

# FTP commands (similar to SFTP):
binary                       # Set binary transfer mode
passive                      # Enable passive mode
get file.txt                 # Download file
put file.txt                 # Upload file
mget *.txt                   # Download multiple files
quit                         # Exit
```

---

## Network Troubleshooting

### Common Network Issues and Solutions

#### 1. No Network Connectivity

```bash
# Check physical connection
sudo ethtool eth0  # Check link status

# Check interface status
ip link show eth0

# Check IP configuration
ip addr show eth0

# Check default gateway
ip route show default

# Test DNS resolution
nslookup google.com
dig google.com
```

#### 2. Slow Network Performance

```bash
# Check interface errors
ip -s link show eth0

# Monitor network usage
sudo iftop -i eth0

# Check for network congestion
ping -f -c 100 gateway_ip    # Flood ping

# Test bandwidth
iperf3 -c remote_server

# Check MTU issues
ping -M do -s 1472 destination  # Test if MTU is causing fragmentation
```

#### 3. DNS Resolution Problems

```bash
# Check DNS servers
cat /etc/resolv.conf

# Test DNS resolution
nslookup google.com 8.8.8.8
dig @8.8.8.8 google.com

# Flush DNS cache
sudo systemd-resolve --flush-caches  # Ubuntu 18.04+
sudo service nscd restart            # Older systems

# Check DNS configuration files
sudo systemd-resolve --status
```

### Network Diagnostic Script

```bash
#!/bin/bash
# network_diagnostic.sh - Comprehensive network diagnostic

echo "=== Network Diagnostic Report ==="
echo "Date: $(date)"
echo

# 1. Interface Status
echo "1. Network Interfaces:"
ip addr show | grep -E "^[0-9]+:|inet "
echo

# 2. Routing Information
echo "2. Routing Table:"
ip route show
echo

# 3. DNS Configuration
echo "3. DNS Configuration:"
cat /etc/resolv.conf | grep -v "^#" | grep -v "^$"
echo

# 4. Connectivity Tests
echo "4. Connectivity Tests:"
targets=("8.8.8.8" "google.com" "1.1.1.1")
for target in "${targets[@]}"; do
    if ping -c 1 -W 3 "$target" >/dev/null 2>&1; then
        echo "✓ $target - Reachable"
    else
        echo "✗ $target - Not reachable"
    fi
done
echo

# 5. Listening Services
echo "5. Listening Network Services:"
ss -tuln | head -20
echo

# 6. ARP Table
echo "6. ARP Table:"
ip neigh show | head -10
echo

# 7. Network Statistics
echo "7. Network Interface Statistics:"
cat /proc/net/dev | grep -E "(eth|wlan|en)" | head -5
```

---

## Practical Scenarios

### Scenario 1: Setting up Static IP Configuration

```bash
#!/bin/bash
# setup_static_ip.sh

INTERFACE="eth0"
IP_ADDRESS="192.168.1.100"
SUBNET_MASK="24"
GATEWAY="192.168.1.1"
DNS_SERVERS="8.8.8.8,8.8.4.4"

echo "Configuring static IP for $INTERFACE"

# Create netplan configuration (Ubuntu)
cat > /etc/netplan/99-static-config.yaml << EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    $INTERFACE:
      dhcp4: false
      addresses:
        - $IP_ADDRESS/$SUBNET_MASK
      gateway4: $GATEWAY
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
        search:
          - local
EOF

# Test and apply configuration
echo "Testing configuration..."
netplan try
```

### Scenario 2: Network Monitoring Setup

```bash
#!/bin/bash
# network_monitor.sh - Continuous network monitoring

LOG_FILE="/var/log/network_monitor.log"
INTERFACE="eth0"
CHECK_INTERVAL=60

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

while true; do
    # Check interface status
    if ip link show "$INTERFACE" | grep -q "state UP"; then
        LINK_STATUS="UP"
    else
        LINK_STATUS="DOWN"
        log_message "WARNING: Interface $INTERFACE is DOWN"
    fi
    
    # Check connectivity
    if ping -c 1 -W 3 8.8.8.8 >/dev/null 2>&1; then
        CONNECTIVITY="OK"
    else
        CONNECTIVITY="FAILED"
        log_message "ERROR: Internet connectivity failed"
    fi
    
    # Get interface statistics
    RX_BYTES=$(cat /sys/class/net/$INTERFACE/statistics/rx_bytes)
    TX_BYTES=$(cat /sys/class/net/$INTERFACE/statistics/tx_bytes)
    RX_ERRORS=$(cat /sys/class/net/$INTERFACE/statistics/rx_errors)
    TX_ERRORS=$(cat /sys/class/net/$INTERFACE/statistics/tx_errors)
    
    # Log status
    log_message "Status: Link=$LINK_STATUS, Connectivity=$CONNECTIVITY, RX=$RX_BYTES, TX=$TX_BYTES, RX_Errors=$RX_ERRORS, TX_Errors=$TX_ERRORS"
    
    sleep "$CHECK_INTERVAL"
done
```

### Scenario 3: Automated Network Setup for Lab Environment

```bash
#!/bin/bash
# lab_network_setup.sh

# Define network configuration
LAB_NETWORK="10.0.100.0/24"
GATEWAY="10.0.100.1"
DNS_SERVER="10.0.100.1"

setup_lab_interface() {
    local hostname="$1"
    local ip_suffix="$2"
    
    # Calculate IP address
    local ip_address="10.0.100.$ip_suffix"
    
    echo "Setting up $hostname with IP $ip_address"
    
    # Create temporary interface configuration
    ip addr add "$ip_address/24" dev eth1
    ip link set eth1 up
    
    # Add route if needed
    ip route add "$LAB_NETWORK" dev eth1
    
    # Update hostname
    hostnamectl set-hostname "$hostname"
    
    # Add to /etc/hosts
    echo "$ip_address $hostname" >> /etc/hosts
}

# Get machine identifier and setup accordingly
MACHINE_ID=$(hostname | grep -o '[0-9]\+$')
if [ -n "$MACHINE_ID" ]; then
    setup_lab_interface "lab-$(printf "%02d" $MACHINE_ID)" "$((100 + MACHINE_ID))"
else
    echo "Unable to determine machine ID from hostname"
    exit 1
fi
```

---

## Lab Exercises

### Exercise 1: Network Interface Management
1. List all network interfaces and their configurations
2. Configure a static IP address temporarily
3. Test connectivity to local gateway and remote host
4. Document the network configuration

### Exercise 2: Subnetting Practice
1. Given network 192.168.10.0/24, create 4 subnets
2. Calculate network addresses, broadcast addresses, and usable host ranges
3. Configure interfaces with addresses from different subnets
4. Test inter-subnet communication

### Exercise 3: Remote Access Setup
1. Configure SSH server with key-based authentication
2. Set up SCP/SFTP for file transfers
3. Create SSH tunnel for secure communication
4. Test remote access from different network segments

### Exercise 4: Network Troubleshooting
1. Simulate network connectivity issues
2. Use diagnostic tools to identify problems
3. Implement solutions and verify fixes
4. Create troubleshooting documentation

---

## Quick Reference Commands

```bash
# Interface Management
ip addr show                     # Show all interfaces
ip link set eth0 up             # Bring interface up
ip addr add 192.168.1.100/24 dev eth0  # Add IP address

# Routing
ip route show                    # Show routing table
ip route add default via 192.168.1.1   # Add default gateway

# Connectivity Testing
ping -c 4 google.com            # Test connectivity
traceroute google.com           # Show network path
nmap -sP 192.168.1.0/24        # Network scan

# Remote Access
ssh user@hostname               # SSH connection
scp file.txt user@host:~/       # Secure copy
rsync -avz /local/ user@host:/remote/  # Sync files

# Network Statistics
ss -tuln                        # Show listening ports
netstat -rn                     # Show routing table
iftop                           # Monitor bandwidth usage
```
