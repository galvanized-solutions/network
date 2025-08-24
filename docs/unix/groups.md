# Linux User Groups

## Understanding Linux Groups

Linux groups are collections of users that share common permissions and access rights. Groups provide an efficient way to manage permissions for multiple users simultaneously, following the principle of least privilege while enabling collaborative work environments.

### How Groups Work

**Group Identification:**
- **GID (Group ID):** Numeric identifier for groups (similar to UID for users)
- **Group Name:** Human-readable name for the group
- **Primary Group:** Every user has one primary group (usually matches username)
- **Secondary Groups:** Users can belong to multiple additional groups

**Permission Structure:**
Linux uses the traditional Unix permission model: **Owner - Group - Other**
- **Owner:** The user who owns the file/directory
- **Group:** Members of the file's assigned group
- **Other:** Everyone else on the system

**Example:**
```bash
-rw-r--r-- 1 alice developers 1024 Jan 15 10:30 project.txt
#  │  │  │    │      │
#  │  │  │    │      └── Group: developers
#  │  │  │    └────────── Owner: alice  
#  │  │  └─────────────── Other permissions: read
#  │  └────────────────── Group permissions: read
#  └───────────────────── Owner permissions: read, write
```

## Group Management Commands

### Viewing Groups

```bash
# Display current user's groups
groups

# Show groups for specific user
groups username

# List all groups on system
getent group

# Show detailed group information
getent group groupname

# Display group memberships with GIDs
id username

# Show primary group for user
id -gn username

# Show all secondary groups for user
id -Gn username
```

### Group File Locations

```bash
# Group definitions and members
cat /etc/group

# Group passwords (rarely used)
cat /etc/gshadow

# Show group format
# groupname:password:GID:user1,user2,user3
```

### Creating and Managing Groups

```bash
# Create new group
sudo groupadd groupname

# Create group with specific GID
sudo groupadd -g 1500 customgroup

# Create system group (GID < 1000)
sudo groupadd -r systemgroup

# Delete group
sudo groupdel groupname

# Rename group
sudo groupmod -n newname oldname

# Change group GID
sudo groupmod -g 2000 groupname
```

### Adding/Removing Users

```bash
# Add user to group (secondary group)
sudo usermod -aG groupname username

# Add user to multiple groups
sudo usermod -aG group1,group2,group3 username

# Set user's primary group
sudo usermod -g primarygroup username

# Remove user from group
sudo gpasswd -d username groupname

# Add user to group (alternative method)
sudo gpasswd -a username groupname

# Remove all users from group
sudo gpasswd -M "" groupname

# Set group members (replaces existing)
sudo gpasswd -M user1,user2,user3 groupname
```

## Standard System Groups

### Root and Administrative Groups

#### root (GID 0)
**Purpose:** Superuser group with unlimited system access
**Members:** Usually only the root user
**Permissions:** Full system access, bypass all permission checks
```bash
# Files owned by root group are critical system files
find /etc -group root -type f | head -10
```

#### wheel (GID varies)
**Purpose:** Administrative users who can use su to become root
**Usage:** Traditional Unix group for system administrators
**Note:** Not all Linux distributions use this group by default
```bash
# Check if wheel group exists
getent group wheel

# Traditional wheel group configuration in /etc/pam.d/su:
# auth required pam_wheel.so use_uid
```

#### sudo (GID varies)
**Purpose:** Users who can execute commands with sudo
**Members:** Users authorized to run administrative commands
**Configuration:** Defined in /etc/sudoers and /etc/sudoers.d/
```bash
# List sudo group members
getent group sudo

# Check user's sudo permissions
sudo -l -U username
```

#### admin (GID varies)
**Purpose:** Administrative users (Ubuntu/Debian)
**Usage:** Similar to sudo group, often deprecated in favor of sudo
**Note:** Older distributions may use this instead of sudo group

### System Service Groups

#### daemon (GID 2)
**Purpose:** System daemons and background services
**Usage:** Non-privileged system processes
**Security:** Services run as daemon to limit privileges
```bash
# Processes running as daemon
ps -u daemon
```

#### sys (GID 3)
**Purpose:** System files and processes
**Usage:** System-related files that need group access
**Files:** Various system configuration files

#### adm (GID 4)
**Purpose:** Administrative monitoring and log access
**Members:** System administrators who need to read logs
**Files:** Log files in /var/log/
```bash
# Files accessible to adm group
find /var/log -group adm -type f | head -10
```

