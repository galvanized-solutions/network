# Linux User Management

## Understanding Linux User Management

Linux is a multi-user operating system where user management is fundamental to security and system organization. Every process, file, and resource is owned by a user, and permissions control access to system resources.

### How Linux Users Work

**User Types:**
- **Root (UID 0):** Superuser with unlimited system access
- **System Users (UID 1-999):** Service accounts for daemons and system processes
- **Regular Users (UID 1000+):** Human users with limited privileges
- **Groups:** Collections of users sharing permissions

**User Information Storage:**
- `/etc/passwd`: User account information (username, UID, GID, home directory, shell)
- `/etc/shadow`: Encrypted passwords and password policies
- `/etc/group`: Group definitions and memberships
- `/etc/gshadow`: Group passwords (rarely used)

**Authentication & Authorization:**
- **Authentication:** Verifying user identity (passwords, keys, tokens)
- **Authorization:** Determining what authenticated users can access
- **sudo:** Temporary privilege escalation mechanism
- **PAM (Pluggable Authentication Modules):** Flexible authentication framework

### Essential User Management Commands

#### User Information & Listing
```bash
# Display current user
whoami

# Show detailed user information
id [username]

# List all users
cat /etc/passwd | cut -d: -f1

# List users with home directories
ls /home

# Show currently logged-in users
who
w

# Display user's groups
groups [username]

# Show last login information
last [username]
lastlog
```

#### User Account Management
```bash
# Create new user (interactive)
sudo adduser username

# Create user (non-interactive)
sudo useradd -m -s /bin/bash username

# Set user password
sudo passwd username

# Change user's shell
sudo usermod -s /bin/bash username

# Lock user account
sudo passwd -l username

# Unlock user account
sudo passwd -u username

# Delete user (keep home directory)
sudo userdel username

# Delete user and home directory
sudo userdel -r username

# Force delete user (even if logged in)
sudo userdel -f username
```

#### Group Management
```bash
# Create new group
sudo groupadd groupname

# Add user to group
sudo usermod -aG groupname username

# Remove user from group
sudo gpasswd -d username groupname

# Change user's primary group
sudo usermod -g groupname username

# List group members
getent group groupname

# Delete group
sudo groupdel groupname
```

#### Password & Account Policies
```bash
# Check password status
sudo passwd -S username

# Set password expiration
sudo chage -M 90 username

# Force password change on next login
sudo chage -d 0 username

# Set account expiration date
sudo chage -E YYYY-MM-DD username

# Display password aging information
sudo chage -l username

# Lock account after failed attempts
sudo pam_tally2 --user=username --lock
```

#### Sudo & Privilege Management
```bash
# Check user's sudo privileges
sudo -l -U username

# Edit sudoers file safely
sudo visudo

# Check current user's sudo permissions
sudo -l

# Run command as another user
sudo -u username command

# Switch to another user
sudo su - username

# List sudo group members
getent group sudo
```

#### User Monitoring & Auditing
```bash
# Show failed login attempts
sudo grep "Failed password" /var/log/auth.log

# Monitor sudo usage
sudo tail -f /var/log/auth.log

# Check user login history
lastlog | grep username

# Display current user sessions
who -u

# Show processes by user
ps -u username

# Find files owned by user
find /path -user username

# Check user disk usage
du -sh /home/username
```

#### Advanced User Management
```bash
# Change user's home directory
sudo usermod -d /new/home/path -m username

# Add user to multiple groups
sudo usermod -aG group1,group2,group3 username

# Set user's full name/comment
sudo usermod -c "Full Name" username

# Create system user (for services)
sudo useradd -r -s /bin/false servicename

# Set user quotas (if quota enabled)
sudo setquota -u username soft_limit hard_limit 0 0 /filesystem

# Copy user account to another system
sudo getent passwd username >> /backup/passwd.backup
```
---

## User and Root Security Hardening

This guide focuses specifically on securing user accounts and root access on Raspberry Pi systems. This is a critical security foundation that should be implemented before any other services or automation.

## Overview

Raspberry Pi systems ship with dangerous default configurations that create multiple security vulnerabilities:
- Default `pi` user with unrestricted sudo access
- Weak or default passwords
- Dangerous sudoers configurations
- Insecure PAM settings
- Root account vulnerabilities

## Critical Security Issues

### 🚨 Confirmed Vulnerabilities

