# Service Management with systemctl

## Table of Contents
1. [systemd Overview](#systemd-overview)
2. [Service Management Basics](#service-management-basics)
3. [Creating Custom Services](#creating-custom-services)
4. [Service Dependencies](#service-dependencies)
5. [Service Monitoring](#service-monitoring)
6. [Troubleshooting Services](#troubleshooting-services)
7. [Advanced Service Configuration](#advanced-service-configuration)
8. [Practical Scenarios](#practical-scenarios)

---

## systemd Overview

### What is systemd?

systemd is a system and service manager for Linux operating systems. It serves as:
- **Init system**: First process started by kernel (PID 1)
- **Service manager**: Controls system services
- **Resource manager**: Manages system resources
- **Logger**: Centralized logging system

### systemd Components

- **systemctl**: Main command for controlling systemd
- **journalctl**: View and manage logs
- **systemd-analyze**: Analyze system performance
- **systemd units**: Configuration files for services, sockets, devices, etc.

### Unit Types

| Type | Extension | Purpose |
|------|-----------|---------|
| Service | .service | System services and applications |
| Socket | .socket | IPC sockets and network sockets |
| Target | .target | Group of units (similar to runlevels) |
| Mount | .mount | Filesystem mount points |
| Timer | .timer | Scheduled tasks (like cron) |
| Device | .device | Hardware devices |
| Swap | .swap | Swap files or partitions |

---

## Service Management Basics

### Viewing Services

```bash
# List all services
systemctl list-units --type=service

# List all services (including inactive)
systemctl list-units --type=service --all

# List only active services
systemctl list-units --type=service --state=active

# List failed services
systemctl list-units --type=service --state=failed

# Show service status
systemctl status nginx
systemctl status apache2.service  # .service extension optional

# Check if service is active
systemctl is-active nginx
systemctl is-enabled nginx
systemctl is-failed nginx
```

### Starting and Stopping Services

```bash
# Start a service
sudo systemctl start nginx

# Stop a service
sudo systemctl stop nginx

# Restart a service
sudo systemctl restart nginx

# Reload service configuration (without stopping)
sudo systemctl reload nginx

# Restart if running, start if not running
sudo systemctl try-restart nginx

# Reload configuration or restart if reload not supported
sudo systemctl reload-or-restart nginx
```

### Enabling and Disabling Services

```bash
# Enable service (start at boot)
sudo systemctl enable nginx

# Disable service (don't start at boot)
sudo systemctl disable nginx

# Enable and start service immediately
sudo systemctl enable --now nginx

# Disable and stop service immediately
sudo systemctl disable --now nginx

# Mask service (prevent it from being started)
sudo systemctl mask nginx

# Unmask service
sudo systemctl unmask nginx
```

### Service Information

```bash
# Show detailed service information
systemctl show nginx

# Show specific properties
systemctl show nginx -p ActiveState,SubState,ExecStart

# Show service dependencies
systemctl list-dependencies nginx

# Show reverse dependencies (what depends on this service)
systemctl list-dependencies --reverse nginx

# Show service file location
systemctl show nginx -p FragmentPath

# View service configuration file
systemctl cat nginx
```

---

## Creating Custom Services

### Basic Service Unit Structure

```ini
[Unit]
Description=My Custom Application
Documentation=https://example.com/docs
After=network.target
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=NODE_ENV=production
EnvironmentFile=-/etc/default/myapp

[Install]
WantedBy=multi-user.target
```

### Service Types

```bash
# Type=simple (default)
# Main process doesn't fork
[Service]
Type=simple
ExecStart=/usr/bin/myapp

# Type=forking
# Main process forks and parent exits
[Service]
Type=forking
ExecStart=/usr/bin/myapp --daemon
PIDFile=/var/run/myapp.pid

# Type=oneshot
# Process runs and exits (like a script)
[Service]
Type=oneshot
ExecStart=/usr/local/bin/setup-script.sh
RemainAfterExit=yes

# Type=notify
# Service notifies systemd when ready
[Service]
Type=notify
ExecStart=/usr/bin/myapp
NotifyAccess=main

# Type=idle
# Delays execution until other services start
[Service]
Type=idle
ExecStart=/usr/bin/myapp
```

### Creating a Custom Service

```bash
# Example: Creating a Python web application service

# 1. Create service file
sudo tee /etc/systemd/system/webapp.service << EOF
[Unit]
Description=Python Web Application
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=webapp
Group=webapp
WorkingDirectory=/opt/webapp
Environment=PYTHONPATH=/opt/webapp
ExecStart=/opt/webapp/venv/bin/python app.py
ExecReload=/bin/kill -HUP \$MAINPID
Restart=on-failure
RestartSec=3
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# 2. Create user and directories
sudo useradd -r -s /bin/false webapp
sudo mkdir -p /opt/webapp
sudo chown webapp:webapp /opt/webapp

# 3. Reload systemd and enable service
sudo systemctl daemon-reload
sudo systemctl enable webapp.service
sudo systemctl start webapp.service

# 4. Check status
systemctl status webapp.service
```

### Environment Variables and Configuration

```bash
# Method 1: Environment directive in service file
[Service]
Environment=NODE_ENV=production
Environment="DATABASE_URL=postgresql://user:pass@localhost/db"

# Method 2: EnvironmentFile
[Service]
EnvironmentFile=/etc/default/myapp
EnvironmentFile=-/etc/default/myapp-local  # - prefix means optional

# /etc/default/myapp file format:
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@localhost/db
LOG_LEVEL=info

# Method 3: Multiple environment files
[Service]
EnvironmentFile=/etc/default/common
EnvironmentFile=/etc/default/myapp
```

---

## Service Dependencies

### Dependency Types

```bash
# Requires: Hard dependency (service fails if dependency fails)
[Unit]
Requires=postgresql.service

# Wants: Soft dependency (service continues if dependency fails)
[Unit]
Wants=network-online.target

# After: Start after specified units
[Unit]
After=network.target postgresql.service

# Before: Start before specified units
[Unit]
Before=nginx.service

# Conflicts: Cannot run with specified units
[Unit]
Conflicts=apache2.service

# BindsTo: Similar to Requires but also stops if dependency stops
[Unit]
BindsTo=postgresql.service

# PartOf: When dependency is restarted, this unit is also restarted
[Unit]
PartOf=postgresql.service
```

### Common Dependency Patterns

```bash
# Web application depending on database
[Unit]
Description=Web Application
After=network.target postgresql.service
Wants=network-online.target
Requires=postgresql.service

# Service that needs network
[Unit]
Description=Network Service
After=network.target network-online.target
Wants=network-online.target

# Service with multiple optional dependencies
[Unit]
Description=Application with optional features
After=network.target redis.service memcached.service
Wants=redis.service memcached.service
```

### Systemd Targets

```bash
# Common targets
systemctl list-units --type=target

# Key targets:
# - multi-user.target: Normal multi-user system
# - graphical.target: GUI system
# - network.target: Basic networking
# - network-online.target: Full network connectivity

# Create custom target
sudo tee /etc/systemd/system/webapp-stack.target << EOF
[Unit]
Description=Web Application Stack
Wants=postgresql.service redis.service nginx.service webapp.service
After=postgresql.service redis.service

[Install]
WantedBy=multi-user.target
EOF

# Enable custom target
sudo systemctl enable webapp-stack.target
```

---

## Service Monitoring

### Real-time Monitoring

```bash
# Monitor service status continuously
watch systemctl status nginx

# Follow service logs in real-time
journalctl -u nginx -f

# Monitor multiple services
systemctl status nginx apache2 mysql

# System-wide service overview
systemctl list-units --type=service --state=running
```

### Service Performance Analysis

```bash
# System boot analysis
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain

# Service-specific analysis
systemd-analyze critical-chain nginx.service

# Plot boot timeline (creates SVG file)
systemd-analyze plot > boot-analysis.svg

# Verify service files
systemd-analyze verify /etc/systemd/system/myapp.service
```

### Service Monitoring Script

```bash
#!/bin/bash
# service_monitor.sh - Monitor critical services

SERVICES=("nginx" "postgresql" "redis" "webapp")
LOG_FILE="/var/log/service_monitor.log"
EMAIL="admin@example.com"

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

check_service() {
    local service="$1"
    
    if systemctl is-active --quiet "$service"; then
        log_message "✓ $service is running"
        return 0
    else
        log_message "✗ $service is not running"
        
        # Try to restart the service
        log_message "Attempting to restart $service"
        if systemctl restart "$service"; then
            log_message "✓ Successfully restarted $service"
            echo "Service $service was down but has been restarted" | \
                mail -s "Service Alert: $service restarted" "$EMAIL"
        else
            log_message "✗ Failed to restart $service"
            echo "CRITICAL: Unable to restart $service" | \
                mail -s "CRITICAL: Service $service failed" "$EMAIL"
            return 1
        fi
    fi
}

# Check all services
log_message "Starting service health check"
failed_services=0

for service in "${SERVICES[@]}"; do
    if ! check_service "$service"; then
        ((failed_services++))
    fi
done

if [ "$failed_services" -gt 0 ]; then
    log_message "Service check completed with $failed_services failures"
    exit 1
else
    log_message "All services are healthy"
    exit 0
fi
```

### Automated Service Recovery

```bash
#!/bin/bash
# service_recovery.sh - Automated service recovery with escalation

SERVICE="$1"
MAX_RESTART_ATTEMPTS=3
RESTART_DELAY=30

if [ -z "$SERVICE" ]; then
    echo "Usage: $0 <service-name>"
    exit 1
fi

restart_count=0
while [ $restart_count -lt $MAX_RESTART_ATTEMPTS ]; do
    if systemctl is-active --quiet "$SERVICE"; then
        echo "Service $SERVICE is running"
        exit 0
    fi
    
    ((restart_count++))
    echo "Attempt $restart_count: Restarting $SERVICE"
    
    systemctl restart "$SERVICE"
    sleep $RESTART_DELAY
    
    if systemctl is-active --quiet "$SERVICE"; then
        echo "Service $SERVICE restarted successfully"
        exit 0
    fi
done

# If we get here, all restart attempts failed
echo "CRITICAL: Unable to restart $SERVICE after $MAX_RESTART_ATTEMPTS attempts"

# Escalation procedures
echo "Service $SERVICE failed to start after $MAX_RESTART_ATTEMPTS attempts" | \
    mail -s "CRITICAL: Service Failure" admin@example.com

# Log detailed information
journalctl -u "$SERVICE" --since "1 hour ago" > "/tmp/${SERVICE}_failure.log"

# Could also trigger additional recovery procedures here
# systemctl reboot  # Last resort
```

---

## Troubleshooting Services

### Common Service Issues

#### Service Won't Start

```bash
# Check service status and logs
systemctl status myservice
journalctl -u myservice -n 50

# Check service file syntax
systemd-analyze verify /etc/systemd/system/myservice.service

# Check permissions
ls -la /etc/systemd/system/myservice.service
sudo -u myservice-user /path/to/executable  # Test as service user

# Check dependencies
systemctl list-dependencies myservice
```

#### Service Keeps Restarting

```bash
# Check restart configuration
systemctl show myservice -p Restart,RestartSec

# View recent restarts
journalctl -u myservice --since "1 hour ago"

# Temporarily disable restart to debug
sudo systemctl edit myservice
# Add:
[Service]
Restart=no
```

#### Performance Issues

```bash
# Check resource usage
systemctl status myservice
journalctl -u myservice -o json | jq '.MESSAGE'

# Monitor resource consumption
systemd-cgtop

# Check service limits
systemctl show myservice -p LimitCPU,LimitMEMORY,LimitNOFILE
```

### Debugging Techniques

```bash
# Run service manually for debugging
sudo -u service-user /path/to/service --foreground --debug

# Increase logging verbosity
sudo systemctl edit myservice
# Add:
[Service]
Environment=LOG_LEVEL=debug

# Enable debug logging for systemd
sudo systemctl edit myservice
# Add:
[Service]
StandardOutput=journal
StandardError=journal
LogLevelMax=debug

# Use strace to trace system calls
strace -f -o /tmp/service.trace systemctl start myservice
```

### Service Troubleshooting Script

```bash
#!/bin/bash
# service_debug.sh - Comprehensive service debugging

SERVICE="$1"
if [ -z "$SERVICE" ]; then
    echo "Usage: $0 <service-name>"
    exit 1
fi

echo "=== Service Debugging Report for $SERVICE ==="
echo "Generated: $(date)"
echo

# Service status
echo "1. Service Status:"
systemctl status "$SERVICE" --no-pager
echo

# Service configuration
echo "2. Service Configuration:"
systemctl cat "$SERVICE"
echo

# Dependencies
echo "3. Dependencies:"
echo "Required by:"
systemctl list-dependencies --reverse "$SERVICE" --no-pager
echo "Requires:"
systemctl list-dependencies "$SERVICE" --no-pager
echo

# Recent logs
echo "4. Recent Logs (last 20 lines):"
journalctl -u "$SERVICE" -n 20 --no-pager
echo

# Resource usage
echo "5. Resource Information:"
systemctl show "$SERVICE" -p CPUUsageNSec,MemoryCurrent,TasksCurrent
echo

# File permissions and locations
echo "6. File Permissions:"
SERVICE_FILE=$(systemctl show "$SERVICE" -p FragmentPath --value)
if [ -n "$SERVICE_FILE" ]; then
    ls -la "$SERVICE_FILE"
    
    # Check ExecStart permissions
    EXEC_START=$(systemctl show "$SERVICE" -p ExecStart --value)
    if [ -n "$EXEC_START" ]; then
        EXEC_PATH=$(echo "$EXEC_START" | grep -o '/[^[:space:]]*' | head -1)
        if [ -f "$EXEC_PATH" ]; then
            echo "Executable: $EXEC_PATH"
            ls -la "$EXEC_PATH"
        fi
    fi
fi
echo

# Environment
echo "7. Environment Variables:"
systemctl show "$SERVICE" -p Environment,EnvironmentFiles
echo

echo "=== End of Debug Report ==="
```

---

## Advanced Service Configuration

### Resource Limits

```bash
# CPU and memory limits
[Service]
CPUQuota=50%              # Limit to 50% of one CPU core
MemoryLimit=512M          # Limit memory to 512MB
MemoryMax=1G             # Hard memory limit (systemd v230+)
TasksMax=100             # Limit number of processes/threads

# File descriptor limits
LimitNOFILE=65536        # Maximum open files
LimitNPROC=32768         # Maximum processes

# Core dump settings
LimitCORE=infinity       # Allow core dumps
```

### Security Settings

```bash
# User and group isolation
[Service]
User=myapp
Group=myapp
DynamicUser=yes          # Create temporary user

# Filesystem protection
ProtectSystem=strict     # Read-only /usr, /boot, /efi
ProtectHome=yes          # Hide /home directories
ReadOnlyPaths=/etc       # Make specific paths read-only
ReadWritePaths=/var/myapp # Exception for read-only paths

# Network restrictions
PrivateNetwork=yes       # Disable network access
IPAddressAllow=192.168.1.0/24  # Allow specific IP ranges
IPAddressDeny=any        # Deny all IPs (use with Allow)

# Capability restrictions
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Namespace isolation
PrivateTmp=yes           # Private /tmp
PrivateDevices=yes       # Limited /dev
ProtectKernelTunables=yes # Protect /proc/sys, /sys
ProtectControlGroups=yes  # Protect cgroup filesystem
```

### Advanced Restart Policies

```bash
[Service]
Restart=on-failure       # Restart on failure only
RestartSec=5            # Wait 5 seconds before restart
StartLimitInterval=300   # Time window for start limit
StartLimitBurst=3       # Max start attempts in time window
RestartPreventExitStatus=23 24  # Don't restart on these exit codes
```

### Service Templates

```bash
# Create template: myapp@.service
[Unit]
Description=My App Instance %i
After=network.target

[Service]
Type=simple
ExecStart=/opt/myapp/bin/myapp --instance=%i --config=/etc/myapp/%i.conf
User=myapp
Group=myapp

[Install]
WantedBy=multi-user.target

# Use template for multiple instances
sudo systemctl enable myapp@web.service
sudo systemctl enable myapp@api.service
sudo systemctl start myapp@web.service
sudo systemctl start myapp@api.service
```

---

## Practical Scenarios

### Scenario 1: High-Availability Web Application

```bash
#!/bin/bash
# setup_ha_webapp.sh

# Create main application service
sudo tee /etc/systemd/system/webapp-main.service << EOF
[Unit]
Description=Web Application Main Instance
After=network.target postgresql.service redis.service
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=simple
User=webapp
Group=webapp
WorkingDirectory=/opt/webapp
ExecStart=/opt/webapp/venv/bin/gunicorn -c gunicorn.conf.py app:application
ExecReload=/bin/kill -HUP \$MAINPID
Restart=always
RestartSec=3
KillMode=mixed
TimeoutStopSec=30

# Resource limits
MemoryMax=1G
CPUQuota=200%

# Security
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/opt/webapp/logs /opt/webapp/uploads
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
EOF

# Create backup instance
sudo tee /etc/systemd/system/webapp-backup.service << EOF
[Unit]
Description=Web Application Backup Instance
After=webapp-main.service
BindsTo=webapp-main.service

[Service]
Type=simple
User=webapp
Group=webapp
WorkingDirectory=/opt/webapp
ExecStart=/opt/webapp/venv/bin/gunicorn -c gunicorn-backup.conf.py app:application
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Create monitoring service
sudo tee /etc/systemd/system/webapp-monitor.service << EOF
[Unit]
Description=Web Application Monitor
After=webapp-main.service webapp-backup.service
Wants=webapp-main.service webapp-backup.service

[Service]
Type=simple
ExecStart=/opt/webapp/scripts/monitor.py
Restart=always
RestartSec=30
User=webapp

[Install]
WantedBy=multi-user.target
EOF

# Enable and start services
sudo systemctl daemon-reload
sudo systemctl enable webapp-main.service
sudo systemctl enable webapp-backup.service  
sudo systemctl enable webapp-monitor.service
sudo systemctl start webapp-main.service
```

### Scenario 2: Database Backup Service

```bash
# Create database backup service
sudo tee /etc/systemd/system/db-backup.service << EOF
[Unit]
Description=Database Backup Service
Requires=postgresql.service
After=postgresql.service

[Service]
Type=oneshot
User=postgres
Group=postgres
ExecStart=/usr/local/bin/backup-database.sh
StandardOutput=journal
StandardError=journal
EOF

# Create backup timer (runs daily at 2 AM)
sudo tee /etc/systemd/system/db-backup.timer << EOF
[Unit]
Description=Daily Database Backup
Requires=db-backup.service

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
EOF

# Create backup script
sudo tee /usr/local/bin/backup-database.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/backup/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="myapp"

mkdir -p "$BACKUP_DIR"

# Create backup
pg_dump "$DB_NAME" | gzip > "$BACKUP_DIR/backup_${DB_NAME}_${DATE}.sql.gz"

# Cleanup old backups (keep 30 days)
find "$BACKUP_DIR" -name "backup_*.sql.gz" -mtime +30 -delete

echo "Backup completed: backup_${DB_NAME}_${DATE}.sql.gz"
EOF

sudo chmod +x /usr/local/bin/backup-database.sh

# Enable timer
sudo systemctl daemon-reload
sudo systemctl enable db-backup.timer
sudo systemctl start db-backup.timer

# Check timer status
systemctl list-timers db-backup.timer
```

### Scenario 3: Service Health Check and Recovery

```bash
# Create health check service
sudo tee /etc/systemd/system/health-checker.service << EOF
[Unit]
Description=Service Health Checker
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/local/bin/health-checker.py
Restart=always
RestartSec=60
User=monitor
Group=monitor

[Install]
WantedBy=multi-user.target
EOF

# Create health checker script
sudo tee /usr/local/bin/health-checker.py << 'EOF'
#!/usr/bin/env python3
import subprocess
import time
import smtplib
from email.mime.text import MIMEText
import logging

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')
logger = logging.getLogger(__name__)

SERVICES_TO_MONITOR = ['nginx', 'postgresql', 'redis', 'webapp-main']
CHECK_INTERVAL = 300  # 5 minutes
EMAIL_ALERTS = True
SMTP_SERVER = 'localhost'
ADMIN_EMAIL = 'admin@example.com'

def send_alert(subject, message):
    if not EMAIL_ALERTS:
        return
    
    try:
        msg = MIMEText(message)
        msg['Subject'] = subject
        msg['From'] = 'monitor@example.com'
        msg['To'] = ADMIN_EMAIL
        
        with smtplib.SMTP(SMTP_SERVER) as server:
            server.send_message(msg)
        logger.info(f"Alert sent: {subject}")
    except Exception as e:
        logger.error(f"Failed to send alert: {e}")

def check_service(service_name):
    try:
        result = subprocess.run(
            ['systemctl', 'is-active', service_name],
            capture_output=True, text=True
        )
        return result.returncode == 0
    except Exception as e:
        logger.error(f"Error checking {service_name}: {e}")
        return False

def restart_service(service_name):
    try:
        subprocess.run(['systemctl', 'restart', service_name], check=True)
        logger.info(f"Successfully restarted {service_name}")
        return True
    except subprocess.CalledProcessError as e:
        logger.error(f"Failed to restart {service_name}: {e}")
        return False

def main():
    logger.info("Health checker started")
    
    while True:
        for service in SERVICES_TO_MONITOR:
            if not check_service(service):
                logger.warning(f"Service {service} is not running")
                
                if restart_service(service):
                    send_alert(
                        f"Service Alert: {service} restarted",
                        f"Service {service} was down but has been restarted successfully."
                    )
                else:
                    send_alert(
                        f"CRITICAL: Service {service} failed to restart",
                        f"Service {service} is down and failed to restart. Manual intervention required."
                    )
            else:
                logger.info(f"Service {service} is healthy")
        
        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    main()
EOF

sudo chmod +x /usr/local/bin/health-checker.py

# Create monitor user
sudo useradd -r -s /bin/false monitor

# Enable health checker
sudo systemctl daemon-reload
sudo systemctl enable health-checker.service
sudo systemctl start health-checker.service
```

---

## Lab Exercises

### Exercise 1: Custom Service Creation
1. Create a custom Python/Node.js service
2. Configure proper user isolation and security settings
3. Set up appropriate dependencies and restart policies
4. Test service startup, shutdown, and restart scenarios

### Exercise 2: Service Dependencies and Targets
1. Create a multi-service application stack
2. Configure proper dependency relationships
3. Create a custom target for the entire stack
4. Test dependency resolution and startup order

### Exercise 3: Service Monitoring and Recovery
1. Implement automated service health checking
2. Create alerting mechanisms for service failures
3. Set up automatic recovery procedures
4. Test failure scenarios and recovery processes

### Exercise 4: Performance and Security Hardening
1. Configure resource limits for services
2. Implement security restrictions and sandboxing
3. Set up service templates for scalability
4. Monitor and optimize service performance

---

## Quick Reference Commands

```bash
# Service Control
systemctl start/stop/restart/reload service  # Control service
systemctl enable/disable service             # Boot-time behavior
systemctl status service                     # Service status
systemctl is-active/enabled/failed service   # Quick status check

# Service Information
systemctl list-units --type=service         # List services
systemctl show service                       # Detailed info
systemctl cat service                        # View service file
systemctl list-dependencies service          # Show dependencies

# Configuration Management
sudo systemctl daemon-reload                # Reload systemd config
sudo systemctl edit service                 # Create override file
systemd-analyze verify service-file         # Validate service file

# Monitoring and Logs
journalctl -u service -f                    # Follow service logs
systemctl list-timers                       # Show active timers
systemd-analyze blame                       # Boot time analysis
systemd-cgtop                              # Resource usage by service
```
