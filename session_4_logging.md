# System Logging and Monitoring

## Table of Contents
1. [Linux Logging System Overview](#linux-logging-system-overview)
2. [Systemd Journal (journalctl)](#systemd-journal-journalctl)
3. [Traditional Log Files](#traditional-log-files)
4. [Log Rotation and Management](#log-rotation-and-management)
5. [Remote Logging](#remote-logging)
6. [Log Analysis and Monitoring](#log-analysis-and-monitoring)
7. [Performance Monitoring](#performance-monitoring)
8. [Practical Scenarios](#practical-scenarios)

---

## Linux Logging System Overview

### Logging Architecture

Modern Linux systems use a dual logging approach:
1. **systemd Journal**: Binary log storage with journalctl interface
2. **Traditional Syslog**: Text-based logs in /var/log/

### Log Levels (Severity)

| Level | Numeric | Keyword | Description |
|-------|---------|---------|-------------|
| 0 | emerg | System unusable |
| 1 | alert | Action must be taken immediately |
| 2 | crit | Critical conditions |
| 3 | err | Error conditions |
| 4 | warning | Warning conditions |
| 5 | notice | Normal but significant |
| 6 | info | Informational messages |
| 7 | debug | Debug-level messages |

### Facilities

Common log facilities:
- **kern**: Kernel messages
- **user**: User-level messages  
- **mail**: Mail system
- **daemon**: System daemons
- **auth**: Authorization messages
- **syslog**: Syslog daemon messages
- **local0-local7**: Custom facilities

---

## Systemd Journal (journalctl)

### Basic journalctl Usage

```bash
# View all journal entries
journalctl

# View logs from current boot
journalctl -b

# View logs from previous boot
journalctl -b -1

# Follow logs in real-time (like tail -f)
journalctl -f

# Show last N lines
journalctl -n 50

# Show logs since specific time
journalctl --since "2024-01-01 10:00:00"
journalctl --since "1 hour ago"
journalctl --since yesterday
journalctl --since "2 days ago"

# Show logs until specific time
journalctl --until "2024-01-01 15:00:00"

# Time range
journalctl --since "2024-01-01" --until "2024-01-02"
```

### Service-Specific Logs

```bash
# View logs for specific service
journalctl -u nginx
journalctl -u sshd.service

# Multiple services
journalctl -u nginx -u apache2

# Follow service logs
journalctl -u nginx -f

# Service logs with specific priority
journalctl -u nginx -p err

# Show logs for service since last boot
journalctl -u nginx -b
```

### Advanced Filtering

```bash
# Filter by priority
journalctl -p err                    # Errors and higher
journalctl -p warning..emerg         # Warning to emergency
journalctl -p 3                      # Numeric priority

# Filter by facility
journalctl SYSLOG_FACILITY=0         # Kernel messages

# Filter by process ID
journalctl _PID=1234

# Filter by user
journalctl _UID=1000
journalctl _UID=1000 -f

# Filter by executable
journalctl /usr/bin/ssh

# Filter by systemd unit
journalctl _SYSTEMD_UNIT=nginx.service

# Multiple filters (AND operation)
journalctl _SYSTEMD_UNIT=nginx.service -p err

# Field-based filtering
journalctl MESSAGE_ID=39f53479d3a045ac8e11786248231fbf
```

### Journal Configuration

```bash
# View journal configuration
journalctl --show

# Journal disk usage
journalctl --disk-usage

# Verify journal integrity
journalctl --verify

# Vacuum old entries
sudo journalctl --vacuum-time=7d     # Keep 7 days
sudo journalctl --vacuum-size=1G     # Keep 1GB max
sudo journalctl --vacuum-files=5     # Keep 5 files max
```

#### /etc/systemd/journald.conf

```ini
[Journal]
# Storage location
Storage=persistent              # persistent, volatile, auto, none

# Size limits
SystemMaxUse=1G                # Maximum disk usage
SystemKeepFree=15%             # Keep 15% of filesystem free
SystemMaxFileSize=128M         # Max size per journal file
SystemMaxFiles=100             # Max number of journal files

# Retention
MaxRetentionSec=1month         # Keep logs for 1 month
MaxFileSec=1week              # Rotate weekly

# Rate limiting
RateLimitInterval=30s          # Time window
RateLimitBurst=1000           # Max messages per window

# Forward to syslog
ForwardToSyslog=yes           # Forward to traditional syslog
ForwardToWall=yes             # Forward to wall
```

### Journal Analysis

```bash
# Show journal statistics
journalctl --header

# Boot information
journalctl --list-boots

# System boot analysis
journalctl -b --no-pager | grep -i error
journalctl -b --no-pager | grep -i failed

# Export journal data
journalctl --output=json > journal-export.json
journalctl --output=json-pretty -n 10

# Different output formats
journalctl --output=short      # Default format
journalctl --output=verbose    # All available data
journalctl --output=cat        # Only message text
journalctl --output=json       # JSON format
```

---

## Traditional Log Files

### Important Log Files

```bash
# System logs
/var/log/syslog              # General system messages (Debian/Ubuntu)
/var/log/messages            # General system messages (RHEL/CentOS)
/var/log/kern.log            # Kernel messages
/var/log/dmesg               # Kernel ring buffer

# Authentication
/var/log/auth.log            # Authentication logs (Debian/Ubuntu)
/var/log/secure              # Authentication logs (RHEL/CentOS)
/var/log/faillog             # Failed login attempts
/var/log/lastlog             # Last login information

# Application logs
/var/log/apache2/            # Apache web server
/var/log/nginx/              # Nginx web server
/var/log/mysql/              # MySQL database
/var/log/postgresql/         # PostgreSQL database

# System events
/var/log/cron                # Cron job logs
/var/log/mail.log            # Mail server logs
/var/log/daemon.log          # System daemon logs
```

### Syslog Configuration

#### rsyslog (/etc/rsyslog.conf)

```bash
# Basic rsyslog configuration
# Modules
module(load="imuxsock")        # Unix socket input
module(load="imklog")          # Kernel logging

# Rules (facility.priority destination)
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                  /var/log/secure
mail.*                                      /var/log/maillog
cron.*                                      /var/log/cron
*.emerg                                     :omusrmsg:*
uucp,news.crit                             /var/log/spooler
local7.*                                    /var/log/boot.log

# Network logging
*.* @@remote-server:514        # UDP forwarding
*.* @@remote-server:514        # TCP forwarding

# Custom application logging
local0.*                       /var/log/myapp.log
```

### Log File Monitoring

```bash
# Real-time log monitoring
tail -f /var/log/syslog
tail -f /var/log/auth.log

# Monitor multiple files
multitail /var/log/syslog /var/log/auth.log

# Follow log with highlighting
tail -f /var/log/auth.log | grep --color=always "Failed\|Accepted"

# Monitor log with timestamps
tail -f /var/log/syslog | while read line; do 
    echo "$(date): $line"
done
```

### Log Analysis Commands

```bash
# Search for patterns
grep "Failed password" /var/log/auth.log
grep -i error /var/log/syslog | head -20

# Count occurrences
grep -c "Failed password" /var/log/auth.log

# Show context around matches
grep -B5 -A5 "error" /var/log/syslog

# Search in compressed logs
zgrep "error" /var/log/syslog.1.gz

# Find large log files
find /var/log -name "*.log" -size +100M

# Analyze log patterns by time
awk '{print $1, $2, $3}' /var/log/auth.log | sort | uniq -c

# Top IP addresses from Apache logs
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -nr | head -10
```

---

## Log Rotation and Management

### logrotate Configuration

#### /etc/logrotate.conf

```bash
# Global settings
weekly                    # Rotate weekly
rotate 4                 # Keep 4 old copies
create                   # Create new log file after rotation
dateext                  # Use date as extension
compress                 # Compress rotated logs
delaycompress           # Compress on next rotation cycle

# Include specific configurations
include /etc/logrotate.d

# System logs
/var/log/wtmp {
    monthly
    create 0664 root utmp
    minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
```

#### Custom Log Rotation (/etc/logrotate.d/myapp)

```bash
/var/log/myapp/*.log {
    daily                   # Rotate daily
    missingok              # Don't error if log file missing
    rotate 30              # Keep 30 days of logs
    compress               # Compress old logs
    delaycompress         # Don't compress current log
    notifempty            # Don't rotate empty files
    create 0644 myapp myapp  # Create new file with permissions
    sharedscripts         # Run scripts only once for multiple files
    
    postrotate            # Commands after rotation
        /usr/bin/systemctl reload myapp > /dev/null 2>&1 || true
    endscript
    
    prerotate             # Commands before rotation
        if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
            run-parts /etc/logrotate.d/httpd-prerotate; \
        fi \
    endscript
}
```

### Manual Log Rotation

```bash
# Force log rotation
sudo logrotate -f /etc/logrotate.conf

# Debug log rotation
sudo logrotate -d /etc/logrotate.conf

# Rotate specific configuration
sudo logrotate /etc/logrotate.d/nginx

# Check logrotate status
cat /var/lib/logrotate/status
```

### Log Management Script

```bash
#!/bin/bash
# log_manager.sh - Comprehensive log management

LOG_DIR="/var/log"
RETENTION_DAYS=30
MAX_SIZE="100M"
EMAIL="admin@example.com"

# Function to check disk usage
check_disk_usage() {
    local threshold=80
    local usage=$(df "$LOG_DIR" | awk 'NR==2 {print int($5)}')
    
    if [ "$usage" -gt "$threshold" ]; then
        echo "WARNING: Log directory is ${usage}% full" | \
            mail -s "Disk Usage Alert" "$EMAIL"
        return 1
    fi
    return 0
}

# Function to clean old logs
clean_old_logs() {
    echo "Cleaning logs older than $RETENTION_DAYS days..."
    find "$LOG_DIR" -name "*.log.*" -mtime +$RETENTION_DAYS -delete
    find "$LOG_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete
}

# Function to find large log files
find_large_logs() {
    echo "Large log files (>$MAX_SIZE):"
    find "$LOG_DIR" -name "*.log" -size +$MAX_SIZE -exec ls -lh {} \; | \
        awk '{print $9 ": " $5}'
}

# Function to analyze log growth
analyze_log_growth() {
    echo "Log growth analysis (last 24 hours):"
    
    for log in $(find "$LOG_DIR" -name "*.log" -mtime -1); do
        size=$(du -h "$log" | cut -f1)
        lines=$(wc -l < "$log" 2>/dev/null || echo "0")
        echo "$log: $size ($lines lines)"
    done
}

# Function to generate log summary
generate_summary() {
    local summary_file="/tmp/log_summary_$(date +%Y%m%d).txt"
    
    cat > "$summary_file" << EOF
Log Management Summary - $(date)

Disk Usage:
$(df -h "$LOG_DIR")

Top 10 Largest Log Files:
$(find "$LOG_DIR" -name "*.log" -exec du -h {} \; | sort -hr | head -10)

Recent Error Summary:
$(grep -i error /var/log/syslog | tail -10)

Recent Authentication Failures:
$(grep "Failed password" /var/log/auth.log | tail -10)
EOF

    echo "Summary generated: $summary_file"
    cat "$summary_file" | mail -s "Daily Log Summary" "$EMAIL"
}

# Main execution
echo "Starting log management - $(date)"

if ! check_disk_usage; then
    clean_old_logs
fi

find_large_logs
analyze_log_growth
generate_summary

echo "Log management completed - $(date)"
```

---

## Remote Logging

### Centralized Logging with rsyslog

#### Server Configuration (/etc/rsyslog.conf)

```bash
# Enable network logging modules
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

# Template for remote logs
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"

# Store remote logs separately
if $fromhost-ip != '127.0.0.1' then ?RemoteLogs
& stop

# Alternative: store by facility
$template DynamicFile,"/var/log/remote/%HOSTNAME%/%syslogfacility-text%.log"
*.* ?DynamicFile
```

#### Client Configuration

```bash
# Forward all logs to remote server
*.* @@log-server.example.com:514    # TCP
*.* @log-server.example.com:514     # UDP

# Forward specific facilities
mail.* @@log-server.example.com:514
auth.* @@log-server.example.com:514

# Forward with specific template
$template MyFormat,"%timegenerated% %HOSTNAME% %syslogtag% %msg%\n"
*.* @@log-server.example.com:514;MyFormat
```

### Secure Remote Logging (TLS)

#### Server Configuration

```bash
# Enable TLS module
module(load="imtcp" StreamDriver.Name="gtls" StreamDriver.Mode="1")
input(type="imtcp" port="6514")

# Certificate configuration
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/ssl/certs/ca.pem"
    DefaultNetstreamDriverCertFile="/etc/ssl/certs/server-cert.pem"
    DefaultNetstreamDriverKeyFile="/etc/ssl/private/server-key.pem"
)
```

#### Client Configuration

```bash
# Configure TLS client
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/ssl/certs/ca.pem"
    DefaultNetstreamDriverCertFile="/etc/ssl/certs/client-cert.pem"  
    DefaultNetstreamDriverKeyFile="/etc/ssl/private/client-key.pem"
)

# Forward with TLS
*.* @@log-server.example.com:6514
```

---

## Log Analysis and Monitoring

### Log Analysis Tools

#### GoAccess (Web Log Analyzer)

```bash
# Install GoAccess
sudo apt install goaccess

# Real-time Apache/Nginx log analysis
goaccess /var/log/apache2/access.log -o /var/www/html/report.html --real-time-html

# Configuration for different log formats
goaccess /var/log/nginx/access.log --log-format=COMBINED -o report.html
```

#### AWK-based Log Analysis

```bash
# Analyze Apache access logs
awk '{
    ip[$1]++; 
    status[$9]++; 
    bytes+=$10
} END {
    print "Top 10 IPs:";
    for (i in ip) print ip[i], i | "sort -nr | head -10";
    close("sort -nr | head -10");
    
    print "\nStatus Codes:";
    for (i in status) print status[i], i;
    
    print "\nTotal Bytes:", bytes;
}' /var/log/apache2/access.log
```

### Log Monitoring Script

```bash
#!/bin/bash
# log_monitor.sh - Real-time log monitoring and alerting

CONFIG_FILE="/etc/log_monitor.conf"
ALERT_EMAIL="admin@example.com"
ALERT_THRESHOLD=5

# Default patterns to monitor
declare -A PATTERNS=(
    ["error"]="error|Error|ERROR"
    ["auth_failure"]="Failed password|authentication failure|invalid user"
    ["disk_full"]="No space left on device|disk full"
    ["segfault"]="segmentation fault|segfault"
    ["oom"]="Out of memory|OOM killer"
)

# Load configuration if exists
if [ -f "$CONFIG_FILE" ]; then
    source "$CONFIG_FILE"
fi

# Function to send alert
send_alert() {
    local pattern="$1"
    local count="$2"
    local log_file="$3"
    
    local subject="Log Alert: $pattern detected"
    local message="Pattern '$pattern' detected $count times in $log_file in the last minute."
    
    echo "$message" | mail -s "$subject" "$ALERT_EMAIL"
    logger "LOG_MONITOR: Alert sent for pattern '$pattern' ($count occurrences)"
}

# Function to monitor log file
monitor_log() {
    local log_file="$1"
    local temp_file="/tmp/log_monitor_$(basename "$log_file").tmp"
    
    if [ ! -f "$log_file" ]; then
        echo "Warning: $log_file not found"
        return
    fi
    
    # Get current position or start from end
    if [ -f "$temp_file" ]; then
        last_pos=$(cat "$temp_file")
    else
        last_pos=$(wc -c < "$log_file")
        echo "$last_pos" > "$temp_file"
    fi
    
    # Get new content since last check
    current_pos=$(wc -c < "$log_file")
    if [ "$current_pos" -gt "$last_pos" ]; then
        new_content=$(tail -c +$((last_pos + 1)) "$log_file")
        
        # Check each pattern
        for pattern_name in "${!PATTERNS[@]}"; do
            pattern="${PATTERNS[$pattern_name]}"
            count=$(echo "$new_content" | grep -c -E "$pattern")
            
            if [ "$count" -ge "$ALERT_THRESHOLD" ]; then
                send_alert "$pattern_name" "$count" "$log_file"
            fi
        done
        
        echo "$current_pos" > "$temp_file"
    fi
}

# Main monitoring loop
main() {
    local log_files=(
        "/var/log/syslog"
        "/var/log/auth.log"
        "/var/log/apache2/error.log"
        "/var/log/nginx/error.log"
    )
    
    echo "Starting log monitoring - $(date)"
    
    while true; do
        for log_file in "${log_files[@]}"; do
            monitor_log "$log_file"
        done
        
        sleep 60  # Check every minute
    done
}

# Handle signals
trap 'echo "Log monitoring stopped"; exit 0' SIGTERM SIGINT

main
```

### Security Log Analysis

```bash
#!/bin/bash
# security_log_analysis.sh

AUTH_LOG="/var/log/auth.log"
REPORT_FILE="/tmp/security_report_$(date +%Y%m%d).txt"

{
    echo "Security Log Analysis Report - $(date)"
    echo "========================================"
    echo
    
    # Failed login attempts
    echo "1. Failed Login Attempts (Top 10 IPs):"
    grep "Failed password" "$AUTH_LOG" | \
        awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head -10
    echo
    
    # Successful logins
    echo "2. Successful Logins:"
    grep "Accepted password" "$AUTH_LOG" | \
        awk '{print $9 "@" $(NF-3) " from " $(NF-1)}' | sort | uniq -c
    echo
    
    # Root login attempts
    echo "3. Root Login Attempts:"
    grep "Failed password for root" "$AUTH_LOG" | wc -l | \
        xargs echo "Failed root login attempts:"
    grep "Accepted password for root" "$AUTH_LOG" | wc -l | \
        xargs echo "Successful root logins:"
    echo
    
    # Invalid users
    echo "4. Invalid User Attempts:"
    grep "Invalid user" "$AUTH_LOG" | \
        awk '{print $8}' | sort | uniq -c | sort -nr | head -10
    echo
    
    # SSH key authentication
    echo "5. SSH Key Authentication:"
    grep "Accepted publickey" "$AUTH_LOG" | wc -l | \
        xargs echo "Successful key authentications:"
    echo
    
    # Unusual activities
    echo "6. Unusual Activities:"
    grep -E "(sudo|su):" "$AUTH_LOG" | tail -10
    echo
    
    # Time-based analysis
    echo "7. Login Activity by Hour:"
    grep "Accepted password" "$AUTH_LOG" | \
        awk '{print $3}' | cut -d: -f1 | sort | uniq -c | sort -k2n
    
} > "$REPORT_FILE"

echo "Security analysis completed: $REPORT_FILE"
cat "$REPORT_FILE"
```

---

## Performance Monitoring

### System Resource Monitoring

```bash
# CPU and Memory monitoring
top -b -n1 | head -20
htop  # Interactive process viewer
vmstat 1 5  # Virtual memory statistics
iostat 1 5  # I/O statistics
sar -u 1 5  # System activity reporter

# Disk usage monitoring
df -h
du -sh /var/log/*
lsof +L1  # Find deleted files still held open

# Network monitoring
netstat -tuln
ss -tuln
iftop  # Network bandwidth usage
nload  # Network load monitor
```

### Performance Monitoring Script

```bash
#!/bin/bash
# performance_monitor.sh

LOG_FILE="/var/log/performance_monitor.log"
ALERT_EMAIL="admin@example.com"

# Thresholds
CPU_THRESHOLD=80
MEMORY_THRESHOLD=80
DISK_THRESHOLD=85
LOAD_THRESHOLD=5.0

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

send_alert() {
    local subject="$1"
    local message="$2"
    echo "$message" | mail -s "$subject" "$ALERT_EMAIL"
    log_message "ALERT: $subject"
}

check_cpu() {
    local cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d% -f1)
    cpu_usage=${cpu_usage%.*}  # Remove decimal part
    
    if [ "$cpu_usage" -gt "$CPU_THRESHOLD" ]; then
        send_alert "High CPU Usage" "CPU usage is at ${cpu_usage}%"
    fi
    
    log_message "CPU usage: ${cpu_usage}%"
}

check_memory() {
    local mem_info=$(free | grep Mem:)
    local total=$(echo "$mem_info" | awk '{print $2}')
    local used=$(echo "$mem_info" | awk '{print $3}')
    local mem_usage=$((used * 100 / total))
    
    if [ "$mem_usage" -gt "$MEMORY_THRESHOLD" ]; then
        send_alert "High Memory Usage" "Memory usage is at ${mem_usage}%"
    fi
    
    log_message "Memory usage: ${mem_usage}%"
}

check_disk() {
    while read filesystem size used avail usage mountpoint; do
        if [[ "$usage" =~ ^[0-9]+% ]]; then
            usage_num=${usage%\%}
            if [ "$usage_num" -gt "$DISK_THRESHOLD" ]; then
                send_alert "High Disk Usage" \
                    "Disk usage on $mountpoint is at $usage"
            fi
            log_message "Disk usage $mountpoint: $usage"
        fi
    done < <(df -h | grep -E '^/dev/')
}

check_load() {
    local load_avg=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | tr -d ',')
    
    if (( $(echo "$load_avg > $LOAD_THRESHOLD" | bc -l) )); then
        send_alert "High System Load" "System load average is $load_avg"
    fi
    
    log_message "Load average: $load_avg"
}

check_services() {
    local critical_services=("nginx" "apache2" "mysql" "postgresql" "ssh")
    
    for service in "${critical_services[@]}"; do
        if ! systemctl is-active --quiet "$service" 2>/dev/null; then
            # Service might not exist, check if it's installed
            if systemctl list-unit-files | grep -q "^$service"; then
                send_alert "Service Down" "Service $service is not running"
            fi
        else
            log_message "Service $service: running"
        fi
    done
}

# Main monitoring loop
main() {
    log_message "Starting performance monitoring"
    
    while true; do
        check_cpu
        check_memory  
        check_disk
        check_load
        check_services
        
        log_message "Performance check completed"
        sleep 300  # Check every 5 minutes
    done
}

# Handle signals
trap 'log_message "Performance monitoring stopped"; exit 0' SIGTERM SIGINT

main
```

---

## Practical Scenarios

### Scenario 1: Web Server Log Analysis

```bash
#!/bin/bash
# web_log_analyzer.sh

ACCESS_LOG="/var/log/nginx/access.log"
ERROR_LOG="/var/log/nginx/error.log"
REPORT_DIR="/var/www/html/reports"

mkdir -p "$REPORT_DIR"

# Generate daily report
generate_daily_report() {
    local date=$(date +%Y-%m-%d)
    local report_file="$REPORT_DIR/daily_report_$date.html"
    
    cat > "$report_file" << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Web Server Report - $date</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .error { color: red; }
        .warning { color: orange; }
    </style>
</head>
<body>
    <h1>Web Server Daily Report - $date</h1>
    
    <h2>Traffic Summary</h2>
    <table>
        <tr><th>Metric</th><th>Value</th></tr>
        <tr><td>Total Requests</td><td>$(wc -l < "$ACCESS_LOG")</td></tr>
        <tr><td>Unique Visitors</td><td>$(awk '{print $1}' "$ACCESS_LOG" | sort | uniq | wc -l)</td></tr>
        <tr><td>Total Bandwidth</td><td>$(awk '{sum += $10} END {printf "%.2f MB", sum/1024/1024}' "$ACCESS_LOG")</td></tr>
    </table>
    
    <h2>Top 10 IP Addresses</h2>
    <table>
        <tr><th>Requests</th><th>IP Address</th></tr>
EOF

    awk '{print $1}' "$ACCESS_LOG" | sort | uniq -c | sort -nr | head -10 | \
        while read count ip; do
            echo "        <tr><td>$count</td><td>$ip</td></tr>" >> "$report_file"
        done

    cat >> "$report_file" << EOF
    </table>
    
    <h2>Status Code Distribution</h2>
    <table>
        <tr><th>Status Code</th><th>Count</th><th>Description</th></tr>
EOF

    awk '{print $9}' "$ACCESS_LOG" | sort | uniq -c | sort -nr | \
        while read count status; do
            case $status in
                200) desc="OK" ;;
                301|302) desc="Redirect" ;;
                404) desc="Not Found" ;;
                500) desc="Server Error" ;;
                *) desc="Other" ;;
            esac
            
            if [[ $status == 4* || $status == 5* ]]; then
                echo "        <tr class=\"error\"><td>$status</td><td>$count</td><td>$desc</td></tr>" >> "$report_file"
            else
                echo "        <tr><td>$status</td><td>$count</td><td>$desc</td></tr>" >> "$report_file"
            fi
        done

    cat >> "$report_file" << EOF
    </table>
    
    <h2>Recent Errors</h2>
    <pre>
$(tail -20 "$ERROR_LOG" | sed 's/</\&lt;/g; s/>/\&gt;/g')
    </pre>
    
    <p>Report generated on $(date)</p>
</body>
</html>
EOF

    echo "Report generated: $report_file"
}

generate_daily_report
```

### Scenario 2: Database Performance Monitoring

```bash
#!/bin/bash
# db_monitor.sh - Database performance monitoring

DB_TYPE="postgresql"  # or "mysql"
LOG_FILE="/var/log/db_monitor.log"
SLOW_QUERY_THRESHOLD=5  # seconds

monitor_postgresql() {
    local db_name="$1"
    
    # Check connection count
    local connections=$(sudo -u postgres psql -d "$db_name" -t -c \
        "SELECT count(*) FROM pg_stat_activity;")
    
    # Check slow queries
    local slow_queries=$(sudo -u postgres psql -d "$db_name" -t -c \
        "SELECT count(*) FROM pg_stat_activity WHERE state = 'active' AND query_start < now() - interval '$SLOW_QUERY_THRESHOLD seconds';")
    
    # Check database size
    local db_size=$(sudo -u postgres psql -d "$db_name" -t -c \
        "SELECT pg_size_pretty(pg_database_size('$db_name'));")
    
    log_message "PostgreSQL - DB: $db_name, Connections: $connections, Slow queries: $slow_queries, Size: $db_size"
    
    # Alert on high connections
    if [ "$connections" -gt 80 ]; then
        send_alert "High Database Connections" \
            "PostgreSQL has $connections active connections"
    fi
    
    # Alert on slow queries
    if [ "$slow_queries" -gt 0 ]; then
        send_alert "Slow Database Queries" \
            "PostgreSQL has $slow_queries slow queries running"
    fi
}

monitor_mysql() {
    local mysql_cmd="mysql -u monitor -p$MYSQL_PASSWORD"
    
    # Check connection count
    local connections=$($mysql_cmd -e "SHOW STATUS LIKE 'Threads_connected';" | \
        awk 'NR==2 {print $2}')
    
    # Check slow queries
    local slow_queries=$($mysql_cmd -e "SHOW STATUS LIKE 'Slow_queries';" | \
        awk 'NR==2 {print $2}')
    
    log_message "MySQL - Connections: $connections, Slow queries: $slow_queries"
    
    if [ "$connections" -gt 80 ]; then
        send_alert "High Database Connections" \
            "MySQL has $connections active connections"
    fi
}

# Main execution
case $DB_TYPE in
    "postgresql")
        monitor_postgresql "myapp"
        ;;
    "mysql")
        monitor_mysql
        ;;
esac
```

### Scenario 3: Security Event Correlation

```bash
#!/bin/bash
# security_correlator.sh - Correlate security events from multiple sources

WORK_DIR="/tmp/security_analysis"
REPORT_FILE="$WORK_DIR/security_correlation_$(date +%Y%m%d_%H%M).txt"

mkdir -p "$WORK_DIR"

# Collect events from different sources
collect_events() {
    # SSH brute force attempts
    grep "Failed password" /var/log/auth.log | \
        awk '{print $1" "$2" "$3" SSH_BRUTEFORCE " $(NF-3)}' > "$WORK_DIR/ssh_events.txt"
    
    # Web attacks from Apache/Nginx
    grep -E "(404|403)" /var/log/nginx/access.log | \
        awk '{print $4" "$5" WEB_404 " $1}' | tr -d '[]' > "$WORK_DIR/web_events.txt"
    
    # Firewall blocks
    grep "iptables denied" /var/log/syslog | \
        awk '{print $1" "$2" "$3" FIREWALL_BLOCK"}' > "$WORK_DIR/firewall_events.txt"
    
    # System errors
    grep -i "error" /var/log/syslog | \
        awk '{print $1" "$2" "$3" SYSTEM_ERROR"}' > "$WORK_DIR/system_events.txt"
}

# Correlate events by IP address
correlate_by_ip() {
    local report_section="$1"
    
    # Extract IPs from all events
    cat "$WORK_DIR"/*_events.txt | awk '{print $NF}' | \
        grep -E '^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$' | \
        sort | uniq -c | sort -nr > "$WORK_DIR/ip_frequency.txt"
    
    echo "$report_section" >> "$REPORT_FILE"
    echo "====================" >> "$REPORT_FILE"
    
    while read count ip; do
        if [ "$count" -gt 5 ]; then
            echo "Suspicious IP: $ip (Score: $count)" >> "$REPORT_FILE"
            
            # Show events for this IP
            grep "$ip" "$WORK_DIR"/*_events.txt | head -10 >> "$REPORT_FILE"
            echo >> "$REPORT_FILE"
        fi
    done < "$WORK_DIR/ip_frequency.txt"
}

# Time-based correlation
correlate_by_time() {
    echo "Time-based Event Correlation" >> "$REPORT_FILE"
    echo "=============================" >> "$REPORT_FILE"
    
    # Group events by hour
    cat "$WORK_DIR"/*_events.txt | \
        awk '{print $3}' | cut -d: -f1 | sort | uniq -c | sort -nr | \
        head -5 >> "$REPORT_FILE"
    
    echo >> "$REPORT_FILE"
}

# Generate final report
generate_report() {
    {
        echo "Security Event Correlation Report"
        echo "Generated: $(date)"
        echo "================================="
        echo
        
        echo "Event Summary:"
        echo "SSH Brute Force: $(wc -l < "$WORK_DIR/ssh_events.txt")"
        echo "Web 404s: $(wc -l < "$WORK_DIR/web_events.txt")"
        echo "Firewall Blocks: $(wc -l < "$WORK_DIR/firewall_events.txt")"
        echo "System Errors: $(wc -l < "$WORK_DIR/system_events.txt")"
        echo
        
    } > "$REPORT_FILE"
    
    correlate_by_ip "IP-based Correlation"
    correlate_by_time
    
    echo "Security correlation report: $REPORT_FILE"
}

# Main execution
collect_events
generate_report

# Email report if significant events found
if [ "$(wc -l < "$WORK_DIR/ip_frequency.txt")" -gt 0 ]; then
    mail -s "Security Correlation Report" admin@example.com < "$REPORT_FILE"
fi
```

---

## Lab Exercises

### Exercise 1: Log Configuration and Analysis
1. Configure rsyslog for custom application logging
2. Set up log rotation for application logs
3. Create scripts to analyze log patterns
4. Implement automated log alerting

### Exercise 2: Centralized Logging Setup
1. Configure a central log server with rsyslog
2. Set up multiple clients to forward logs
3. Implement secure log transmission with TLS
4. Create dashboards for log visualization

### Exercise 3: Performance Monitoring System
1. Implement comprehensive system monitoring
2. Create alerting mechanisms for various metrics
3. Set up automated performance reporting
4. Integrate with existing monitoring tools

### Exercise 4: Security Log Analysis
1. Develop security event correlation system
2. Create automated threat detection scripts
3. Implement incident response workflows
4. Generate security compliance reports

---

## Quick Reference Commands

```bash
# Journal Management
journalctl -u service -f                    # Follow service logs
journalctl --since "1 hour ago" -p err     # Recent errors
journalctl --vacuum-time=7d                # Clean old logs
systemctl status service                   # Service status with logs

# Traditional Logs
tail -f /var/log/syslog                    # Follow system log
grep -i error /var/log/syslog              # Find errors
logrotate -f /etc/logrotate.conf           # Force log rotation
zgrep pattern /var/log/file.gz             # Search compressed logs

# Performance Monitoring
top                                        # Process monitor
htop                                       # Interactive process monitor
iotop                                      # I/O monitor
vmstat 1 5                                # Virtual memory stats
iostat 1 5                                # I/O statistics

# Log Analysis
awk '{print $1}' access.log | sort | uniq -c  # Count unique IPs
grep "Failed password" /var/log/auth.log   # Find failed logins
journalctl -p err --since yesterday        # Yesterday's errors
```