**Issue 1: `su -` Working Without Password**
- **Symptom:** `su -` works even when `sudo passwd -S root` shows "root NP" 
- **Root Cause:** PAM `pam_rootok.so` combined with unlimited sudo access
- **Risk Level:** CRITICAL - Complete system compromise

**Issue 2: Multiple NOPASSWD Files**
- **Symptom:** Multiple files in `/etc/sudoers.d/` with `(ALL) NOPASSWD: ALL`
- **Common Files:** `010_pi-nopasswd`, user-specific files
- **Risk Level:** CRITICAL - Unrestricted root access

**Issue 3: Default Pi User**
- **Symptom:** Default `pi` user with unlimited sudo access
- **File:** `/etc/sudoers.d/010_pi-nopasswd`
- **Risk Level:** HIGH - Well-known attack vector

## User Account Audit

### List Current Users
```bash
# List all users
cat /etc/passwd | cut -d: -f1

# List users with home directories (actual users)
ls /home

# Show users with sudo privileges
getent group sudo

# Show detailed user information
id username

# Check specific user's sudo permissions
sudo -l -U username
```

### Find Dangerous Configurations
```bash
# Find ALL dangerous sudo rules
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/

# List all sudoers files
ls -la /etc/sudoers.d/

# Check for specific dangerous files
ls -la /etc/sudoers.d/010_pi-nopasswd
ls -la /etc/sudoers.d/*nopasswd*

# Check default pi user status
id pi 2>/dev/null && echo "WARNING: Default 'pi' user exists!" || echo "Default 'pi' user: REMOVED (good)"

# Check your current sudo permissions
sudo -l
```

## Root Access Security

### Understanding the Vulnerability

**🚨 PAM rootok Vulnerability Explained:**

The vulnerability chain:
1. **Dangerous sudo:** `(ALL) NOPASSWD: ALL` makes user "effectively root"
2. **PAM configuration:** `auth sufficient pam_rootok.so` in `/etc/pam.d/su`
3. **Result:** `su -` works without password because PAM sees user as "already root"

### Diagnose PAM Configuration
```bash
# Check PAM configuration for su
cat /etc/pam.d/su

# USER'S CONFIRMED DANGEROUS CONFIGURATION:
# "auth sufficient pam_rootok.so"  <- Allows su if "effectively root"

# Check for other dangerous PAM modules:
grep -E "sufficient.*(pam_wheel.so trust|pam_permit.so|pam_group.so)" /etc/pam.d/su

# Check user group memberships
groups $(whoami)
id $(whoami)

# Check wheel group (if exists)
getent group wheel
```

### Emergency Sudo Cleanup

**🚨 CRITICAL: Remove All Dangerous NOPASSWD Rules**

```bash
# STEP 1: Backup everything
sudo mkdir -p /root/sudoers-backup-$(date +%Y%m%d)
sudo cp -r /etc/sudoers.d/* /root/sudoers-backup-$(date +%Y%m%d)/
echo "Backed up sudoers files to /root/sudoers-backup-$(date +%Y%m%d)/"

# STEP 2: Find all dangerous files
echo "=== DANGEROUS FILES DETECTED ==="
sudo find /etc/sudoers.d/ -type f -exec grep -l "NOPASSWD.*ALL" {} \;

# STEP 3: Handle specific Raspberry Pi default file
if [ -f /etc/sudoers.d/010_pi-nopasswd ]; then
    echo "🚨 FOUND: 010_pi-nopasswd (DEFAULT RASPBERRY PI VULNERABILITY)"
    echo "Contents:"
    sudo cat /etc/sudoers.d/010_pi-nopasswd
    echo "This file gives 'pi' user unrestricted root access!"
fi

# STEP 4: Remove ALL dangerous files
echo "=== REMOVING ALL DANGEROUS FILES ==="
sudo rm -f /etc/sudoers.d/010_pi-nopasswd
sudo rm -f /etc/sudoers.d/*nopasswd*
sudo rm -f /etc/sudoers.d/*NOPASSWD*

# Remove any remaining files with NOPASSWD ALL
for file in $(sudo find /etc/sudoers.d/ -type f -exec grep -l "NOPASSWD.*ALL" {} \; 2>/dev/null); do
    echo "Removing dangerous file: $file"
    sudo rm "$file"
done

# STEP 5: Create secure replacement configuration
USER=$(whoami)
cat << EOF | sudo tee /etc/sudoers.d/10-$USER-secure
# SECURE CONFIGURATION - Replaces dangerous (ALL) NOPASSWD: ALL

# Minimal required commands only (no password)
$USER ALL=(ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get install *, /usr/bin/apt-get upgrade *, /usr/bin/apt-get dist-upgrade *, /usr/bin/apt-get autoremove, /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl reload *, /usr/bin/systemctl enable *, /usr/bin/systemctl disable *, /usr/bin/systemctl status *, /usr/bin/ufw *, /bin/mkdir -p *, /bin/chown *, /bin/chmod *, /usr/bin/tee *, /bin/cp *

# REQUIRE PASSWORD for everything else
$USER ALL=(ALL) PASSWD: ALL

# Security restrictions
Defaults:$USER !visiblepw
Defaults:$USER always_set_home
Defaults:$USER env_reset
Defaults:$USER secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults:$USER logfile="/var/log/sudo.log"
Defaults:$USER log_input
Defaults:$USER log_output
EOF

# STEP 6: Clean main sudoers file if needed
echo "Check main sudoers file for dangerous rules:"
sudo grep "NOPASSWD.*ALL" /etc/sudoers || echo "No dangerous rules in main sudoers file"

# STEP 7: Verify cleanup worked
echo "=== VERIFICATION ==="
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/ || echo "SUCCESS: No dangerous NOPASSWD ALL rules found!"

# STEP 8: Test new configuration
sudo -l
echo "Your sudo permissions should show:"
echo "- Limited NOPASSWD commands (apt, systemctl, etc.)"
echo "- (ALL) PASSWD: ALL for everything else"
```