#### tty (GID 5)
**Purpose:** Terminal and TTY device access
**Usage:** Controls access to terminal devices
**Devices:** /dev/tty*, /dev/pts/*
```bash
# TTY devices owned by tty group
ls -l /dev/tty* | head -5
```

#### disk (GID 6)
**Purpose:** Direct disk access (dangerous)
**Warning:** Members can read/write raw disk devices
**Security Risk:** Can bypass file system permissions
```bash
# Disk devices (be careful!)
ls -l /dev/sd* /dev/nvme*
```

#### lp (GID 7)
**Purpose:** Line printer and printing system
**Services:** CUPS, printing daemons
**Devices:** Printer devices, print spools
```bash
# Printing related files
ls -l /dev/lp* 2>/dev/null || echo "No printer devices found"
```

#### mail (GID 8)
**Purpose:** Mail system access
**Services:** Postfix, Sendmail, mail delivery agents
**Files:** Mail spools, mail configuration
```bash
# Mail system files
ls -l /var/mail/ /var/spool/mail/ 2>/dev/null
```

#### news (GID 9)
**Purpose:** Network news system (NNTP)
**Usage:** News server software (rarely used today)
**Historical:** Legacy group from early Unix systems

#### uucp (GID 10)
**Purpose:** Unix-to-Unix Copy Protocol
**Usage:** Historical communication system
**Modern:** Rarely used, legacy group

### Hardware Access Groups

#### audio (GID varies)
**Purpose:** Audio device access
**Members:** Users who need to play/record audio
**Devices:** /dev/snd/*, ALSA devices
```bash
# Audio devices
ls -l /dev/snd/
getent group audio
```

#### video (GID varies)
**Purpose:** Video device access
**Members:** Users who need camera/video access
**Devices:** /dev/video*, webcams, video capture
```bash
# Video devices
ls -l /dev/video* 2>/dev/null
getent group video
```

#### input (GID varies)
**Purpose:** Input device access
**Devices:** Keyboards, mice, touchpads
**Files:** /dev/input/*
```bash
# Input devices
ls -l /dev/input/
```

#### dialout (GID varies)
**Purpose:** Serial port access
**Usage:** Modems, serial communications
**Devices:** /dev/ttyS*, /dev/ttyUSB*, /dev/ttyACM*
```bash
# Serial devices
ls -l /dev/ttyS* /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

#### cdrom (GID varies)
**Purpose:** CD-ROM/DVD access
**Devices:** Optical drives
**Usage:** Mount/access optical media
```bash
# Optical devices
ls -l /dev/cdrom /dev/dvd /dev/sr* 2>/dev/null
```

#### floppy (GID varies)
**Purpose:** Floppy disk access (historical)
**Devices:** /dev/fd*
**Modern Usage:** Mostly obsolete

#### tape (GID varies)
**Purpose:** Tape drive access
**Usage:** Backup systems, enterprise storage
**Devices:** /dev/st*, /dev/nst*

### Network and Communication Groups

#### netdev (GID varies)
**Purpose:** Network device management
**Usage:** Network configuration tools
**Permissions:** Configure network interfaces

#### plugdev (GID varies)
**Purpose:** Pluggable device access
**Usage:** USB devices, removable media
**Members:** Desktop users who need device access
```bash
# Check plugdev membership
getent group plugdev
```

#### bluetooth (GID varies)
**Purpose:** Bluetooth device access
**Services:** BlueZ bluetooth stack
**Permissions:** Bluetooth configuration and usage

### Desktop and User Groups

#### users (GID 100)
**Purpose:** General user group
**Usage:** Default group for regular users
**Permissions:** Basic user-level access

#### staff (GID 50)
**Purpose:** Staff users (system dependent)
**Usage:** Varies by distribution
**Permissions:** Often enhanced user privileges

#### games (GID varies)
**Purpose:** Game-related files and scores
**Files:** Game high scores, game data
**Directory:** /usr/games/, /var/games/

### Development and Compilation Groups

#### src (GID varies)
**Purpose:** Source code access
**Files:** System source code directories
**Usage:** Software development

### Docker and Container Groups

#### docker (GID varies)
**Purpose:** Docker daemon access
**Warning:** Equivalent to root access
**Usage:** Users who can run Docker containers
```bash
# Docker group members can effectively become root
getent group docker

# Docker socket permissions
ls -l /var/run/docker.sock
```

## Security Considerations

### Dangerous Groups

**Groups that provide root-equivalent access:**
- **docker:** Can mount host filesystem in containers
- **disk:** Direct disk access bypasses file permissions
- **sudo:** Can execute commands as root
- **wheel:** Traditional su access to root

**Example Docker privilege escalation:**
```bash
# User in docker group can become root
docker run -v /:/host -it ubuntu chroot /host bash
```

### Group Security Best Practices

#### 1. Principle of Least Privilege
```bash
# Only add users to necessary groups
sudo usermod -aG necessarygroup username

# Regular audit of group memberships
for group in sudo docker adm; do
    echo "=== $group group members ==="
    getent group $group
    echo
done
```

#### 2. Monitor Dangerous Groups
```bash
# Alert on dangerous group membership changes
sudo auditd -w /etc/group -p wa -k group_changes
sudo auditd -w /etc/gshadow -p wa -k group_changes

# Weekly group audit script
#!/bin/bash
for group in docker disk sudo wheel adm; do
    members=$(getent group $group 2>/dev/null | cut -d: -f4)
    if [ -n "$members" ]; then
        echo "WARNING: Users in dangerous group '$group': $members"
    fi
done
```

#### 3. Group Cleanup
```bash
# Remove users from unnecessary groups
sudo gpasswd -d username unnecessarygroup

# Find empty groups
getent group | awk -F: '$4 == "" {print $1}' | head -10

# Find groups with only one member
getent group | awk -F: 'gsub(/,/, "") == 0 && $4 != "" {print $1 ": " $4}'
```

## Common Group Management Tasks

### Setting Up Development Environment
```bash
# Create development group
sudo groupadd developers

# Add users to development group
sudo usermod -aG developers alice
sudo usermod -aG developers bob

# Create shared development directory
sudo mkdir /opt/projects
sudo chgrp developers /opt/projects
sudo chmod 2775 /opt/projects  # SetGID bit preserves group ownership

# Verify setup
ls -ld /opt/projects
getent group developers
```

### Web Server Setup
```bash
# Add user to web server group
sudo usermod -aG www-data username

# Set up web directory with proper group permissions
sudo mkdir -p /var/www/mysite
sudo chgrp www-data /var/www/mysite
sudo chmod g+s /var/www/mysite  # SetGID for new files
```

### Database Access
```bash
# Add user to database groups
sudo usermod -aG mysql dbadmin
sudo usermod -aG postgresql pgadmin

# Verify database group access
groups dbadmin
groups pgadmin
```

## Troubleshooting Group Issues

### Common Problems

#### User Can't Access Files/Devices
```bash
# Check user's group memberships
id username
groups username

# Check file/device group ownership
ls -l /path/to/file
ls -l /dev/device

# Add user to appropriate group
sudo usermod -aG requiredgroup username

# User must log out/in for group changes to take effect
# Or use newgrp command temporarily
newgrp requiredgroup
```

#### Permission Denied Despite Group Membership
```bash
# Check if user needs to re-login for group changes
# Current session groups
groups

# User's actual groups (after re-login)
id

# Force group refresh without logout (temporary)
exec sg groupname newshell
```

#### SetGID Not Working
```bash
# Check SetGID bit on directory
ls -ld /path/to/directory
# Should show 's' in group permission: drwxrws---

# Set SetGID bit
sudo chmod g+s /path/to/directory

# Verify new files inherit group
touch /path/to/directory/testfile
ls -l /path/to/directory/testfile
```

### Group Audit Commands

```bash
# Complete group membership report
echo "=== System Group Audit ==="
for user in $(cut -d: -f1 /etc/passwd | grep -v '^#'); do
    groups_list=$(groups $user 2>/dev/null | cut -d: -f2)
    if [ -n "$groups_list" ]; then
        echo "$user: $groups_list"
    fi
done

# Find users in administrative groups
for group in root sudo wheel admin docker; do
    members=$(getent group $group 2>/dev/null | cut -d: -f4)
    if [ -n "$members" ]; then
        echo "Admin group '$group': $members"
    fi
done

# Check for unusual high-GID groups
awk -F: '$3 > 65000 {print $1 ":" $3}' /etc/group
```

---

**Understanding Linux groups is essential for system security and user management. Proper group configuration enables secure collaboration while maintaining the principle of least privilege. Always audit group memberships regularly and be cautious when adding users to powerful groups like docker, disk, or sudo.**