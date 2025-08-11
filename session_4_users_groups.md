# User and Group Management

## Table of Contents
1. [Understanding Users and Groups](#understanding-users-and-groups)
2. [User Account Files](#user-account-files)
3. [Adding Users and Groups](#adding-users-and-groups)
4. [Modifying Users and Groups](#modifying-users-and-groups)
5. [Authentication Methods](#authentication-methods)
6. [Automation Scripts](#automation-scripts)
7. [Practical Scenarios](#practical-scenarios)

---

## Understanding Users and Groups

### User Types in Linux

1. **Root User (UID 0)**: Superuser with complete system access
2. **System Users (UID 1-999)**: Used by system services and daemons
3. **Regular Users (UID 1000+)**: Human users with limited privileges

### Groups
- **Primary Group**: User's default group (GID in /etc/passwd)
- **Secondary Groups**: Additional groups user belongs to
- **System Groups**: Used by system processes

---

## User Account Files

### /etc/passwd
Contains user account information:
```bash
# Format: username:password:UID:GID:GECOS:home_directory:shell
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
john:x:1001:1001:John Doe,,,:/home/john:/bin/bash
```

### /etc/shadow
Contains encrypted passwords and password policies:
```bash
# Format: username:encrypted_password:last_change:min_age:max_age:warn_period:inactive:expire:reserved
root:$6$xyz123...:19000:0:99999:7:::
john:$6$abc456...:19001:0:90:7:30::
```

### /etc/group
Contains group information:
```bash
# Format: group_name:password:GID:user_list
root:x:0:
sudo:x:27:john,alice
developers:x:1500:john,bob,charlie
```

### /etc/gshadow
Contains group password information (rarely used):
```bash
# Format: group_name:encrypted_password:administrators:members
sudo:*::john,alice
developers:!::john,bob,charlie
```

---

## Adding Users and Groups

### Creating Users

#### Using useradd (Low-level command)
```bash
# Basic user creation
sudo useradd -m -s /bin/bash john

# Create user with specific options
sudo useradd -m -s /bin/bash -c "John Doe" -e 2025-12-31 john

# Create user with specific UID and GID
sudo useradd -m -u 1500 -g 1500 -s /bin/bash john
```

#### Using adduser (Interactive, Debian/Ubuntu)
```bash
# Interactive user creation (recommended for beginners)
sudo adduser john

# Create system user
sudo adduser --system --group serviceuser
```

#### Common useradd Options
```bash
-m, --create-home     # Create home directory
-s, --shell          # Login shell
-c, --comment        # User description (GECOS field)
-e, --expiredate     # Account expiration date
-g, --gid            # Primary group
-G, --groups         # Secondary groups
-u, --uid            # User ID
-d, --home-dir       # Home directory path
```

### Creating Groups

```bash
# Create a new group
sudo groupadd developers

# Create group with specific GID
sudo groupadd -g 1500 developers

# Create system group
sudo groupadd -r servicegroup
```

---

## Modifying Users and Groups

### User Modification (usermod)

```bash
# Add user to secondary group
sudo usermod -aG sudo john

# Change user's shell
sudo usermod -s /bin/zsh john

# Change user's home directory
sudo usermod -d /new/home/path -m john

# Lock user account
sudo usermod -L john

# Unlock user account
sudo usermod -U john

# Set account expiration
sudo usermod -e 2025-12-31 john

# Change user's primary group
sudo usermod -g newgroup john
```

### Password Management

```bash
# Set password for user
sudo passwd john

# Force password change on next login
sudo passwd -e john

# Lock/unlock password
sudo passwd -l john    # Lock
sudo passwd -u john    # Unlock

# Set password aging policies
sudo chage -M 90 -m 7 -W 14 john
# Max age: 90 days, Min age: 7 days, Warning: 14 days

# View password aging information
sudo chage -l john
```

### Group Modification (groupmod)

```bash
# Change group name
sudo groupmod -n newname oldname

# Change group ID
sudo groupmod -g 1600 developers

# Add user to group
sudo gpasswd -a john developers

# Remove user from group
sudo gpasswd -d john developers
```

---

## Authentication Methods

### PAM (Pluggable Authentication Modules)

PAM configuration files are located in `/etc/pam.d/`

#### Common PAM Files:
- `/etc/pam.d/common-auth`: Authentication settings
- `/etc/pam.d/common-password`: Password policies
- `/etc/pam.d/sudo`: Sudo authentication

#### Example PAM Configuration
```bash
# /etc/pam.d/common-auth
auth    [success=1 default=ignore]      pam_unix.so nullok_secure
auth    requisite                       pam_deny.so
auth    required                        pam_permit.so
```

### SSH Key Authentication

```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -C "john@company.com"

# Copy public key to remote server
ssh-copy-id john@server.example.com

# Manual key installation
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh/
```

---

## Automation Scripts

### Script: Add Multiple Users

```bash
#!/bin/bash
# add_users.sh - Script to add multiple users

USERS_FILE="users.txt"
DEFAULT_PASSWORD="TempPass123!"

if [ ! -f "$USERS_FILE" ]; then
    echo "Error: $USERS_FILE not found!"
    echo "Create a file with format: username:fullname:group"
    exit 1
fi

while IFS=':' read -r username fullname group; do
    # Skip empty lines and comments
    [[ -z "$username" || "$username" =~ ^#.*$ ]] && continue
    
    echo "Creating user: $username"
    
    # Create user
    if useradd -m -s /bin/bash -c "$fullname" "$username"; then
        echo "User $username created successfully"
        
        # Set temporary password
        echo "$username:$DEFAULT_PASSWORD" | chpasswd
        
        # Force password change on first login
        passwd -e "$username"
        
        # Add to secondary group if specified
        if [ ! -z "$group" ] && getent group "$group" > /dev/null; then
            usermod -aG "$group" "$username"
            echo "Added $username to group $group"
        fi
    else
        echo "Failed to create user $username"
    fi
done < "$USERS_FILE"

echo "User creation completed!"
```

### Script: Password Policy Enforcement

```bash
#!/bin/bash
# enforce_password_policy.sh

USER_LIST=$(cut -d: -f1 /etc/passwd | grep -E '^[^:]+$' | grep -v root)

for user in $USER_LIST; do
    # Check if it's a regular user (UID >= 1000)
    uid=$(id -u "$user" 2>/dev/null)
    if [ "$uid" -ge 1000 ] 2>/dev/null; then
        echo "Applying password policy to $user"
        
        # Set password aging
        chage -M 90 -m 7 -W 14 -I 30 "$user"
        
        # Check password strength (requires cracklib)
        echo "Password policy applied for $user"
    fi
done
```

### Script: User Account Audit

```bash
#!/bin/bash
# user_audit.sh - Audit user accounts

echo "=== USER ACCOUNT AUDIT ==="
echo "Date: $(date)"
echo

# Users with UID 0 (should only be root)
echo "Users with UID 0:"
awk -F: '$3 == 0 {print $1}' /etc/passwd
echo

# Users with empty passwords
echo "Users with empty passwords:"
awk -F: '$2 == "" {print $1}' /etc/shadow
echo

# Users with expired accounts
echo "Users with expired accounts:"
while IFS=: read -r user pass lastchange min max warn inactive expire reserved; do
    if [ "$expire" != "" ] && [ "$expire" -lt "$(date +%s)" ]; then
        echo "$user (expired: $(date -d @$expire))"
    fi
done < /etc/shadow
echo

# Users not logged in for 90 days
echo "Users inactive for 90+ days:"
find /home -maxdepth 1 -type d -mtime +90 -exec basename {} \; 2>/dev/null | grep -v "^home$"
```

---

## Practical Scenarios

### Scenario 1: Setting up Development Team

```bash
#!/bin/bash
# Setup development team structure

# Create groups
groupadd developers
groupadd qa_team
groupadd devops

# Create shared directories
mkdir -p /opt/projects/{dev,staging,production}
chgrp developers /opt/projects/dev
chgrp qa_team /opt/projects/staging  
chgrp devops /opt/projects/production

# Set permissions
chmod 775 /opt/projects/*
chmod g+s /opt/projects/*  # Set GID bit for group ownership inheritance

# Create users
declare -A users=(
    ["alice"]="Alice Johnson:developers,devops"
    ["bob"]="Bob Smith:developers"
    ["charlie"]="Charlie Brown:qa_team"
    ["diana"]="Diana Prince:devops"
)

for username in "${!users[@]}"; do
    IFS=':' read -r fullname groups <<< "${users[$username]}"
    
    # Create user
    useradd -m -s /bin/bash -c "$fullname" "$username"
    
    # Set temporary password
    echo "$username:Dev123!" | chpasswd
    passwd -e "$username"
    
    # Add to groups
    usermod -aG "$groups" "$username"
    
    echo "Created user $username in groups: $groups"
done
```

### Scenario 2: Temporary User Account

```bash
#!/bin/bash
# create_temp_user.sh - Create temporary user account

USERNAME="$1"
DURATION_DAYS="$2"

if [ $# -ne 2 ]; then
    echo "Usage: $0 <username> <duration_in_days>"
    exit 1
fi

# Calculate expiration date
EXPIRE_DATE=$(date -d "+${DURATION_DAYS} days" +%Y-%m-%d)

# Create user with expiration
useradd -m -s /bin/bash -e "$EXPIRE_DATE" "$USERNAME"

# Generate random password
TEMP_PASSWORD=$(openssl rand -base64 12)
echo "$USERNAME:$TEMP_PASSWORD" | chpasswd

# Create user info file
cat > "/tmp/${USERNAME}_info.txt" << EOF
Temporary User Account Created
Username: $USERNAME
Password: $TEMP_PASSWORD
Expires: $EXPIRE_DATE
Created: $(date)
EOF

echo "Temporary user $USERNAME created, expires on $EXPIRE_DATE"
echo "Credentials saved to /tmp/${USERNAME}_info.txt"
```

### Scenario 3: User Migration Script

```bash
#!/bin/bash
# migrate_user.sh - Migrate user to new server

SOURCE_USER="$1"
TARGET_SERVER="$2"

if [ $# -ne 2 ]; then
    echo "Usage: $0 <username> <target_server>"
    exit 1
fi

echo "Migrating user $SOURCE_USER to $TARGET_SERVER"

# Extract user information
USER_INFO=$(getent passwd "$SOURCE_USER")
if [ -z "$USER_INFO" ]; then
    echo "User $SOURCE_USER not found"
    exit 1
fi

# Parse user information
IFS=':' read -r username x uid gid gecos home shell <<< "$USER_INFO"

# Get groups
GROUPS=$(groups "$SOURCE_USER" | cut -d' ' -f3- | tr ' ' ',')

# Create user on target server
ssh root@"$TARGET_SERVER" "
    # Create user
    useradd -u $uid -g $gid -c '$gecos' -d '$home' -s '$shell' -m '$username'
    
    # Add to groups (create groups if they don't exist)
    for group in \$(echo '$GROUPS' | tr ',' ' '); do
        groupadd -f \$group 2>/dev/null
        usermod -aG \$group '$username'
    done
"

# Copy home directory
rsync -avz -e ssh "$home/" root@"$TARGET_SERVER":"$home/"

# Copy shadow entry (password)
SHADOW_ENTRY=$(sudo grep "^$SOURCE_USER:" /etc/shadow)
ssh root@"$TARGET_SERVER" "
    # Update shadow file
    sed -i '/^$username:/d' /etc/shadow
    echo '$SHADOW_ENTRY' >> /etc/shadow
"

echo "User migration completed for $SOURCE_USER"
```

---

## Lab Exercises

### Exercise 1: Create Development Environment
1. Create groups: `frontend`, `backend`, `database`
2. Create users and assign to appropriate groups:
   - john → frontend, backend
   - alice → backend, database  
   - bob → frontend
3. Create shared directories with proper permissions
4. Test file creation and group ownership

### Exercise 2: Password Policy Implementation
1. Configure password aging for all users
2. Set password complexity requirements
3. Create script to check policy compliance
4. Test with temporary user account

### Exercise 3: User Account Maintenance
1. Create audit script for user accounts
2. Identify and handle inactive accounts
3. Generate user activity report
4. Implement automated cleanup for expired accounts

---

## Common Troubleshooting

### User Creation Issues
```bash
# Check if username already exists
getent passwd username

# Verify group exists
getent group groupname

# Check UID/GID conflicts
awk -F: '{print $3}' /etc/passwd | sort -n | uniq -d  # Duplicate UIDs
```

### Permission Problems
```bash
# Reset home directory permissions
sudo chown -R username:usergroup /home/username
sudo chmod 755 /home/username

# Fix SSH directory permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Account Lockouts
```bash
# Check account status
sudo passwd -S username

# Check failed login attempts
sudo grep "Failed password" /var/log/auth.log | grep username

# Unlock account
sudo passwd -u username
sudo pam_tally2 --user=username --reset
```