### Root Account Security

**Step-by-Step Root Hardening:**

```bash
# STEP 1: Check current root status
sudo passwd -S root
# Output meanings:
# P = Password set and unlocked
# L = Password set but locked  
# NP = No password set

# STEP 2: Choose security approach

# OPTION A: Maximum Security (Recommended)
sudo passwd root        # Set strong emergency password
sudo passwd -l root     # Lock the account

# OPTION B: High Security (No emergency password)
sudo passwd -d root     # Remove password
sudo passwd -l root     # Lock account

# OPTION C: Moderate (Basic locking)
sudo passwd -l root     # Just lock existing password

# STEP 3: Verify root is locked
sudo passwd -S root
# Should show "L" (locked)

# STEP 4: Test security
su - root              # Should FAIL with "Authentication failure"
sudo su -              # Should work (emergency access)

# STEP 5: Disable root SSH completely
sudo sed -i 's/#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh

# STEP 6: Verify SSH config
sudo grep "PermitRootLogin" /etc/ssh/sshd_config
# Should show "PermitRootLogin no"
```

### Fix PAM rootok Vulnerability

**If `su -` still works after sudo cleanup:**

```bash
# STEP 1: Backup PAM configuration
sudo cp /etc/pam.d/su /etc/pam.d/su.backup

# STEP 2: Create secure PAM configuration
sudo tee /etc/pam.d/su << 'EOF'
# Secure PAM configuration for su
auth       required   pam_unix.so
auth       required   pam_wheel.so use_uid
account    required   pam_unix.so
session    required   pam_unix.so
session    optional   pam_xauth.so
EOF

# STEP 3: Test that su - now fails
su -
# Should fail with "Authentication failure"

# STEP 4: Verify emergency access works
sudo su -
# Should work (proper emergency access using your sudo privileges)
```

## Default Pi User Removal

**Remove the dangerous default user:**

```bash
# STEP 1: Ensure you have another user with sudo access
getent group sudo
# Verify your user is in the sudo group

# STEP 2: Remove pi user (if it exists)
sudo userdel -r pi 2>/dev/null || echo "Pi user already removed"

# STEP 3: Remove pi's sudo access files
sudo rm -f /etc/sudoers.d/010_pi-nopasswd
sudo rm -f /etc/sudoers.d/pi

# STEP 4: Verify removal
id pi 2>/dev/null && echo "WARNING: Pi user still exists!" || echo "SUCCESS: Pi user removed"
```

## User Access Management

### Secure User Creation

**Create new users with proper restrictions:**

```bash
# Create new user
sudo adduser newuser

# Add to specific groups only (avoid sudo group unless needed)
sudo usermod -aG users newuser

# For administrative users, create restricted sudo access
cat << EOF | sudo tee /etc/sudoers.d/newuser
# Restricted sudo access for newuser
newuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *, /usr/bin/journalctl
newuser ALL=(ALL) PASSWD: /usr/bin/systemctl restart *, /usr/bin/systemctl stop *
EOF
```

