# Network Information System (NIS)

## Table of Contents
1. [NIS Overview and Concepts](#nis-overview-and-concepts)
2. [NIS Architecture](#nis-architecture)
3. [Setting Up NIS Master Server](#setting-up-nis-master-server)
4. [Configuring NIS Slave Server](#configuring-nis-slave-server)
5. [NIS Client Configuration](#nis-client-configuration)
6. [Managing NIS Maps](#managing-nis-maps)
7. [NIS Security](#nis-security)
8. [Advanced NIS Configuration](#advanced-nis-configuration)
9. [Practical Scenarios](#practical-scenarios)

---

## NIS Overview and Concepts

### What is NIS?

Network Information System (NIS), originally called Yellow Pages (YP), is a client-server directory service protocol for distributing system configuration data such as:

- User accounts and passwords
- Group information
- Host names and IP addresses
- Network services and protocols
- Automount maps

### Key Benefits

1. **Centralized Management**: Single point of administration for user accounts
2. **Consistency**: Same user information across all networked systems
3. **Scalability**: Easy to add new systems to the network
4. **Simplicity**: Straightforward configuration and maintenance
5. **Integration**: Works well with other network services

### Common Use Cases in AI Lab

- **Multi-user Workstations**: Researchers accessing different machines
- **Compute Clusters**: Consistent user environment across nodes
- **Development Environment**: Same user credentials for all tools
- **Resource Sharing**: Authentication for shared GPU/storage resources

---

## NIS Architecture

### Components

```
NIS Domain: ailab.local
========================

    [NIS Master Server]
         |
    ┌────┴────┐
    |         |
[Slave 1] [Slave 2]     [NIS Clients]
    |         |         /    |    \
    └─────────┼────────┘     |     \
              |              |      \
         [Client 1]    [Client 2]  [Client 3]
```

### NIS Maps (Databases)

| Map Name | Purpose | Source File |
|----------|---------|-------------|
| passwd.byname | User accounts by username | /etc/passwd |
| passwd.byuid | User accounts by UID | /etc/passwd |
| group.byname | Groups by name | /etc/group |
| group.bygid | Groups by GID | /etc/group |
| hosts.byname | Hostnames by name | /etc/hosts |
| hosts.byaddr | Hostnames by IP | /etc/hosts |
| networks.byname | Network names | /etc/networks |
| services.byname | Network services | /etc/services |
| protocols.byname | Network protocols | /etc/protocols |

### NIS Process Flow

1. **Client Request**: NIS client needs user information
2. **Local Cache Check**: Client checks local cache first
3. **NIS Query**: Query sent to NIS server if not cached
4. **Map Lookup**: Server searches appropriate NIS map
5. **Response**: Information returned to client
6. **Caching**: Client caches result for future use

---

## Setting Up NIS Master Server

### Installation

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nis rpcbind

# CentOS/RHEL
sudo yum install ypserv ypbind rpcbind

# Enable and start services
sudo systemctl enable rpcbind ypserv
sudo systemctl start rpcbind ypserv
```

### Domain Configuration

```bash
# Set NIS domain name
echo "ailab.local" | sudo tee /etc/defaultdomain

# Configure domain in /etc/sysconfig/network (RHEL/CentOS)
echo "NISDOMAIN=ailab.local" | sudo tee -a /etc/sysconfig/network

# Set domain for current session
sudo domainname ailab.local
```

### Master Server Configuration

#### /etc/ypserv.conf

```bash
# Main NIS server configuration file
sudo tee /etc/ypserv.conf << 'EOF'
# Global settings
dns: no
files: 30

# Access control - restrict by network
# Format: map:host(network/netmask):security:mangle:field
*:*:none

# Allow access from lab network
*:192.168.1.0/255.255.255.0:none
*:10.0.0.0/255.0.0.0:none

# Specific map restrictions
passwd.byname:*:port:yes
passwd.byuid:*:port:yes
shadow.byname:*:port:yes

# Master server settings
slp: no
xfr_check_port: yes
EOF
```

#### Initialize NIS Database

```bash
# Initialize NIS maps from local files
sudo /usr/lib/yp/ypinit -m

# The script will prompt for slave servers (optional for now)
# Just press Enter to continue with master only

# Verify map creation
sudo ls -la /var/yp/ailab.local/
```

### User Management Setup

#### Create NIS Users

```bash
#!/bin/bash
# setup_nis_users.sh - Create users for NIS

NIS_DOMAIN="ailab.local"
USERS_FILE="nis_users.txt"

# Create user list file
cat > "$USERS_FILE" << 'EOF'
# Format: username:uid:gid:fullname:home:shell
researcher1:5001:1000:John Doe:/home/researcher1:/bin/bash
researcher2:5002:1000:Jane Smith:/home/researcher2:/bin/bash
student1:6001:1001:Alice Johnson:/home/student1:/bin/bash
student2:6002:1001:Bob Wilson:/home/student2:/bin/bash
admin_user:4001:1002:Admin User:/home/admin_user:/bin/bash
EOF

# Process each user
while IFS=':' read -r username uid gid fullname homedir shell; do
    # Skip comments and empty lines
    [[ $username =~ ^#.*$ || -z $username ]] && continue
    
    echo "Creating user: $username"
    
    # Create user account
    sudo useradd -u "$uid" -g "$gid" -c "$fullname" \
                 -d "$homedir" -s "$shell" "$username"
    
    # Set temporary password (users should change on first login)
    echo "$username:TempPass123!" | sudo chpasswd
    sudo passwd -e "$username"  # Force password change
    
    # Create home directory if it doesn't exist
    if [ ! -d "$homedir" ]; then
        sudo mkdir -p "$homedir"
        sudo cp -r /etc/skel/. "$homedir/"
        sudo chown -R "$username:$gid" "$homedir"
        sudo chmod 755 "$homedir"
    fi
    
    echo "User $username created successfully"
done < "$USERS_FILE"

# Update NIS maps
echo "Updating NIS maps..."
cd /var/yp
sudo make

echo "NIS user setup completed!"
```

### Service Management

```bash
# Start NIS master services
sudo systemctl start rpcbind
sudo systemctl start ypserv
sudo systemctl start ypxfrd    # For slave server updates

# Enable services for boot
sudo systemctl enable rpcbind ypserv ypxfrd

# Check service status
sudo systemctl status ypserv
sudo systemctl status rpcbind

# Test NIS maps
ypcat passwd.byname
ypcat group.byname
ypmatch researcher1 passwd.byname
```

---

## Configuring NIS Slave Server

### Installation and Basic Setup

```bash
# Install NIS packages
sudo apt install nis rpcbind

# Set domain name
echo "ailab.local" | sudo tee /etc/defaultdomain
sudo domainname ailab.local
```

### Slave Server Configuration

```bash
# Initialize as slave server
sudo /usr/lib/yp/ypinit -s nis-master.ailab.local

# The script will:
# 1. Contact the master server
# 2. Download all NIS maps
# 3. Set up slave configuration

# Verify slave setup
sudo ls -la /var/yp/ailab.local/
```

### Automatic Map Updates

#### Configure Map Transfer

```bash
# /etc/cron.d/nis-slave-update
sudo tee /etc/cron.d/nis-slave-update << 'EOF'
# Update NIS maps from master every 30 minutes
*/30 * * * * root /usr/lib/yp/ypxfr_1perhour
0 2 * * * root /usr/lib/yp/ypxfr_1perday
0 3 * * 0 root /usr/lib/yp/ypxfr_2perweek
EOF
```

### High Availability Setup

```bash
#!/bin/bash
# setup_nis_ha.sh - High availability NIS configuration

MASTER_SERVER="nis-master.ailab.local"
SLAVE_SERVERS=("nis-slave1.ailab.local" "nis-slave2.ailab.local")

# Configure master for slave support
configure_master() {
    echo "Configuring master server for slaves..."
    
    # Update master's ypserv configuration
    sudo tee -a /etc/ypserv.conf << 'EOF'

# Slave server configuration
# Allow slaves to transfer maps
*:192.168.1.10:none    # slave1
*:192.168.1.11:none    # slave2
EOF
    
    # Restart master services
    sudo systemctl restart ypserv ypxfrd
}

# Monitor slave synchronization
check_slave_sync() {
    local slave_server="$1"
    
    echo "Checking synchronization with $slave_server..."
    
    # Compare map timestamps
    master_time=$(ypcat -h "$MASTER_SERVER" passwd.byname | wc -l)
    slave_time=$(ypcat -h "$slave_server" passwd.byname | wc -l)
    
    if [ "$master_time" -eq "$slave_time" ]; then
        echo "✓ $slave_server is synchronized ($slave_time entries)"
    else
        echo "✗ $slave_server out of sync (Master: $master_time, Slave: $slave_time)"
        return 1
    fi
}

# Main execution
if [[ $(hostname -f) == "$MASTER_SERVER" ]]; then
    configure_master
else
    echo "Run this script on the NIS master server"
    exit 1
fi

# Check all slaves
for slave in "${SLAVE_SERVERS[@]}"; do
    check_slave_sync "$slave"
done
```

---

## NIS Client Configuration

### Client Installation

```bash
# Install NIS client packages
sudo apt install nis libnss-nis

# Set NIS domain
echo "ailab.local" | sudo tee /etc/defaultdomain
sudo domainname ailab.local
```

### NSS Configuration

#### /etc/nsswitch.conf

```bash
# Configure name service switch to use NIS
sudo cp /etc/nsswitch.conf /etc/nsswitch.conf.backup

sudo tee /etc/nsswitch.conf << 'EOF'
# Name Service Switch configuration file.

passwd:         files nis
group:          files nis
shadow:         files nis

hosts:          files dns nis
networks:       files nis

protocols:      db files nis
services:       db files nis
ethers:         db files nis
rpc:            db files nis

netgroup:       nis
automount:      files nis
EOF
```

### Authentication Configuration

#### PAM Configuration (Ubuntu/Debian)

```bash
# Install additional PAM modules
sudo apt install libpam-modules

# Update PAM configuration for NIS authentication
sudo pam-auth-update --enable nis

# Manual PAM configuration if needed
# /etc/pam.d/common-auth
auth    [success=2 default=ignore]      pam_unix.so nullok_secure
auth    [success=1 default=ignore]      pam_nis.so
auth    requisite                       pam_deny.so
auth    required                        pam_permit.so

# /etc/pam.d/common-account
account [success=1 new_authtok_reqd=done default=ignore]        pam_unix.so
account requisite                       pam_deny.so
account required                        pam_permit.so

# /etc/pam.d/common-password
password        [success=2 default=ignore]      pam_unix.so obscure sha512
password        [success=1 default=ignore]      pam_nis.so
password        requisite                       pam_deny.so
password        required                        pam_permit.so
```

### Client Services

```bash
# Start NIS client services
sudo systemctl start rpcbind ypbind
sudo systemctl enable rpcbind ypbind

# Test NIS connectivity
ypwhich                    # Show which NIS server we're bound to
ypcat passwd.byname        # List all NIS users
ypmatch researcher1 passwd # Get specific user info

# Test user authentication
getent passwd researcher1  # Should show NIS user info
id researcher1             # Should show user details
```

### Home Directory Setup

```bash
# Create home directories for NIS users
sudo mkdir -p /home

# Set proper permissions
sudo chmod 755 /home

# Option 1: Auto-create home directories on login
sudo tee -a /etc/pam.d/common-session << 'EOF'
session required pam_mkhomedir.so skel=/etc/skel umask=022
EOF

# Option 2: Pre-create home directories
#!/bin/bash
# create_nis_homes.sh

NIS_DOMAIN="ailab.local"

# Get list of NIS users
ypcat passwd.byname | while IFS=':' read username x uid gid gecos home shell; do
    # Skip system users (UID < 1000)
    if [ "$uid" -ge 1000 ]; then
        echo "Creating home directory for $username"
        
        if [ ! -d "$home" ]; then
            sudo mkdir -p "$home"
            sudo cp -r /etc/skel/. "$home/"
            sudo chown -R "$uid:$gid" "$home"
            sudo chmod 755 "$home"
            echo "Home directory created: $home"
        else
            echo "Home directory already exists: $home"
        fi
    fi
done
```

---

## Managing NIS Maps

### Understanding NIS Maps

NIS maps are essentially databases that store various system information. Each map has a specific purpose and is built from corresponding system files.

### Map Building Process

```bash
# Manual map rebuild (run on master)
cd /var/yp
sudo make

# Rebuild specific map
cd /var/yp
sudo make passwd

# Force complete rebuild
cd /var/yp
sudo make clean
sudo make
```

### Custom NIS Maps

#### Creating Custom Maps

```bash
#!/bin/bash
# create_custom_maps.sh - Create custom NIS maps for AI Lab

# Create software versions map
sudo tee /etc/software_versions << 'EOF'
python3:3.11.2
pytorch:2.0.1
tensorflow:2.13.0
cuda:11.8
cudnn:8.6.0
jupyter:4.0.3
vscode:1.82.0
EOF

# Create GPU allocation map
sudo tee /etc/gpu_allocation << 'EOF'
gpu-node1:researcher1,researcher2
gpu-node2:researcher3,student1
gpu-node3:available
gpu-node4:maintenance
EOF

# Update Makefile to include custom maps
sudo tee -a /var/yp/Makefile << 'EOF'

# Custom AI Lab maps
software_versions.byname: /etc/software_versions
	@echo "Updating $@..."
	$(MAKEDBM) /etc/software_versions $(YPMAPDIR)/$@

gpu_allocation.byname: /etc/gpu_allocation
	@echo "Updating $@..."
	$(MAKEDBM) /etc/gpu_allocation $(YPMAPDIR)/$@

# Add to all target
all: passwd group hosts networks protocols services \
     software_versions.byname gpu_allocation.byname
EOF

# Build custom maps
cd /var/yp
sudo make software_versions.byname gpu_allocation.byname
```

### Map Maintenance Scripts

```bash
#!/bin/bash
# nis_maintenance.sh - NIS map maintenance and monitoring

NIS_DOMAIN="ailab.local"
LOG_FILE="/var/log/nis_maintenance.log"
EMAIL="admin@ailab.local"

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

# Check map consistency
check_map_consistency() {
    local map_name="$1"
    local master_server="nis-master.ailab.local"
    local slave_servers=("nis-slave1.ailab.local" "nis-slave2.ailab.local")
    
    log_message "Checking consistency for map: $map_name"
    
    # Get master map checksum
    master_checksum=$(ypcat -h "$master_server" "$map_name" | md5sum | cut -d' ' -f1)
    log_message "Master checksum for $map_name: $master_checksum"
    
    # Check each slave
    for slave in "${slave_servers[@]}"; do
        slave_checksum=$(ypcat -h "$slave" "$map_name" 2>/dev/null | md5sum | cut -d' ' -f1)
        
        if [ "$master_checksum" = "$slave_checksum" ]; then
            log_message "✓ $slave synchronized for $map_name"
        else
            log_message "✗ $slave out of sync for $map_name"
            # Trigger map transfer
            ssh "$slave" "sudo /usr/lib/yp/ypxfr $map_name"
        fi
    done
}

# Monitor NIS service health
check_service_health() {
    local servers=("nis-master.ailab.local" "nis-slave1.ailab.local" "nis-slave2.ailab.local")
    
    for server in "${servers[@]}"; do
        if ping -c 1 "$server" >/dev/null 2>&1; then
            if ypcat -h "$server" passwd.byname >/dev/null 2>&1; then
                log_message "✓ $server is healthy"
            else
                log_message "✗ $server not responding to NIS queries"
                echo "$server NIS service issue" | mail -s "NIS Alert" "$EMAIL"
            fi
        else
            log_message "✗ $server unreachable"
            echo "$server unreachable" | mail -s "NIS Alert" "$EMAIL"
        fi
    done
}

# Update user statistics
update_user_stats() {
    local stats_file="/tmp/nis_user_stats.txt"
    
    {
        echo "NIS User Statistics - $(date)"
        echo "============================="
        echo
        echo "Total Users: $(ypcat passwd.byname | wc -l)"
        echo "Total Groups: $(ypcat group.byname | wc -l)"
        echo
        echo "Users by UID range:"
        echo "System (0-999): $(ypcat passwd.byname | awk -F: '$3 < 1000' | wc -l)"
        echo "Regular (1000-4999): $(ypcat passwd.byname | awk -F: '$3 >= 1000 && $3 < 5000' | wc -l)"
        echo "Research (5000-5999): $(ypcat passwd.byname | awk -F: '$3 >= 5000 && $3 < 6000' | wc -l)"
        echo "Students (6000-6999): $(ypcat passwd.byname | awk -F: '$3 >= 6000 && $3 < 7000' | wc -l)"
        echo
        echo "Recent logins (last 7 days):"
        last | grep -v "reboot\|shutdown" | head -20
    } > "$stats_file"
    
    log_message "User statistics updated: $stats_file"
}

# Main execution
log_message "Starting NIS maintenance check"

# Check critical maps
maps=("passwd.byname" "group.byname" "hosts.byname")
for map in "${maps[@]}"; do
    check_map_consistency "$map"
done

check_service_health
update_user_stats

log_message "NIS maintenance check completed"
```

---

## NIS Security

### Security Considerations

NIS has several inherent security limitations:
1. **Plain text transmission**: Data sent unencrypted
2. **No authentication**: Anyone can query NIS maps
3. **Password exposure**: Encrypted passwords visible in maps
4. **Broadcast domain**: Maps accessible to entire network segment

### Securing NIS

#### Network-based Security

```bash
# Firewall rules for NIS (iptables)
#!/bin/bash
# nis_firewall.sh

NIS_NETWORK="192.168.1.0/24"
NIS_MASTER="192.168.1.10"
NIS_SLAVES="192.168.1.11 192.168.1.12"

# Allow NIS traffic only from specific networks
iptables -A INPUT -p tcp --dport 111 -s "$NIS_NETWORK" -j ACCEPT  # RPC portmapper
iptables -A INPUT -p udp --dport 111 -s "$NIS_NETWORK" -j ACCEPT
iptables -A INPUT -p tcp --dport 834 -s "$NIS_NETWORK" -j ACCEPT  # ypserv
iptables -A INPUT -p udp --dport 834 -s "$NIS_NETWORK" -j ACCEPT
iptables -A INPUT -p tcp --dport 835 -s "$NIS_NETWORK" -j ACCEPT  # ypbind
iptables -A INPUT -p udp --dport 835 -s "$NIS_NETWORK" -j ACCEPT

# Block NIS from external networks
iptables -A INPUT -p tcp --dport 111 -j DROP
iptables -A INPUT -p udp --dport 111 -j DROP
iptables -A INPUT -p tcp --dport 834 -j DROP
iptables -A INPUT -p udp --dport 834 -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

#### Access Control

```bash
# Enhanced /etc/ypserv.conf with strict access control
sudo tee /etc/ypserv.conf << 'EOF'
# Strict NIS access control

# Deny all by default
*:*:deny

# Allow specific networks for specific maps
passwd.byname:192.168.1.0/255.255.255.0:none
passwd.byuid:192.168.1.0/255.255.255.0:none
group.byname:192.168.1.0/255.255.255.0:none
group.bygid:192.168.1.0/255.255.255.0:none

# Restrict shadow map to master and slaves only
shadow.byname:192.168.1.10:none        # Master
shadow.byname:192.168.1.11:none        # Slave 1
shadow.byname:192.168.1.12:none        # Slave 2
shadow.byname:*:deny                   # Deny all others

# Administrative maps
hosts.byname:192.168.1.0/255.255.255.0:none
networks.byname:192.168.1.0/255.255.255.0:none
services.byname:192.168.1.0/255.255.255.0:none

# Security settings
dns: no
files: 30
slp: no
xfr_check_port: yes
EOF
```

#### Shadow Password Protection

```bash
#!/bin/bash
# secure_shadow.sh - Secure shadow password handling

# Create restricted shadow map
create_shadow_map() {
    echo "Creating secure shadow map..."
    
    # Filter shadow file to remove sensitive information
    awk -F: '{
        # Keep username and encrypted password, zero out other fields
        printf "%s:%s:0:0:99999:7:::\n", $1, $2
    }' /etc/shadow > /tmp/shadow.nis
    
    # Build shadow map with restricted file
    cd /var/yp
    sudo /usr/lib/yp/makedbm /tmp/shadow.nis /var/yp/ailab.local/shadow.byname
    
    # Clean up temporary file
    sudo rm /tmp/shadow.nis
    
    # Set restrictive permissions on shadow map
    sudo chmod 600 /var/yp/ailab.local/shadow.byname
    sudo chown root:root /var/yp/ailab.local/shadow.byname
}

# Modify Makefile to exclude shadow by default
secure_makefile() {
    sudo cp /var/yp/Makefile /var/yp/Makefile.backup
    
    # Remove shadow from default targets
    sudo sed -i 's/shadow\.byname//g' /var/yp/Makefile
    
    echo "Makefile secured - shadow maps must be built manually"
}

create_shadow_map
secure_makefile
```

### Monitoring and Auditing

```bash
#!/bin/bash
# nis_audit.sh - Security auditing for NIS

AUDIT_LOG="/var/log/nis_security_audit.log"
ALERT_EMAIL="security@ailab.local"

audit_message() {
    echo "$(date): $1" | tee -a "$AUDIT_LOG"
}

# Check for unauthorized access attempts
check_access_logs() {
    audit_message "Checking NIS access logs..."
    
    # Check for unusual query patterns
    grep ypserv /var/log/syslog | grep -E "(refused|denied|unauthorized)" | \
        tail -10 | while read line; do
            audit_message "SECURITY: $line"
            echo "$line" | mail -s "NIS Security Alert" "$ALERT_EMAIL"
        done
}

# Verify map integrity
verify_map_integrity() {
    local critical_maps=("passwd.byname" "shadow.byname" "group.byname")
    
    for map in "${critical_maps[@]}"; do
        audit_message "Verifying integrity of $map"
        
        # Check map permissions
        map_file="/var/yp/ailab.local/$map"
        if [ -f "$map_file" ]; then
            perms=$(stat -c "%a" "$map_file")
            owner=$(stat -c "%U:%G" "$map_file")
            
            case "$map" in
                "shadow.byname")
                    if [ "$perms" != "600" ] || [ "$owner" != "root:root" ]; then
                        audit_message "SECURITY: $map has incorrect permissions ($perms) or owner ($owner)"
                    fi
                    ;;
                *)
                    if [ "$perms" != "644" ] || [ "$owner" != "root:root" ]; then
                        audit_message "WARNING: $map has unusual permissions ($perms) or owner ($owner)"
                    fi
                    ;;
            esac
        fi
    done
}

# Check for rogue NIS servers
detect_rogue_servers() {
    audit_message "Scanning for rogue NIS servers..."
    
    # Scan network for NIS services
    nmap -p 111,834,835 192.168.1.0/24 | grep -B5 -A5 "open" | \
        grep -E "Nmap scan report|111/tcp.*open|834/tcp.*open|835/tcp.*open" > /tmp/nis_scan.txt
    
    # Parse results and check against known servers
    known_servers=("192.168.1.10" "192.168.1.11" "192.168.1.12")
    
    grep "Nmap scan report" /tmp/nis_scan.txt | while read line; do
        server_ip=$(echo "$line" | awk '{print $5}' | tr -d '()')
        
        if [[ ! " ${known_servers[@]} " =~ " ${server_ip} " ]]; then
            if grep -A3 "$server_ip" /tmp/nis_scan.txt | grep -q "834/tcp.*open"; then
                audit_message "SECURITY: Rogue NIS server detected at $server_ip"
                echo "Unauthorized NIS server detected at $server_ip" | \
                    mail -s "CRITICAL: Rogue NIS Server" "$ALERT_EMAIL"
            fi
        fi
    done
    
    rm /tmp/nis_scan.txt
}

# Main audit execution
audit_message "Starting NIS security audit"
check_access_logs
verify_map_integrity
detect_rogue_servers
audit_message "NIS security audit completed"
```

---

## Advanced NIS Configuration

### Multi-domain NIS Setup

```bash
#!/bin/bash
# multi_domain_nis.sh - Configure multiple NIS domains

# Domain configuration
RESEARCH_DOMAIN="research.ailab.local"
STUDENT_DOMAIN="student.ailab.local"
ADMIN_DOMAIN="admin.ailab.local"

setup_multi_domain() {
    local domain="$1"
    local base_uid="$2"
    
    echo "Setting up NIS domain: $domain"
    
    # Create domain-specific directory
    sudo mkdir -p "/var/yp/$domain"
    
    # Domain-specific user file
    case "$domain" in
        "$RESEARCH_DOMAIN")
            create_research_users "$base_uid" "$domain"
            ;;
        "$STUDENT_DOMAIN")
            create_student_users "$base_uid" "$domain"
            ;;
        "$ADMIN_DOMAIN")
            create_admin_users "$base_uid" "$domain"
            ;;
    esac
    
    # Build maps for domain
    cd /var/yp
    sudo make NOPUSH=true DOMAIN="$domain"
}

create_research_users() {
    local base_uid="$1"
    local domain="$2"
    
    # Create research-specific users
    for i in {1..20}; do
        username="researcher$i"
        uid=$((base_uid + i))
        
        sudo useradd -u "$uid" -g researchers -c "Researcher $i" \
                     -d "/home/research/$username" -s /bin/bash "$username"
        echo "$username:ResearchPass123!" | sudo chpasswd
        sudo passwd -e "$username"
    done
}

create_student_users() {
    local base_uid="$1"
    local domain="$2"
    
    # Create student-specific users
    for i in {1..50}; do
        username="student$i"
        uid=$((base_uid + i))
        
        sudo useradd -u "$uid" -g students -c "Student $i" \
                     -d "/home/students/$username" -s /bin/bash "$username"
        echo "$username:StudentPass123!" | sudo chpasswd
        sudo passwd -e "$username"
    done
}

# Set up domains
setup_multi_domain "$RESEARCH_DOMAIN" 5000
setup_multi_domain "$STUDENT_DOMAIN" 6000
setup_multi_domain "$ADMIN_DOMAIN" 4000
```

### NIS with LDAP Integration

```bash
#!/bin/bash
# nis_ldap_integration.sh - Integrate NIS with LDAP backend

# Install required packages
install_ldap_support() {
    sudo apt update
    sudo apt install slapd ldap-utils libnss-ldap libpam-ldap
}

# Configure NIS to use LDAP as backend
configure_nis_ldap() {
    # Create LDAP-backed NIS configuration
    sudo tee /etc/ypldapd.conf << 'EOF'
# LDAP server configuration
uri ldap://ldap.ailab.local:389
base dc=ailab,dc=local
binddn cn=nisadmin,dc=ailab,dc=local
bindpw secret_password

# Map configurations
user_base ou=users,dc=ailab,dc=local
group_base ou=groups,dc=ailab,dc=local

# Attribute mappings
user_filter (objectClass=posixAccount)
group_filter (objectClass=posixGroup)

# NIS domain
domain ailab.local
EOF

    # Start LDAP-NIS daemon
    sudo systemctl enable ypldapd
    sudo systemctl start ypldapd
}

install_ldap_support
configure_nis_ldap
```

---

## Practical Scenarios

### Scenario 1: AI Research Lab Setup

```bash
#!/bin/bash
# research_lab_nis.sh - Complete NIS setup for AI research lab

LAB_DOMAIN="ailab.research"
MASTER_SERVER="nis-master.research.local"
NFS_SERVER="storage.research.local"

# User categories with different UID ranges
FACULTY_UID_START=5000
RESEARCHER_UID_START=5100
STUDENT_UID_START=6000
VISITOR_UID_START=7000

setup_research_environment() {
    echo "Setting up AI Research Lab NIS environment..."
    
    # Create groups
    sudo groupadd -g 1001 faculty
    sudo groupadd -g 1002 researchers  
    sudo groupadd -g 1003 phd_students
    sudo groupadd -g 1004 masters_students
    sudo groupadd -g 1005 visitors
    sudo groupadd -g 1006 gpu_users
    sudo groupadd -g 1007 cluster_users
    
    # Create faculty accounts
    create_faculty_accounts
    
    # Create researcher accounts
    create_researcher_accounts
    
    # Create student accounts
    create_student_accounts
    
    # Set up special access groups
    setup_resource_groups
    
    # Configure research-specific maps
    create_research_maps
}

create_faculty_accounts() {
    local faculty_list=(
        "prof_smith:Dr. John Smith:john.smith@ailab.edu"
        "prof_jones:Dr. Sarah Jones:sarah.jones@ailab.edu"
        "prof_brown:Dr. Michael Brown:michael.brown@ailab.edu"
    )
    
    local uid=$FACULTY_UID_START
    
    for entry in "${faculty_list[@]}"; do
        IFS=':' read -r username fullname email <<< "$entry"
        
        echo "Creating faculty account: $username"
        sudo useradd -u $uid -g faculty -G gpu_users,cluster_users \
                     -c "$fullname" -d "/home/faculty/$username" \
                     -s /bin/bash "$username"
        
        # Set up faculty-specific environment
        setup_faculty_environment "$username" "$email"
        
        ((uid++))
    done
}

create_researcher_accounts() {
    # Postdoc researchers
    for i in {1..10}; do
        username="postdoc$i"
        uid=$((RESEARCHER_UID_START + i))
        
        sudo useradd -u $uid -g researchers -G gpu_users,cluster_users \
                     -c "Postdoc Researcher $i" -d "/home/researchers/$username" \
                     -s /bin/bash "$username"
        
        setup_researcher_environment "$username"
    done
}

setup_resource_groups() {
    # GPU allocation groups
    sudo groupadd -g 2001 gpu_high_priority
    sudo groupadd -g 2002 gpu_medium_priority
    sudo groupadd -g 2003 gpu_low_priority
    
    # Storage quota groups
    sudo groupadd -g 2010 storage_100gb
    sudo groupadd -g 2011 storage_500gb
    sudo groupadd -g 2012 storage_unlimited
    
    # Cluster access groups
    sudo groupadd -g 2020 cluster_cpu
    sudo groupadd -g 2021 cluster_gpu
    sudo groupadd -g 2022 cluster_high_mem
}

create_research_maps() {
    # Research project mapping
    sudo tee /etc/research_projects << 'EOF'
nlp_project:prof_smith,postdoc1,phd_student1,phd_student2
cv_project:prof_jones,postdoc2,postdoc3,masters_student1
robotics_project:prof_brown,postdoc4,visitor1
EOF

    # GPU allocation mapping
    sudo tee /etc/gpu_scheduling << 'EOF'
gpu01:nlp_project:0800-1800:weekdays
gpu02:cv_project:0900-2100:everyday
gpu03:robotics_project:weekend
gpu04:available
EOF

    # Software environment mapping
    sudo tee /etc/software_stacks << 'EOF'
pytorch_2.0:cuda11.8,python3.11,jupyter,tensorboard
tensorflow_2.13:cuda11.8,python3.10,jupyter,wandb
research_base:git,vim,tmux,htop,nvtop
EOF

    # Update NIS Makefile
    sudo tee -a /var/yp/Makefile << 'EOF'

# Research-specific maps
research_projects.byname: /etc/research_projects
	@echo "Updating $@..."
	$(MAKEDBM) /etc/research_projects $(YPMAPDIR)/$@

gpu_scheduling.byname: /etc/gpu_scheduling
	@echo "Updating $@..."
	$(MAKEDBM) /etc/gpu_scheduling $(YPMAPDIR)/$@

software_stacks.byname: /etc/software_stacks
	@echo "Updating $@..."
	$(MAKEDBM) /etc/software_stacks $(YPMAPDIR)/$@

# Add to all target
all: passwd group hosts networks protocols services \
     research_projects.byname gpu_scheduling.byname software_stacks.byname
EOF

    # Build all maps
    cd /var/yp
    sudo make
}

setup_faculty_environment() {
    local username="$1"
    local email="$2"
    local homedir="/home/faculty/$username"
    
    # Create home directory structure
    sudo mkdir -p "$homedir"/{Projects,Publications,Datasets,Models}
    sudo mkdir -p "$homedir"/.config/{jupyter,git}
    
    # Set up Git configuration
    sudo tee "$homedir/.config/git/config" << EOF
[user]
    name = $username
    email = $email
[core]
    editor = vim
[push]
    default = simple
EOF
    
    # Jupyter configuration for research
    sudo tee "$homedir/.config/jupyter/jupyter_notebook_config.py" << 'EOF'
c.NotebookApp.ip = '0.0.0.0'
c.NotebookApp.open_browser = False
c.NotebookApp.port = 8888
c.NotebookApp.notebook_dir = '/home/faculty'
EOF
    
    # Set ownership
    sudo chown -R "$username:faculty" "$homedir"
    sudo chmod 755 "$homedir"
}

# Main execution
if [[ $EUID -eq 0 ]]; then
    setup_research_environment
    echo "AI Research Lab NIS setup completed!"
else
    echo "Please run this script as root"
    exit 1
fi
```

### Scenario 2: Course Environment Setup

```bash
#!/bin/bash
# course_nis_setup.sh - NIS setup for academic courses

SEMESTER="fall2025"
DOMAIN="course.ailab.local"

setup_course_environment() {
    local course_code="$1"
    local instructor="$2"
    local student_count="$3"
    
    echo "Setting up course environment for $course_code"
    
    # Create course-specific group
    local group_name="${course_code,,}_students"  # Lowercase course code
    local gid=$((9000 + $(echo "$course_code" | tr -cd '0-9' | head -c 3)))
    
    sudo groupadd -g "$gid" "$group_name"
    
    # Create student accounts for course
    for i in $(seq 1 "$student_count"); do
        local username="${course_code,,}_student$(printf "%02d" $i)"
        local uid=$((10000 + gid + i))
        
        sudo useradd -u "$uid" -g "$group_name" \
                     -c "Student for $course_code" \
                     -d "/home/courses/$course_code/$username" \
                     -s /bin/bash "$username"
        
        # Set course password
        echo "$username:Course123!" | sudo chpasswd
        sudo passwd -e "$username"
        
        # Create course directory structure
        setup_student_course_env "$course_code" "$username"
    done
    
    # Create course-specific NIS maps
    create_course_maps "$course_code" "$instructor" "$student_count"
}

setup_student_course_env() {
    local course_code="$1"
    local username="$2"
    local homedir="/home/courses/$course_code/$username"
    
    # Create directory structure
    sudo mkdir -p "$homedir"/{assignments,projects,data,submissions}
    
    # Copy course materials
    if [ -d "/course_materials/$course_code" ]; then
        sudo cp -r "/course_materials/$course_code"/* "$homedir/assignments/"
    fi
    
    # Set up course-specific environment
    sudo tee "$homedir/.bashrc" << EOF
# Course environment for $course_code
export COURSE_HOME="$homedir"
export PYTHONPATH="\$PYTHONPATH:\$COURSE_HOME/lib"

# Course aliases
alias submit='cp \$1 \$COURSE_HOME/submissions/'
alias check-data='ls -la \$COURSE_HOME/data/'

# Load course modules
module load python/3.11
module load jupyter/4.0
EOF
    
    # Set ownership and permissions
    sudo chown -R "$username:${course_code,,}_students" "$homedir"
    sudo chmod 755 "$homedir"
    sudo chmod 777 "$homedir/submissions"  # For assignment submission
}

create_course_maps() {
    local course_code="$1"
    local instructor="$2"
    local student_count="$3"
    
    # Course enrollment map
    local enrollment_file="/etc/course_enrollment_${course_code,,}"
    sudo tee "$enrollment_file" << EOF
# Course: $course_code
# Instructor: $instructor
# Students: $student_count
# Semester: $SEMESTER
EOF

    # Add student entries
    for i in $(seq 1 "$student_count"); do
        local username="${course_code,,}_student$(printf "%02d" $i)"
        echo "$username:enrolled:$(date +%Y-%m-%d)" | sudo tee -a "$enrollment_file"
    done
    
    # Update Makefile
    sudo tee -a /var/yp/Makefile << EOF

# Course enrollment map for $course_code
course_enrollment_${course_code,,}.byname: $enrollment_file
	@echo "Updating \$@..."
	\$(MAKEDBM) $enrollment_file \$(YPMAPDIR)/\$@
EOF

    # Rebuild maps
    cd /var/yp
    sudo make "course_enrollment_${course_code,,}.byname"
}

# Set up multiple courses
setup_course_environment "CS591" "prof_smith" 30
setup_course_environment "CS592" "prof_jones" 25
setup_course_environment "CS690" "prof_brown" 15

echo "Course environments setup completed!"
```

### Scenario 3: Dynamic User Provisioning

```bash
#!/bin/bash
# dynamic_user_provisioning.sh - Dynamic NIS user management

API_ENDPOINT="https://registry.ailab.local/api/users"
NIS_DOMAIN="ailab.local"
LOG_FILE="/var/log/nis_provisioning.log"

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

# Fetch user changes from central registry
fetch_user_changes() {
    local last_sync_file="/var/lib/nis/last_sync"
    local last_sync="1970-01-01T00:00:00Z"
    
    if [ -f "$last_sync_file" ]; then
        last_sync=$(cat "$last_sync_file")
    fi
    
    # Fetch changes since last sync
    curl -s "$API_ENDPOINT/changes?since=$last_sync" > /tmp/user_changes.json
    
    if [ $? -eq 0 ]; then
        echo "$(date -u +%Y-%m-%dT%H:%M:%SZ)" > "$last_sync_file"
        return 0
    else
        log_message "ERROR: Failed to fetch user changes"
        return 1
    fi
}

# Process user additions
process_user_additions() {
    jq -r '.additions[] | @base64' /tmp/user_changes.json | while read encoded_user; do
        local user_data=$(echo "$encoded_user" | base64 -d)
        
        local username=$(echo "$user_data" | jq -r '.username')
        local uid=$(echo "$user_data" | jq -r '.uid')
        local gid=$(echo "$user_data" | jq -r '.gid')
        local fullname=$(echo "$user_data" | jq -r '.fullname')
        local email=$(echo "$user_data" | jq -r '.email')
        local groups=$(echo "$user_data" | jq -r '.groups[]' | tr '\n' ',')
        local role=$(echo "$user_data" | jq -r '.role')
        
        log_message "Adding user: $username ($fullname)"
        
        # Create user account
        sudo useradd -u "$uid" -g "$gid" -c "$fullname" \
                     -d "/home/$role/$username" -s /bin/bash "$username"
        
        # Add to secondary groups
        if [ -n "$groups" ]; then
            sudo usermod -aG "${groups%,}" "$username"
        fi
        
        # Set up role-specific environment
        setup_role_environment "$username" "$role" "$email"
        
        # Generate initial password
        local temp_password=$(openssl rand -base64 12)
        echo "$username:$temp_password" | sudo chpasswd
        sudo passwd -e "$username"  # Force password change
        
        # Send welcome email
        send_welcome_email "$username" "$email" "$temp_password"
        
        log_message "User $username added successfully"
    done
}

# Process user modifications
process_user_modifications() {
    jq -r '.modifications[] | @base64' /tmp/user_changes.json | while read encoded_mod; do
        local mod_data=$(echo "$encoded_mod" | base64 -d)
        
        local username=$(echo "$mod_data" | jq -r '.username')
        local changes=$(echo "$mod_data" | jq -r '.changes | keys[]')
        
        log_message "Modifying user: $username"
        
        echo "$changes" | while read change; do
            local new_value=$(echo "$mod_data" | jq -r ".changes.$change")
            
            case "$change" in
                "fullname")
                    sudo usermod -c "$new_value" "$username"
                    ;;
                "groups")
                    # Reset groups and add new ones
                    sudo usermod -G "$new_value" "$username"
                    ;;
                "shell")
                    sudo usermod -s "$new_value" "$username"
                    ;;
                "status")
                    if [ "$new_value" = "disabled" ]; then
                        sudo usermod -L "$username"  # Lock account
                    else
                        sudo usermod -U "$username"  # Unlock account
                    fi
                    ;;
            esac
        done
        
        log_message "User $username modified successfully"
    done
}

# Process user deletions
process_user_deletions() {
    jq -r '.deletions[]' /tmp/user_changes.json | while read username; do
        log_message "Removing user: $username"
        
        # Archive user data before deletion
        archive_user_data "$username"
        
        # Remove user account
        sudo userdel -r "$username" 2>/dev/null || sudo userdel "$username"
        
        log_message "User $username removed successfully"
    done
}

setup_role_environment() {
    local username="$1"
    local role="$2"
    local email="$3"
    local homedir="/home/$role/$username"
    
    # Create home directory
    sudo mkdir -p "$homedir"
    sudo cp -r /etc/skel/. "$homedir/"
    
    # Role-specific setup
    case "$role" in
        "researcher")
            setup_researcher_env "$homedir" "$username" "$email"
            ;;
        "student")
            setup_student_env "$homedir" "$username"
            ;;
        "faculty")
            setup_faculty_env "$homedir" "$username" "$email"
            ;;
        "visitor")
            setup_visitor_env "$homedir" "$username"
            ;;
    esac
    
    # Set ownership
    sudo chown -R "$username:$username" "$homedir"
    sudo chmod 755 "$homedir"
}

send_welcome_email() {
    local username="$1"
    local email="$2"
    local password="$3"
    
    cat << EOF | mail -s "AI Lab Account Created" "$email"
Welcome to the AI Lab!

Your account has been created with the following details:
Username: $username
Temporary Password: $password

Please log in and change your password immediately.

Resources:
- Home Directory: /home/$role/$username
- Lab Wiki: https://wiki.ailab.local
- Support: support@ailab.local

Best regards,
AI Lab System Administration
EOF
}

# Main provisioning process
main() {
    log_message "Starting dynamic user provisioning"
    
    if fetch_user_changes; then
        process_user_additions
        process_user_modifications
        process_user_deletions
        
        # Rebuild NIS maps
        log_message "Rebuilding NIS maps"
        cd /var/yp
        sudo make
        
        # Clean up
        rm -f /tmp/user_changes.json
        
        log_message "Dynamic user provisioning completed"
    else
        log_message "Failed to fetch user changes, skipping provisioning"
        exit 1
    fi
}

main
```

---

## Lab Exercises

### Exercise 1: Basic NIS Setup
1. Install and configure NIS master server
2. Create test users and groups
3. Set up NIS client and test authentication
4. Verify map propagation and consistency

### Exercise 2: Multi-server NIS Environment
1. Deploy NIS master and slave servers
2. Configure automatic map synchronization
3. Test failover scenarios
4. Implement monitoring and alerting

### Exercise 3: Security Implementation
1. Configure NIS access controls
2. Implement network-based security
3. Set up shadow password protection
4. Create security audit procedures

### Exercise 4: Advanced NIS Features
1. Create custom NIS maps for lab resources
2. Implement multi-domain NIS setup
3. Integrate with external authentication systems
4. Develop automation and management scripts

---

## Quick Reference Commands

```bash
# NIS Server Management
domainname ailab.local              # Set NIS domain
/usr/lib/yp/ypinit -m              # Initialize master server
cd /var/yp && sudo make            # Rebuild NIS maps
systemctl restart ypserv ypbind   # Restart NIS services

# NIS Client Operations
ypwhich                            # Show current NIS server
ypcat passwd.byname                # List all users
ypmatch username passwd.byname     # Get specific user
getent passwd username             # Test NSS integration

# Map Management
ypcat -k mapname                   # List map with keys
yppoll mapname                     # Check map version
ypxfr mapname                      # Transfer map from master

# Troubleshooting
rpcinfo -p                         # Check RPC services
ypbind -debug                      # Debug NIS binding
tail -f /var/log/syslog | grep yp # Monitor NIS logs
```
