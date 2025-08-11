# Security and Firewall Management

## Table of Contents
1. [Linux Security Fundamentals](#linux-security-fundamentals)
2. [Firewall Concepts](#firewall-concepts)
3. [iptables Configuration](#iptables-configuration)
4. [UFW (Uncomplicated Firewall)](#ufw-uncomplicated-firewall)
5. [firewalld Management](#firewalld-management)
6. [Security Hardening](#security-hardening)
7. [Intrusion Detection](#intrusion-detection)
8. [Practical Scenarios](#practical-scenarios)

---

## Linux Security Fundamentals

### Security Principles

1. **Principle of Least Privilege**: Grant minimum necessary permissions
2. **Defense in Depth**: Multiple layers of security
3. **Fail Secure**: Default to deny when in doubt
4. **Keep it Simple**: Complex systems are harder to secure
5. **Regular Updates**: Keep systems patched and current

### Linux Security Model

```bash
# File permissions (owner, group, others)
ls -l file.txt
# -rw-r--r-- 1 user group 1234 date file.txt
#  ^^^ ^^^ ^^^
#  |   |   +-- Others permissions
#  |   +------ Group permissions  
#  +---------- Owner permissions

# Special permissions
# SUID (4): Execute as file owner
# SGID (2): Execute as group owner or inherit group
# Sticky bit (1): Only owner can delete files in directory

# Set special permissions
chmod 4755 /usr/bin/passwd    # SUID
chmod 2755 /shared/directory  # SGID
chmod 1777 /tmp               # Sticky bit
```

### Access Control Lists (ACL)

```bash
# Install ACL tools
sudo apt install acl          # Ubuntu/Debian
sudo yum install acl          # CentOS/RHEL

# View ACLs
getfacl filename

# Set ACLs
setfacl -m u:john:rw filename           # User permissions
setfacl -m g:developers:r filename      # Group permissions
setfacl -m o::--- filename              # Others permissions

# Remove ACLs
setfacl -x u:john filename              # Remove user ACL
setfacl -b filename                     # Remove all ACLs

# Default ACLs for directories
setfacl -d -m u:john:rw /shared/dir
```

---

## Firewall Concepts

### Network Security Zones

1. **Public Zone**: Untrusted networks (internet)
2. **DMZ**: Semi-trusted zone for public services
3. **Internal Zone**: Trusted internal network
4. **Management Zone**: Administrative access only

### Firewall Types

- **Packet Filter**: Examines individual packets
- **Stateful Inspection**: Tracks connection state
- **Application Layer**: Inspects application data
- **Proxy Firewall**: Acts as intermediary

### Common Ports and Services

| Port | Service | Protocol | Description |
|------|---------|----------|-------------|
| 22 | SSH | TCP | Secure Shell |
| 23 | Telnet | TCP | Unencrypted remote access |
| 25 | SMTP | TCP | Email transmission |
| 53 | DNS | TCP/UDP | Domain Name System |
| 80 | HTTP | TCP | Web traffic |
| 110 | POP3 | TCP | Email retrieval |
| 143 | IMAP | TCP | Email access |
| 443 | HTTPS | TCP | Secure web traffic |
| 993 | IMAPS | TCP | Secure IMAP |
| 995 | POP3S | TCP | Secure POP3 |

---

## iptables Configuration

### iptables Basics

iptables works with tables containing chains of rules:

- **Tables**: filter, nat, mangle, raw
- **Chains**: INPUT, OUTPUT, FORWARD, PREROUTING, POSTROUTING
- **Targets**: ACCEPT, DROP, REJECT, LOG

### Viewing Current Rules

```bash
# List all rules
sudo iptables -L -n -v

# List specific table
sudo iptables -t nat -L -n -v

# List with line numbers
sudo iptables -L -n --line-numbers

# Show exact command syntax
sudo iptables -S
```

### Basic iptables Rules

```bash
# Allow loopback traffic
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Allow established and related connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (port 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow ping (ICMP)
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Drop all other INPUT traffic
sudo iptables -A INPUT -j DROP
```

### Advanced iptables Rules

```bash
# Allow SSH from specific network only
sudo iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 22 -j ACCEPT

# Rate limiting for SSH (prevent brute force)
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
    -m recent --set
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
    -m recent --update --seconds 60 --hitcount 4 -j DROP

# Log dropped packets
sudo iptables -A INPUT -m limit --limit 5/min -j LOG \
    --log-prefix "iptables denied: " --log-level 7

# Port forwarding (NAT)
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 \
    -j REDIRECT --to-port 80

# Masquerading (for router/gateway)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### iptables Management

```bash
# Save rules (Ubuntu/Debian)
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Restore rules
sudo iptables-restore < /etc/iptables/rules.v4

# Make persistent (Ubuntu/Debian)
sudo apt install iptables-persistent
sudo dpkg-reconfigure iptables-persistent

# CentOS/RHEL
sudo service iptables save
sudo systemctl enable iptables
```

### iptables Script Example

```bash
#!/bin/bash
# secure_firewall.sh - Basic secure firewall setup

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH with rate limiting
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
    -m recent --set --name ssh
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
    -m recent --update --seconds 60 --hitcount 4 --name ssh -j DROP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Log dropped packets
iptables -A INPUT -m limit --limit 5/min -j LOG \
    --log-prefix "IPTables-Dropped: "

# Save rules
iptables-save > /etc/iptables/rules.v4

echo "Firewall configured successfully"
```

---

## UFW (Uncomplicated Firewall)

UFW is a user-friendly frontend for iptables, commonly used on Ubuntu.

### Basic UFW Commands

```bash
# Enable/disable UFW
sudo ufw enable
sudo ufw disable

# Check status
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### UFW Rules

```bash
# Allow/deny by port
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw allow 53/udp
sudo ufw deny 23

# Allow/deny by service name
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Allow from specific IP
sudo ufw allow from 192.168.1.100
sudo ufw allow from 192.168.1.0/24

# Allow specific IP to specific port
sudo ufw allow from 192.168.1.100 to any port 22

# Allow specific interface
sudo ufw allow in on eth0 to any port 80
```

### Advanced UFW Configuration

```bash
# Rate limiting
sudo ufw limit ssh
sudo ufw limit 22/tcp

# Application profiles
sudo ufw app list
sudo ufw allow 'Apache Full'
sudo ufw allow 'OpenSSH'

# Custom application profile
cat > /etc/ufw/applications.d/myapp << EOF
[MyApp]
title=My Application
description=Custom application
ports=3000,3001/tcp
EOF

sudo ufw allow MyApp
```

### UFW Management

```bash
# Delete rules
sudo ufw delete 2              # Delete rule number 2
sudo ufw delete allow 80       # Delete specific rule

# Reset all rules
sudo ufw --force reset

# Logging
sudo ufw logging on
sudo ufw logging medium        # off, low, medium, high, full

# View logs
sudo tail -f /var/log/ufw.log
```

---

## firewalld Management

Firewalld is the default firewall management tool on RHEL/CentOS 7+ and Fedora.

### firewalld Concepts

- **Zones**: Predefined network trust levels
- **Services**: Predefined port/protocol combinations
- **Runtime vs Permanent**: Temporary vs persistent changes

### Basic firewalld Commands

```bash
# Start/stop/enable firewalld
sudo systemctl start firewalld
sudo systemctl stop firewalld
sudo systemctl enable firewalld

# Check status
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

### Zone Management

```bash
# List available zones
sudo firewall-cmd --get-zones

# Get default zone
sudo firewall-cmd --get-default-zone

# Set default zone
sudo firewall-cmd --set-default-zone=public

# Add interface to zone
sudo firewall-cmd --zone=internal --add-interface=eth1 --permanent

# List zone configuration
sudo firewall-cmd --zone=public --list-all
```

### Service and Port Management

```bash
# List available services
sudo firewall-cmd --get-services

# Allow services
sudo firewall-cmd --add-service=http
sudo firewall-cmd --add-service=https
sudo firewall-cmd --permanent --add-service=ssh

# Allow ports
sudo firewall-cmd --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=3306/tcp

# Remove services/ports
sudo firewall-cmd --remove-service=http
sudo firewall-cmd --permanent --remove-port=8080/tcp

# Reload configuration
sudo firewall-cmd --reload
```

### Rich Rules

```bash
# Allow SSH from specific subnet
sudo firewall-cmd --add-rich-rule='rule family="ipv4" \
    source address="192.168.1.0/24" service name="ssh" accept'

# Rate limit SSH connections
sudo firewall-cmd --add-rich-rule='rule service name="ssh" \
    accept limit value="3/m"'

# Log and drop packets
sudo firewall-cmd --add-rich-rule='rule family="ipv4" \
    source address="10.0.0.0/8" log prefix="Blocked: " level="warning" drop'

# Port forwarding
sudo firewall-cmd --add-rich-rule='rule family="ipv4" \
    forward-port port="8080" protocol="tcp" to-port="80"'
```

---

## Security Hardening

### SSH Hardening

```bash
# /etc/ssh/sshd_config security settings
Port 2222                          # Change default port
Protocol 2                         # Use SSH protocol version 2
PermitRootLogin no                  # Disable root login
PasswordAuthentication no           # Use key-based authentication only
PubkeyAuthentication yes            # Enable public key authentication
AllowUsers admin john               # Limit allowed users
MaxAuthTries 3                      # Limit authentication attempts
ClientAliveInterval 300             # Client timeout
ClientAliveCountMax 2               # Max client alive messages
X11Forwarding no                    # Disable X11 forwarding
UsePAM yes                          # Use PAM for authentication

# Restart SSH service
sudo systemctl restart sshd
```

### System Hardening Script

```bash
#!/bin/bash
# system_hardening.sh

echo "Starting system hardening..."

# 1. Update system
apt update && apt upgrade -y

# 2. Install security tools
apt install -y fail2ban ufw rkhunter chkrootkit aide

# 3. Configure fail2ban
cat > /etc/fail2ban/jail.local << EOF
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[ssh]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
EOF

systemctl enable fail2ban
systemctl start fail2ban

# 4. Secure shared memory
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" >> /etc/fstab

# 5. Disable unnecessary services
services=("telnet" "rsh" "rlogin" "ftp")
for service in "${services[@]}"; do
    systemctl disable "$service" 2>/dev/null || true
done

# 6. Set password policies
cat > /etc/security/pwquality.conf << EOF
minlen = 12
minclass = 3
maxrepeat = 2
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
EOF

# 7. Configure automatic updates
cat > /etc/apt/apt.conf.d/20auto-upgrades << EOF
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
EOF

echo "System hardening completed!"
```

### File Integrity Monitoring

```bash
# Install AIDE (Advanced Intrusion Detection Environment)
sudo apt install aide

# Initialize AIDE database
sudo aide --init
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Run integrity check
sudo aide --check

# Update database after legitimate changes
sudo aide --update

# AIDE configuration example (/etc/aide/aide.conf)
/bin Binlib
/sbin Binlib
/usr/bin Binlib
/usr/sbin Binlib
/etc Security
/home Security
```

---

## Intrusion Detection

### fail2ban Configuration

```bash
# Install fail2ban
sudo apt install fail2ban

# Main configuration file: /etc/fail2ban/jail.conf
# Local overrides: /etc/fail2ban/jail.local

# Basic jail.local configuration
cat > /etc/fail2ban/jail.local << EOF
[DEFAULT]
# Ban time (seconds)
bantime = 3600
# Time window to count failures
findtime = 600
# Number of failures before ban
maxretry = 3
# Destination email for notifications
destemail = admin@example.com
# Send email on ban
action = %(action_mwl)s

[ssh]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 7200

[apache-auth]
enabled = true
port = http,https
filter = apache-auth
logpath = /var/log/apache2/*error.log
maxretry = 6

[apache-noscript]
enabled = true
port = http,https
filter = apache-noscript
logpath = /var/log/apache2/*access.log
maxretry = 6
EOF

# Restart fail2ban
sudo systemctl restart fail2ban
```

### fail2ban Management

```bash
# Check fail2ban status
sudo fail2ban-client status

# Check specific jail
sudo fail2ban-client status ssh

# Unban IP address
sudo fail2ban-client set ssh unbanip 192.168.1.100

# Ban IP address manually
sudo fail2ban-client set ssh banip 192.168.1.100

# Reload configuration
sudo fail2ban-client reload
```

### Log Monitoring Script

```bash
#!/bin/bash
# security_monitor.sh

LOG_FILE="/var/log/security_monitor.log"
AUTH_LOG="/var/log/auth.log"
CHECK_INTERVAL=300  # 5 minutes

log_alert() {
    echo "$(date): SECURITY ALERT: $1" | tee -a "$LOG_FILE"
    # Send email notification
    echo "$1" | mail -s "Security Alert" admin@example.com
}

while true; do
    # Check for multiple failed login attempts
    failed_logins=$(grep "Failed password" "$AUTH_LOG" | \
        grep "$(date '+%b %d %H:')" | wc -l)
    
    if [ "$failed_logins" -gt 10 ]; then
        log_alert "High number of failed login attempts: $failed_logins"
    fi
    
    # Check for successful root logins
    root_logins=$(grep "Accepted.*root" "$AUTH_LOG" | \
        grep "$(date '+%b %d %H:')" | wc -l)
    
    if [ "$root_logins" -gt 0 ]; then
        log_alert "Root login detected"
    fi
    
    # Check for new user creation
    new_users=$(grep "new user" "$AUTH_LOG" | \
        grep "$(date '+%b %d %H:')" | wc -l)
    
    if [ "$new_users" -gt 0 ]; then
        log_alert "New user account created"
    fi
    
    # Check for privilege escalation
    sudo_usage=$(grep "sudo:" "$AUTH_LOG" | \
        grep "$(date '+%b %d %H:')" | wc -l)
    
    if [ "$sudo_usage" -gt 20 ]; then
        log_alert "High sudo usage detected: $sudo_usage"
    fi
    
    sleep "$CHECK_INTERVAL"
done
```

---

## Practical Scenarios

### Scenario 1: Web Server Security Setup

```bash
#!/bin/bash
# web_server_security.sh

echo "Setting up web server security..."

# 1. Configure firewall for web server
ufw allow 22/tcp          # SSH
ufw allow 80/tcp          # HTTP
ufw allow 443/tcp         # HTTPS
ufw enable

# 2. Configure iptables for additional security
iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW \
    -m recent --set --name web
iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW \
    -m recent --update --seconds 60 --hitcount 20 --name web -j DROP

# 3. Set up fail2ban for Apache
cat > /etc/fail2ban/jail.d/apache-custom.conf << EOF
[apache-auth]
enabled = true
port = http,https
filter = apache-auth
logpath = /var/log/apache2/error.log
maxretry = 3
bantime = 3600

[apache-badbots]
enabled = true
port = http,https
filter = apache-badbots
logpath = /var/log/apache2/access.log
maxretry = 2
bantime = 86400
EOF

# 4. Harden Apache configuration
cat >> /etc/apache2/apache2.conf << EOF
# Security headers
Header always set X-Content-Type-Options nosniff
Header always set X-Frame-Options DENY
Header always set X-XSS-Protection "1; mode=block"
Header always set Strict-Transport-Security "max-age=31536000"

# Hide server information
ServerTokens Prod
ServerSignature Off
EOF

systemctl restart apache2
systemctl restart fail2ban

echo "Web server security setup completed!"
```

### Scenario 2: SSH Bastion Host Configuration

```bash
#!/bin/bash
# setup_bastion_host.sh

BASTION_USER="bastion"
INTERNAL_NETWORK="10.0.1.0/24"

echo "Setting up SSH bastion host..."

# 1. Create bastion user
useradd -m -s /bin/bash "$BASTION_USER"
mkdir -p "/home/$BASTION_USER/.ssh"
chmod 700 "/home/$BASTION_USER/.ssh"

# 2. Configure SSH for bastion
cat > /etc/ssh/sshd_config << EOF
Port 22
Protocol 2
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
AllowTcpForwarding yes
GatewayPorts no
ClientAliveInterval 300
ClientAliveCountMax 2

# Restrict users to bastion functionality
Match User $BASTION_USER
    AllowTcpForwarding yes
    X11Forwarding no
    PermitTunnel no
    GatewayPorts no
    AllowAgentForwarding yes
    ForceCommand /usr/local/bin/bastion-shell
EOF

# 3. Create bastion shell script
cat > /usr/local/bin/bastion-shell << 'EOF'
#!/bin/bash
# Bastion host shell - restrict user actions

echo "Welcome to the bastion host"
echo "Allowed destinations:"
echo "- 10.0.1.0/24 (Internal network)"
echo ""

while true; do
    read -p "bastion> " cmd args
    
    case "$cmd" in
        ssh)
            # Validate destination IP
            if echo "$args" | grep -qE "^[^@]+@10\.0\.1\.[0-9]+$"; then
                ssh $args
            else
                echo "Access denied. Only internal network allowed."
            fi
            ;;
        scp)
            # Validate SCP operations
            if echo "$args" | grep -qE "10\.0\.1\.[0-9]+"; then
                scp $args
            else
                echo "Access denied. Only internal network allowed."
            fi
            ;;
        exit|quit)
            break
            ;;
        help)
            echo "Available commands: ssh, scp, exit, help"
            ;;
        *)
            echo "Command not allowed. Type 'help' for available commands."
            ;;
    esac
done
EOF

chmod +x /usr/local/bin/bastion-shell

# 4. Configure firewall
ufw allow from any to any port 22
ufw allow from "$INTERNAL_NETWORK"
ufw enable

systemctl restart sshd
echo "Bastion host configuration completed!"
```

### Scenario 3: Network Intrusion Detection

```bash
#!/bin/bash
# network_ids.sh - Simple network intrusion detection

INTERFACE="eth0"
LOG_FILE="/var/log/network_ids.log"
ALERT_THRESHOLD=100

log_event() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

# Monitor for port scans
monitor_port_scans() {
    tcpdump -i "$INTERFACE" -c 1000 -n 'tcp[tcpflags] & (tcp-syn) != 0 and tcp[tcpflags] & (tcp-ack) = 0' 2>/dev/null | \
    awk '{print $3}' | cut -d'.' -f1-4 | sort | uniq -c | \
    while read count ip; do
        if [ "$count" -gt "$ALERT_THRESHOLD" ]; then
            log_event "ALERT: Possible port scan from $ip ($count attempts)"
            # Automatically block suspicious IP
            iptables -A INPUT -s "$ip" -j DROP
        fi
    done
}

# Monitor for unusual traffic patterns
monitor_traffic() {
    ss -tuln | grep LISTEN | wc -l > /tmp/baseline_ports
    
    while true; do
        current_ports=$(ss -tuln | grep LISTEN | wc -l)
        baseline_ports=$(cat /tmp/baseline_ports)
        
        if [ "$current_ports" -gt $((baseline_ports + 5)) ]; then
            log_event "ALERT: Unusual number of listening ports detected"
            ss -tuln | grep LISTEN | diff - /tmp/baseline_ports.list
        fi
        
        sleep 300
    done
}

# Start monitoring
log_event "Starting network intrusion detection"
monitor_port_scans &
monitor_traffic &

wait
```

---

## Lab Exercises

### Exercise 1: Basic Firewall Setup
1. Configure iptables rules for a web server
2. Test connectivity and rule effectiveness
3. Make rules persistent across reboots
4. Document the firewall configuration

### Exercise 2: SSH Security Hardening
1. Change SSH default port
2. Disable password authentication
3. Configure key-based authentication
4. Set up fail2ban for SSH protection
5. Test security measures

### Exercise 3: Network Access Control
1. Create network zones with different trust levels
2. Configure inter-zone communication rules
3. Set up port forwarding for specific services
4. Implement traffic filtering and logging

### Exercise 4: Intrusion Detection Setup
1. Install and configure fail2ban
2. Set up log monitoring for security events
3. Create custom filters for application logs
4. Test detection and response mechanisms

---

## Quick Reference Commands

```bash
# iptables
iptables -L -n -v                    # List rules
iptables -A INPUT -p tcp --dport 80 -j ACCEPT  # Allow HTTP
iptables-save > /etc/iptables/rules.v4  # Save rules

# UFW
ufw enable                           # Enable firewall
ufw allow 22                         # Allow SSH
ufw status numbered                  # Show rules with numbers

# firewalld
firewall-cmd --list-all             # Show configuration
firewall-cmd --add-service=http     # Allow HTTP
firewall-cmd --reload               # Reload configuration

# fail2ban
fail2ban-client status              # Show status
fail2ban-client status ssh          # Show SSH jail status
fail2ban-client set ssh unbanip IP  # Unban IP

# Security monitoring
tail -f /var/log/auth.log           # Monitor authentication
grep "Failed password" /var/log/auth.log  # Find failed logins
last                                # Show login history
```