### Audit Existing Users

```bash
# Review all users with sudo access
echo "=== SUDO GROUP MEMBERS ==="
getent group sudo

# Check each user's specific permissions
for user in $(getent group sudo | cut -d: -f4 | tr ',' ' '); do
    echo "=== $user sudo permissions ==="
    sudo -l -U $user 2>/dev/null || echo "No specific rules"
    echo
done

# Find users with dangerous permissions
echo "=== USERS WITH DANGEROUS PERMISSIONS ==="
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/ | grep -v "^#"
```

## Security Verification

### Complete Security Audit

```bash
echo "=== RASPBERRY PI SECURITY AUDIT ==="

echo "1. Dangerous sudo rules:"
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/ || echo "NONE FOUND (EXCELLENT!)"

echo "2. Files in sudoers.d directory:"
ls -la /etc/sudoers.d/

echo "3. Specific dangerous files:"
[ -f /etc/sudoers.d/010_pi-nopasswd ] && echo "❌ 010_pi-nopasswd exists" || echo "✅ 010_pi-nopasswd removed"
[ -f /etc/sudoers.d/pi ] && echo "❌ pi file exists" || echo "✅ pi file not found"

echo "4. Default pi user status:"
id pi 2>/dev/null && echo "❌ Default 'pi' user exists" || echo "✅ Default 'pi' user removed"

echo "5. Root account status:"
sudo passwd -S root | grep -q "L" && echo "✅ Root account locked" || echo "❌ Root account not locked"

echo "6. Root SSH access:"
grep -q "PermitRootLogin no" /etc/ssh/sshd_config && echo "✅ Root SSH disabled" || echo "❌ Root SSH not disabled"

echo "7. Current user sudo permissions:"
sudo -l

echo "8. PAM su configuration:"
grep -q "pam_rootok.so" /etc/pam.d/su && echo "⚠️  pam_rootok.so present" || echo "✅ No pam_rootok.so"

echo "9. Test su - (should fail):"
echo "Run manually: su -"
echo "Expected: Authentication failure"

echo "10. Test sudo su - (should work):"
echo "Run manually: sudo su -"
echo "Expected: Success (emergency access)"
```

### Ongoing Security Maintenance

**Daily Checks:**
```bash
# Check for unauthorized sudo changes
sudo find /etc/sudoers.d/ -newer /var/log/auth.log -ls

# Review recent sudo usage
sudo tail -20 /var/log/sudo.log 2>/dev/null || echo "Sudo logging not configured"

# Check for new users
tail -5 /etc/passwd
```

**Weekly Checks:**
```bash
# Full sudo audit
sudo grep -r "NOPASSWD" /etc/sudoers /etc/sudoers.d/

# User account audit
getent group sudo
for user in $(getent group sudo | cut -d: -f4 | tr ',' ' '); do
    echo "$user: $(sudo -l -U $user 2>/dev/null | grep -c NOPASSWD)"
done

# Root access attempts
grep "Failed password" /var/log/auth.log | grep root | tail -10
```

## Emergency Recovery

**If you lose sudo access:**

1. **Physical Access Recovery:**
   - Reboot Pi and interrupt boot
   - Add `init=/bin/bash` to kernel line
   - Mount filesystem read-write: `mount -o remount,rw /`
   - Fix sudoers: `visudo`

2. **SSH Key Recovery:**
   - If SSH keys still work, login and use `sudo`
   - Restore sudoers from backup

3. **Recovery User:**
   - Always maintain a second user with sudo access
   - Keep recovery user separate from daily-use accounts

## Security Best Practices

1. **Principle of Least Privilege:**
   - Grant minimum required permissions
   - Use specific commands instead of ALL
   - Regular permission audits

2. **Defense in Depth:**
   - Multiple security layers
   - No single point of failure
   - Comprehensive logging

3. **Regular Auditing:**
   - Weekly user permission reviews
   - Monthly security scans
   - Quarterly access reviews

4. **Incident Response:**
   - Document all changes
   - Maintain recovery procedures
   - Test backup access methods

---

**⚠️ CRITICAL REMINDER:**
Always test your configuration changes! Ensure you maintain emergency access before implementing restrictive security measures. Keep a second terminal open with sudo access while making changes.

This guide provides enterprise-grade security for Raspberry Pi user management while maintaining necessary administrative capabilities.