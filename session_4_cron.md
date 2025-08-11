# Cron Jobs and Task Scheduling

## Table of Contents
1. [Understanding Cron](#understanding-cron)
2. [Cron Syntax and Format](#cron-syntax-and-format)
3. [Managing Cron Jobs](#managing-cron-jobs)
4. [System Cron vs User Cron](#system-cron-vs-user-cron)
5. [Advanced Cron Techniques](#advanced-cron-techniques)
6. [systemd Timers](#systemd-timers)
7. [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
8. [Practical Scenarios](#practical-scenarios)

---

## Understanding Cron

### What is Cron?

Cron is a time-based job scheduler in Unix-like systems that enables users to schedule jobs (commands or scripts) to run automatically at specified times, dates, or intervals.

### Cron Components

- **crond**: The cron daemon that runs in the background
- **crontab**: The command to edit cron jobs and the file containing cron jobs
- **cron.allow/cron.deny**: Files controlling user access to cron
- **/etc/cron.d/**: Directory for system-wide cron jobs
- **/var/log/cron**: Log file for cron activities

### Cron Directories

```bash
# System cron directories
/etc/cron.hourly/      # Scripts run every hour
/etc/cron.daily/       # Scripts run every day
/etc/cron.weekly/      # Scripts run every week  
/etc/cron.monthly/     # Scripts run every month

# Configuration files
/etc/crontab           # System-wide cron table
/etc/cron.allow        # Users allowed to use cron
/etc/cron.deny         # Users denied cron access
/var/spool/cron/       # User cron files (crontabs)
```

---

## Cron Syntax and Format

### Crontab Format

```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
# │ │ │ │ │
# │ │ │ │ │
# * * * * * /path/to/command arg1 arg2
```

### Special Characters

| Character | Meaning | Example |
|-----------|---------|---------|
| * | Any value | `* * * * *` (every minute) |
| , | Value list separator | `1,3,5 * * * *` (1st, 3rd, 5th minute) |
| - | Range of values | `1-5 * * * *` (minutes 1 through 5) |
| / | Step values | `*/15 * * * *` (every 15 minutes) |
| ? | No specific value (some systems) | |
| L | Last (some systems) | `0 0 L * *` (last day of month) |
| W | Weekday (some systems) | `0 0 15W * *` (nearest weekday to 15th) |

### Common Cron Examples

```bash
# Every minute
* * * * * /path/to/script.sh

# Every 5 minutes
*/5 * * * * /path/to/script.sh

# Every hour at minute 0
0 * * * * /path/to/script.sh

# Every day at midnight
0 0 * * * /path/to/script.sh

# Every day at 6:30 AM
30 6 * * * /path/to/script.sh

# Every Monday at 8:00 AM
0 8 * * 1 /path/to/script.sh

# Every 1st of the month at midnight
0 0 1 * * /path/to/script.sh

# Every weekday at 9:00 AM
0 9 * * 1-5 /path/to/script.sh

# Multiple times per day (6 AM, 2 PM, 10 PM)
0 6,14,22 * * * /path/to/script.sh

# Every 15 minutes during business hours
*/15 9-17 * * 1-5 /path/to/script.sh

# Twice a year (January 1st and July 1st)
0 0 1 1,7 * /path/to/script.sh
```

### Special Strings (Shortcuts)

```bash
@reboot         # Run once at startup
@yearly         # Run once a year (0 0 1 1 *)
@annually       # Same as @yearly
@monthly        # Run once a month (0 0 1 * *)
@weekly         # Run once a week (0 0 * * 0)
@daily          # Run once a day (0 0 * * *)
@midnight       # Same as @daily
@hourly         # Run once an hour (0 * * * *)

# Examples
@reboot /path/to/startup_script.sh
@daily /usr/local/bin/backup.sh
@weekly /usr/local/bin/system_maintenance.sh
```

---

## Managing Cron Jobs

### User Crontab Management

```bash
# Edit user's crontab
crontab -e

# List user's crontab
crontab -l

# Remove user's crontab
crontab -r

# Edit another user's crontab (as root)
crontab -u username -e
crontab -u username -l

# Install crontab from file
crontab < cron_file.txt
crontab cron_file.txt

# Backup current crontab
crontab -l > my_cron_backup.txt
```

### System Crontab (/etc/crontab)

```bash
# System crontab includes a user field
# minute hour day month dow user command

# Example /etc/crontab
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed

17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

### Cron Job Environment

```bash
# Cron environment variables
SHELL=/bin/sh          # Default shell
PATH=/usr/bin:/bin     # Limited PATH
HOME=/root             # Home directory
LOGNAME=root          # User name
MAILTO=root           # Email notifications

# Example crontab with environment variables
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.com

0 2 * * * /usr/local/bin/backup.sh
```

### Using /etc/cron.d/

```bash
# Create custom system cron job
sudo tee /etc/cron.d/myapp << EOF
# MyApp maintenance tasks
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=admin@example.com

# Clean temp files every hour
0 * * * * myapp /usr/local/bin/cleanup_temp.sh

# Daily backup at 3 AM
0 3 * * * myapp /usr/local/bin/backup_myapp.sh

# Weekly log rotation on Sundays at 4 AM
0 4 * * 0 myapp /usr/local/bin/rotate_logs.sh
EOF
```

---

## System Cron vs User Cron

### User Cron

- **Location**: `/var/spool/cron/crontabs/username`
- **Access**: Each user manages their own crontab
- **Environment**: Runs with user's privileges
- **Editing**: `crontab -e`

```bash
# User crontab example
# Send a report every weekday at 5 PM
0 17 * * 1-5 /home/user/scripts/daily_report.sh

# Personal backup every Sunday at 2 AM  
0 2 * * 0 /home/user/scripts/backup_personal.sh
```

### System Cron

- **Location**: `/etc/crontab`, `/etc/cron.d/`
- **Access**: Root access required
- **Environment**: Can run as any user (specified in crontab)
- **Editing**: Direct file editing

```bash
# System cron example (/etc/cron.d/system-maintenance)
# System update check daily at 3 AM
0 3 * * * root /usr/local/bin/system_update_check.sh

# Database maintenance weekly
0 2 * * 0 postgres /usr/local/bin/db_maintenance.sh
```

### Access Control

```bash
# Allow specific users to use cron
echo "alice" | sudo tee -a /etc/cron.allow
echo "bob" | sudo tee -a /etc/cron.allow

# Deny specific users from using cron
echo "baduser" | sudo tee -a /etc/cron.deny

# Check if user can use cron
crontab -l  # Will show error if denied

# If neither file exists, only root can use cron
# If cron.allow exists, only listed users can use cron
# If only cron.deny exists, all users except listed ones can use cron
```

---

## Advanced Cron Techniques

### Logging and Error Handling

```bash
# Redirect output to log file
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Separate stdout and stderr
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>> /var/log/backup_error.log

# Email only on error
0 2 * * * /usr/local/bin/backup.sh >/dev/null

# Disable email notifications
MAILTO=""
0 2 * * * /usr/local/bin/backup.sh

# Log with timestamp
0 2 * * * echo "$(date): Starting backup" >> /var/log/backup.log; /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### Locking Mechanisms

```bash
#!/bin/bash
# backup_with_lock.sh - Prevent overlapping executions

LOCKFILE=/var/run/backup.lock

# Function to cleanup on exit
cleanup() {
    rm -f "$LOCKFILE"
    exit $1
}

# Set up signal handlers
trap 'cleanup 1' INT TERM

# Check if already running
if [ -f "$LOCKFILE" ]; then
    if kill -0 `cat "$LOCKFILE"` 2>/dev/null; then
        echo "Backup already running (PID: $(cat $LOCKFILE))"
        exit 1
    else
        echo "Removing stale lock file"
        rm -f "$LOCKFILE"
    fi
fi

# Create lock file
echo $$ > "$LOCKFILE"

# Perform backup
echo "Starting backup at $(date)"
# ... backup commands here ...
echo "Backup completed at $(date)"

# Clean up
cleanup 0
```

### Complex Scheduling

```bash
# Run every 2 hours during business days
0 */2 * * 1-5 /path/to/script.sh

# Run on the 1st and 15th of every month
0 0 1,15 * * /path/to/script.sh

# Run every 30 minutes but only Monday to Friday 9-5
*/30 9-17 * * 1-5 /path/to/script.sh

# Run quarterly (every 3 months on the 1st)
0 0 1 */3 * /path/to/script.sh

# Run on the last day of every month
0 0 28-31 * * [ $(date -d tomorrow +\%d) -eq 1 ] && /path/to/script.sh
```

### Conditional Execution

```bash
# Run only if file exists
0 2 * * * [ -f /tmp/run_backup ] && /usr/local/bin/backup.sh

# Run only on first Monday of month
0 9 1-7 * 1 /path/to/script.sh

# Run only if load average is low
*/5 * * * * [ $(cut -d. -f1 /proc/loadavg) -lt 2 ] && /path/to/maintenance.sh

# Run with different schedules based on conditions
0 2 * * * if [ $(date +\%u) -eq 7 ]; then /full_backup.sh; else /incremental_backup.sh; fi
```

---

## systemd Timers

### Why systemd Timers?

Modern alternative to cron with advantages:
- Better logging integration
- Service dependencies
- More flexible scheduling
- Better resource management

### Creating systemd Timer

#### 1. Create the service file

```bash
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup Service
Wants=backup.timer

[Service]
Type=oneshot
User=backup
Group=backup
ExecStart=/usr/local/bin/backup.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

#### 2. Create the timer file

```bash
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer
Requires=backup.service

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1800

[Install]
WantedBy=timers.target
```

#### 3. Enable and start timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable backup.timer
sudo systemctl start backup.timer

# Check timer status
systemctl status backup.timer
systemctl list-timers backup.timer
```

### Timer Calendar Events

```bash
# Every minute
OnCalendar=*:*

# Every 15 minutes
OnCalendar=*:0/15

# Daily at 6:30 AM
OnCalendar=06:30

# Weekly on Monday at 8 AM
OnCalendar=Mon 08:00

# Monthly on 1st at midnight
OnCalendar=*-*-01 00:00:00

# Yearly on January 1st
OnCalendar=*-01-01 00:00:00

# Complex: Every 5 minutes during business hours, weekdays
OnCalendar=Mon..Fri 09..17:0/5
```

### Advanced Timer Options

```bash
[Timer]
# Time-based triggers
OnCalendar=daily
OnBootSec=15min                # 15 minutes after boot
OnStartupSec=1h               # 1 hour after systemd startup
OnActiveSec=30min             # 30 minutes after timer activation
OnUnitActiveSec=1h            # 1 hour after service last activation
OnUnitInactiveSec=30min       # 30 minutes after service deactivation

# Timer behavior
Persistent=true               # Catch up missed runs
RandomizedDelaySec=300        # Random delay up to 5 minutes
AccuracySec=60s              # Timer accuracy (default 1 minute)
RemainAfterElapse=false      # Remove timer after elapsing
WakeSystem=false             # Wake system from suspend

[Install]
WantedBy=timers.target
```

### Monitoring systemd Timers

```bash
# List all timers
systemctl list-timers

# List specific timer
systemctl list-timers backup.timer

# Show timer status
systemctl status backup.timer

# Show timer logs
journalctl -u backup.timer
journalctl -u backup.service

# Show next run time
systemctl list-timers --no-pager | grep backup
```

---

## Monitoring and Troubleshooting

### Cron Monitoring

```bash
# Check if cron daemon is running
systemctl status cron          # Debian/Ubuntu
systemctl status crond         # RHEL/CentOS

# View cron logs
tail -f /var/log/cron
journalctl -u cron -f

# Check user's cron jobs
crontab -l

# Check system cron jobs
ls -la /etc/cron.d/
cat /etc/crontab

# Check recent cron activity
grep CRON /var/log/syslog | tail -20
```

### Common Issues and Solutions

#### Cron Job Not Running

```bash
# Check cron daemon status
sudo systemctl status cron

# Verify crontab syntax
crontab -l | head -5

# Check user permissions
ls -la /var/spool/cron/crontabs/

# Test command manually
/bin/bash -c "command from crontab"

# Check environment
* * * * * env > /tmp/cron-env.txt
```

#### Environment Issues

```bash
# Common cron environment problems and solutions

# 1. PATH issues - specify full paths
0 2 * * * /usr/bin/python3 /home/user/script.py

# 2. Set PATH in crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 2 * * * python3 /home/user/script.py

# 3. Source user environment
0 2 * * * bash -l -c '/home/user/script.py'

# 4. Set HOME directory
HOME=/home/user
0 2 * * * ./script.py
```

#### Permission Issues

```bash
# Check file permissions
ls -la /path/to/script.sh

# Make script executable
chmod +x /path/to/script.sh

# Check directory permissions
ls -la /path/to/

# Run as specific user
0 2 * * * sudo -u username /path/to/script.sh
```

### Cron Debugging Script

```bash
#!/bin/bash
# cron_debug.sh - Debug cron issues

USER="${1:-$(whoami)}"
DEBUG_FILE="/tmp/cron_debug_$USER.txt"

{
    echo "Cron Debug Report for $USER"
    echo "Generated: $(date)"
    echo "=========================="
    echo
    
    # Check if cron daemon is running
    echo "1. Cron Daemon Status:"
    if systemctl is-active --quiet cron 2>/dev/null || systemctl is-active --quiet crond 2>/dev/null; then
        echo "✓ Cron daemon is running"
    else
        echo "✗ Cron daemon is not running"
    fi
    echo
    
    # Check user's crontab
    echo "2. User Crontab ($USER):"
    if crontab -l -u "$USER" 2>/dev/null; then
        echo "✓ Crontab exists"
    else
        echo "✗ No crontab for user $USER"
    fi
    echo
    
    # Check cron access
    echo "3. Cron Access Control:"
    if [ -f /etc/cron.allow ]; then
        if grep -q "^$USER$" /etc/cron.allow; then
            echo "✓ User explicitly allowed"
        else
            echo "✗ User not in cron.allow"
        fi
    elif [ -f /etc/cron.deny ]; then
        if grep -q "^$USER$" /etc/cron.deny; then
            echo "✗ User explicitly denied"
        else
            echo "✓ User not denied"
        fi
    else
        echo "✓ No access control files (all users allowed)"
    fi
    echo
    
    # Check recent cron activity
    echo "4. Recent Cron Activity:"
    if [ -f /var/log/cron ]; then
        grep "$USER" /var/log/cron | tail -5
    elif command -v journalctl >/dev/null; then
        journalctl -u cron | grep "$USER" | tail -5
    else
        grep CRON /var/log/syslog | grep "$USER" | tail -5
    fi
    echo
    
    # Test cron environment
    echo "5. Cron Environment Test:"
    echo "Creating test cron job to capture environment..."
    
    # Backup current crontab
    crontab -l > "/tmp/crontab_backup_$USER.txt" 2>/dev/null
    
    # Add test job
    (crontab -l 2>/dev/null; echo "* * * * * env > /tmp/cron_env_$USER.txt 2>&1") | crontab -
    
    echo "Test job added. Wait 1-2 minutes and check /tmp/cron_env_$USER.txt"
    
    # Restore original crontab
    if [ -f "/tmp/crontab_backup_$USER.txt" ]; then
        crontab "/tmp/crontab_backup_$USER.txt"
        rm "/tmp/crontab_backup_$USER.txt"
    else
        crontab -r 2>/dev/null
    fi
    
} > "$DEBUG_FILE"

echo "Debug report created: $DEBUG_FILE"
cat "$DEBUG_FILE"
```

---

## Practical Scenarios

### Scenario 1: Automated Backup System

```bash
#!/bin/bash
# automated_backup.sh - Comprehensive backup solution

BACKUP_CONFIG="/etc/backup.conf"
BACKUP_LOG="/var/log/backup.log"
LOCK_FILE="/var/run/backup.lock"
EMAIL="admin@example.com"

# Default configuration
BACKUP_DIRS="/home /etc /var/www"
BACKUP_DEST="/backup"
RETENTION_DAYS=30
COMPRESSION="gzip"

# Load configuration if exists
[ -f "$BACKUP_CONFIG" ] && source "$BACKUP_CONFIG"

# Logging function
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$BACKUP_LOG"
}

# Error handling
error_exit() {
    log_message "ERROR: $1"
    rm -f "$LOCK_FILE"
    echo "$1" | mail -s "Backup Failed" "$EMAIL"
    exit 1
}

# Check if backup is already running
if [ -f "$LOCK_FILE" ]; then
    if kill -0 $(cat "$LOCK_FILE") 2>/dev/null; then
        error_exit "Backup already running (PID: $(cat $LOCK_FILE))"
    else
        log_message "Removing stale lock file"
        rm -f "$LOCK_FILE"
    fi
fi

# Create lock file
echo $$ > "$LOCK_FILE"
trap 'rm -f "$LOCK_FILE"; exit' INT TERM EXIT

# Start backup
log_message "Starting backup process"

# Create backup directory with timestamp
BACKUP_DATE=$(date '+%Y%m%d_%H%M%S')
BACKUP_PATH="$BACKUP_DEST/$BACKUP_DATE"

if ! mkdir -p "$BACKUP_PATH"; then
    error_exit "Cannot create backup directory: $BACKUP_PATH"
fi

# Backup each directory
for dir in $BACKUP_DIRS; do
    if [ -d "$dir" ]; then
        log_message "Backing up $dir"
        
        case "$COMPRESSION" in
            "gzip")
                tar -czf "$BACKUP_PATH/$(basename $dir).tar.gz" -C "$(dirname $dir)" "$(basename $dir)"
                ;;
            "bzip2")
                tar -cjf "$BACKUP_PATH/$(basename $dir).tar.bz2" -C "$(dirname $dir)" "$(basename $dir)"
                ;;
            "none")
                tar -cf "$BACKUP_PATH/$(basename $dir).tar" -C "$(dirname $dir)" "$(basename $dir)"
                ;;
            *)
                tar -czf "$BACKUP_PATH/$(basename $dir).tar.gz" -C "$(dirname $dir)" "$(basename $dir)"
                ;;
        esac
        
        if [ $? -eq 0 ]; then
            log_message "Successfully backed up $dir"
        else
            log_message "Failed to backup $dir"
        fi
    else
        log_message "Directory $dir does not exist, skipping"
    fi
done

# Create backup manifest
cat > "$BACKUP_PATH/manifest.txt" << EOF
Backup created: $(date)
Hostname: $(hostname)
Directories backed up: $BACKUP_DIRS
Compression: $COMPRESSION
Backup size: $(du -sh "$BACKUP_PATH" | cut -f1)
EOF

log_message "Backup manifest created"

# Cleanup old backups
log_message "Cleaning up backups older than $RETENTION_DAYS days"
find "$BACKUP_DEST" -maxdepth 1 -type d -name "[0-9]*" -mtime +$RETENTION_DAYS -exec rm -rf {} \;

# Calculate backup statistics
BACKUP_SIZE=$(du -sh "$BACKUP_PATH" | cut -f1)
TOTAL_BACKUPS=$(ls -1 "$BACKUP_DEST" | wc -l)

log_message "Backup completed successfully"
log_message "Backup size: $BACKUP_SIZE"
log_message "Total backups: $TOTAL_BACKUPS"

# Send success notification
cat << EOF | mail -s "Backup Completed Successfully" "$EMAIL"
Backup completed successfully on $(hostname)

Backup Details:
- Date: $(date)
- Location: $BACKUP_PATH
- Size: $BACKUP_SIZE
- Total Backups: $TOTAL_BACKUPS

Directories backed up:
$BACKUP_DIRS
EOF

# Crontab entries for this script:
# Daily backup at 2 AM
# 0 2 * * * /usr/local/bin/automated_backup.sh
```

### Scenario 2: System Maintenance Automation

```bash
#!/bin/bash
# system_maintenance.sh - Weekly system maintenance

MAINTENANCE_LOG="/var/log/system_maintenance.log"
EMAIL="admin@example.com"
MAINTENANCE_DAY=0  # Sunday

# Check if today is maintenance day
if [ "$(date +%w)" -ne "$MAINTENANCE_DAY" ]; then
    exit 0
fi

log_maintenance() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$MAINTENANCE_LOG"
}

send_report() {
    local subject="$1"
    local report_file="/tmp/maintenance_report_$(date +%Y%m%d).txt"
    
    # Generate report
    {
        echo "System Maintenance Report - $(date)"
        echo "=================================="
        echo
        
        # System information
        echo "System Information:"
        echo "Hostname: $(hostname)"
        echo "Uptime: $(uptime)"
        echo "Kernel: $(uname -r)"
        echo "Distribution: $(lsb_release -d 2>/dev/null | cut -f2 || echo "Unknown")"
        echo
        
        # Disk usage
        echo "Disk Usage:"
        df -h | grep -E '^/dev/'
        echo
        
        # Memory usage
        echo "Memory Usage:"
        free -h
        echo
        
        # Last week's maintenance log
        echo "Maintenance Activities:"
        tail -50 "$MAINTENANCE_LOG"
        
    } > "$report_file"
    
    cat "$report_file" | mail -s "$subject" "$EMAIL"
}

log_maintenance "Starting weekly system maintenance"

# 1. Update package lists
log_maintenance "Updating package lists"
if command -v apt-get >/dev/null; then
    apt-get update >> "$MAINTENANCE_LOG" 2>&1
elif command -v yum >/dev/null; then
    yum check-update >> "$MAINTENANCE_LOG" 2>&1
fi

# 2. Check for available updates
log_maintenance "Checking for available updates"
if command -v apt-get >/dev/null; then
    UPDATES=$(apt list --upgradable 2>/dev/null | wc -l)
    log_maintenance "$((UPDATES-1)) packages can be updated"
elif command -v yum >/dev/null; then
    UPDATES=$(yum list updates 2>/dev/null | grep -v "Updated Packages" | wc -l)
    log_maintenance "$UPDATES packages can be updated"
fi

# 3. Clean package cache
log_maintenance "Cleaning package cache"
if command -v apt-get >/dev/null; then
    apt-get clean >> "$MAINTENANCE_LOG" 2>&1
    apt-get autoremove -y >> "$MAINTENANCE_LOG" 2>&1
elif command -v yum >/dev/null; then
    yum clean all >> "$MAINTENANCE_LOG" 2>&1
fi

# 4. Rotate logs manually if needed
log_maintenance "Forcing log rotation"
logrotate -f /etc/logrotate.conf >> "$MAINTENANCE_LOG" 2>&1

# 5. Check disk usage
log_maintenance "Checking disk usage"
DISK_USAGE=$(df / | awk 'NR==2 {print int($5)}')
if [ "$DISK_USAGE" -gt 80 ]; then
    log_maintenance "WARNING: Disk usage is ${DISK_USAGE}%"
    # Find large files
    find / -type f -size +100M 2>/dev/null | head -10 >> "$MAINTENANCE_LOG"
fi

# 6. Check system load
log_maintenance "Checking system load"
LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | tr -d ',')
log_maintenance "Current load average: $LOAD_AVG"

# 7. Restart services if needed
log_maintenance "Checking services that need restart"
if [ -f /var/run/reboot-required ]; then
    log_maintenance "System reboot required"
    echo "System reboot required on $(hostname)" | mail -s "Reboot Required" "$EMAIL"
fi

# 8. Database maintenance (if applicable)
if systemctl is-active --quiet postgresql; then
    log_maintenance "Running PostgreSQL maintenance"
    sudo -u postgres psql -c "VACUUM ANALYZE;" >> "$MAINTENANCE_LOG" 2>&1
fi

if systemctl is-active --quiet mysql; then
    log_maintenance "Running MySQL maintenance"
    mysql -e "FLUSH LOGS; OPTIMIZE TABLE mysql.general_log;" >> "$MAINTENANCE_LOG" 2>&1
fi

log_maintenance "Weekly maintenance completed"

# Send maintenance report
send_report "Weekly System Maintenance Report - $(hostname)"

# Crontab entry for this script:
# Weekly maintenance on Sunday at 4 AM
# 0 4 * * 0 /usr/local/bin/system_maintenance.sh
```

### Scenario 3: Database Backup with Rotation

```bash
#!/bin/bash
# db_backup_rotation.sh - Database backup with intelligent rotation

CONFIG_FILE="/etc/db_backup.conf"
BACKUP_LOG="/var/log/db_backup.log"
LOCK_FILE="/var/run/db_backup.lock"

# Default configuration
DB_TYPE="postgresql"
DB_HOST="localhost"
DB_PORT="5432"
DB_NAME="myapp"
DB_USER="backup_user"
DB_PASSWORD=""
BACKUP_DIR="/backup/database"
RETENTION_POLICY="7d 4w 6m 2y"  # 7 daily, 4 weekly, 6 monthly, 2 yearly
COMPRESSION="gzip"
EMAIL="admin@example.com"

# Load configuration
[ -f "$CONFIG_FILE" ] && source "$CONFIG_FILE"

# Logging function
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$BACKUP_LOG"
}

# Backup function for PostgreSQL
backup_postgresql() {
    local backup_file="$1"
    
    if [ -n "$DB_PASSWORD" ]; then
        export PGPASSWORD="$DB_PASSWORD"
    fi
    
    pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
        --no-password --verbose > "$backup_file" 2>>"$BACKUP_LOG"
    
    return $?
}

# Backup function for MySQL
backup_mysql() {
    local backup_file="$1"
    
    local mysql_opts="-h $DB_HOST -P $DB_PORT -u $DB_USER"
    if [ -n "$DB_PASSWORD" ]; then
        mysql_opts="$mysql_opts -p$DB_PASSWORD"
    fi
    
    mysqldump $mysql_opts --single-transaction --routines --triggers \
        "$DB_NAME" > "$backup_file" 2>>"$BACKUP_LOG"
    
    return $?
}

# Compression function
compress_backup() {
    local file="$1"
    
    case "$COMPRESSION" in
        "gzip")
            gzip "$file"
            echo "${file}.gz"
            ;;
        "bzip2")
            bzip2 "$file"
            echo "${file}.bz2"
            ;;
        *)
            echo "$file"
            ;;
    esac
}

# Retention management
manage_retention() {
    local backup_dir="$1"
    
    # Parse retention policy
    local daily=$(echo "$RETENTION_POLICY" | grep -o '[0-9]*d' | sed 's/d//')
    local weekly=$(echo "$RETENTION_POLICY" | grep -o '[0-9]*w' | sed 's/w//')
    local monthly=$(echo "$RETENTION_POLICY" | grep -o '[0-9]*m' | sed 's/m//')
    local yearly=$(echo "$RETENTION_POLICY" | grep -o '[0-9]*y' | sed 's/y//')
    
    log_message "Applying retention policy: ${daily}d ${weekly}w ${monthly}m ${yearly}y"
    
    # Keep daily backups
    if [ -n "$daily" ] && [ "$daily" -gt 0 ]; then
        find "$backup_dir" -name "*daily*" -mtime +$daily -delete
        log_message "Cleaned daily backups older than $daily days"
    fi
    
    # Weekly backups (keep Sunday backups as weekly)
    if [ -n "$weekly" ] && [ "$weekly" -gt 0 ]; then
        # Copy Sunday backups to weekly
        find "$backup_dir" -name "*daily*" -daystart -mtime -7 | while read backup; do
            backup_date=$(basename "$backup" | grep -o '[0-9]\{8\}')
            if [ "$(date -d "$backup_date" +%w)" = "0" ]; then  # Sunday
                weekly_backup=$(echo "$backup" | sed 's/daily/weekly/')
                cp "$backup" "$weekly_backup"
                log_message "Created weekly backup: $(basename $weekly_backup)"
            fi
        done
        
        # Clean old weekly backups
        find "$backup_dir" -name "*weekly*" -mtime +$((weekly * 7)) -delete
        log_message "Cleaned weekly backups older than $weekly weeks"
    fi
    
    # Monthly backups (keep 1st of month as monthly)
    if [ -n "$monthly" ] && [ "$monthly" -gt 0 ]; then
        find "$backup_dir" -name "*daily*" -daystart -mtime -31 | while read backup; do
            backup_date=$(basename "$backup" | grep -o '[0-9]\{8\}')
            if [ "$(date -d "$backup_date" +%d)" = "01" ]; then  # 1st of month
                monthly_backup=$(echo "$backup" | sed 's/daily/monthly/')
                cp "$backup" "$monthly_backup"
                log_message "Created monthly backup: $(basename $monthly_backup)"
            fi
        done
        
        # Clean old monthly backups
        find "$backup_dir" -name "*monthly*" -mtime +$((monthly * 31)) -delete
        log_message "Cleaned monthly backups older than $monthly months"
    fi
    
    # Yearly backups (keep January 1st as yearly)
    if [ -n "$yearly" ] && [ "$yearly" -gt 0 ]; then
        find "$backup_dir" -name "*daily*" -daystart -mtime -366 | while read backup; do
            backup_date=$(basename "$backup" | grep -o '[0-9]\{8\}')
            if [ "$(date -d "$backup_date" +%m%d)" = "0101" ]; then  # Jan 1st
                yearly_backup=$(echo "$backup" | sed 's/daily/yearly/')
                cp "$backup" "$yearly_backup"
                log_message "Created yearly backup: $(basename $yearly_backup)"
            fi
        done
        
        # Clean old yearly backups
        find "$backup_dir" -name "*yearly*" -mtime +$((yearly * 366)) -delete
        log_message "Cleaned yearly backups older than $yearly years"
    fi
}

# Main backup process
main() {
    # Check for lock file
    if [ -f "$LOCK_FILE" ]; then
        if kill -0 $(cat "$LOCK_FILE") 2>/dev/null; then
            log_message "Backup already running (PID: $(cat $LOCK_FILE))"
            exit 1
        else
            rm -f "$LOCK_FILE"
        fi
    fi
    
    # Create lock file
    echo $$ > "$LOCK_FILE"
    trap 'rm -f "$LOCK_FILE"' EXIT
    
    log_message "Starting database backup"
    
    # Create backup directory
    mkdir -p "$BACKUP_DIR"
    
    # Generate backup filename
    TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
    BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_daily_${TIMESTAMP}.sql"
    
    # Perform backup
    case "$DB_TYPE" in
        "postgresql")
            backup_postgresql "$BACKUP_FILE"
            ;;
        "mysql")
            backup_mysql "$BACKUP_FILE"
            ;;
        *)
            log_message "ERROR: Unsupported database type: $DB_TYPE"
            exit 1
            ;;
    esac
    
    if [ $? -eq 0 ]; then
        log_message "Database backup completed: $(basename $BACKUP_FILE)"
        
        # Compress backup
        COMPRESSED_FILE=$(compress_backup "$BACKUP_FILE")
        log_message "Backup compressed: $(basename $COMPRESSED_FILE)"
        
        # Get backup size
        BACKUP_SIZE=$(du -h "$COMPRESSED_FILE" | cut -f1)
        log_message "Backup size: $BACKUP_SIZE"
        
        # Manage retention
        manage_retention "$BACKUP_DIR"
        
        # Send success notification
        echo "Database backup completed successfully for $DB_NAME on $(hostname) at $(date). Backup size: $BACKUP_SIZE" | \
            mail -s "Database Backup Successful" "$EMAIL"
        
    else
        log_message "ERROR: Database backup failed"
        echo "Database backup failed for $DB_NAME on $(hostname) at $(date). Check $BACKUP_LOG for details." | \
            mail -s "Database Backup Failed" "$EMAIL"
        exit 1
    fi
}

main

# Crontab entries:
# Daily backup at 3 AM
# 0 3 * * * /usr/local/bin/db_backup_rotation.sh
```

---

## Lab Exercises

### Exercise 1: Basic Cron Setup
1. Create user crontab with various scheduling patterns
2. Set up system-wide cron jobs in /etc/cron.d/
3. Configure proper logging and error handling
4. Test cron jobs and verify execution

### Exercise 2: Advanced Scheduling
1. Implement conditional cron jobs
2. Create jobs with locking mechanisms
3. Set up complex scheduling patterns
4. Configure email notifications

### Exercise 3: systemd Timer Migration
1. Convert existing cron jobs to systemd timers
2. Configure timer dependencies and resource limits
3. Implement advanced timer features
4. Monitor and troubleshoot timer execution

### Exercise 4: Automated Maintenance System
1. Design comprehensive backup solution
2. Implement intelligent retention policies
3. Create system maintenance automation
4. Set up monitoring and alerting

---

## Quick Reference Commands

```bash
# Cron Management
crontab -e                      # Edit user crontab
crontab -l                      # List user crontab
crontab -r                      # Remove user crontab
sudo crontab -u user -e         # Edit another user's crontab

# System Cron
sudo vim /etc/crontab           # Edit system crontab
ls /etc/cron.d/                 # List system cron jobs
sudo vim /etc/cron.d/jobname    # Edit specific system job

# Monitoring
tail -f /var/log/cron           # Monitor cron logs
systemctl status cron          # Check cron daemon status
grep CRON /var/log/syslog       # Find cron activities

# systemd Timers
systemctl list-timers           # List all timers
systemctl status timer.timer    # Check timer status
journalctl -u timer.timer       # View timer logs
systemctl enable --now timer.timer  # Enable and start timer

# Common Cron Patterns
0 0 * * *                       # Daily at midnight
0 */6 * * *                     # Every 6 hours
30 2 * * 1                      # Monday 2:30 AM
0 0 1 * *                       # 1st of month
@reboot                         # At system boot
```
