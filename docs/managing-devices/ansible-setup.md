# Device Management with Ansible

This guide shows how to set up Ansible on a control machine (laptop/desktop) to manage your Raspberry Pi remotely over your local network.

## Security Approach

This guide provides **three security options** for sudo access:

1. **Specific Commands Only** (Most Secure) - Allow passwordless access only to specific commands
2. **Timeout-based Sudo** (Moderate) - Cache sudo password for limited time  
3. **Ansible Vault Password** (Recommended) - Encrypt sudo password, prompt when needed

Choose the option that best fits your security requirements.

## Prerequisites

- Raspberry Pi with SSH enabled
- Control machine (laptop/desktop) on the same network
- Basic knowledge of terminal/command line

## Step 1: Prepare Your Raspberry Pi

### Enable and Harden SSH on Raspberry Pi
```bash
# On the Raspberry Pi
sudo systemctl enable ssh
sudo systemctl start ssh

# Verify SSH is running
sudo systemctl status ssh

# Create backup of SSH config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Apply comprehensive SSH hardening
sudo tee -a /etc/ssh/sshd_config << 'EOF'

# === SECURITY HARDENING ===
# Protocol and Encryption
Protocol 2
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,hmac-sha2-512
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

# Authentication Security
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Access Control
PermitRootLogin no
AllowUsers ansible
DenyUsers pi root
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30

# Network Security
ClientAliveInterval 300
ClientAliveCountMax 2
TCPKeepAlive no
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no

# Logging
SyslogFacility AUTH
LogLevel VERBOSE
EOF

# Test SSH configuration
sudo sshd -t

# Restart SSH service
sudo systemctl restart ssh

# Verify SSH is running with new config
sudo systemctl status ssh
```

### Create a dedicated user for Ansible (recommended)
```bash
# On the Raspberry Pi
sudo adduser ansible
sudo usermod -aG sudo ansible
```

### Configure Sudo Access (Choose One Method)

#### 🚨 CRITICAL: Fix Dangerous `(ALL) NOPASSWD: ALL` Configuration

**If your user has `(ALL) NOPASSWD: ALL` - THIS IS EXTREMELY DANGEROUS!**

```bash
# 1. IMMEDIATELY check your current sudo configuration
sudo -l
# If you see "(ALL) NOPASSWD: ALL" - you have unrestricted root access

# 2. Find ALL dangerous configurations (CRITICAL if multiple files)
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/
# This will show EVERY file with dangerous rules
# NOTE: Multiple files with same config = SECURITY DISASTER

# 3. List all sudoers files for cleanup
ls -la /etc/sudoers.d/
# You need to fix EVERY file that has dangerous rules

# 4. EMERGENCY CLEANUP - Multiple files with dangerous config
# First, identify your username and all dangerous files
USER=$(whoami)
echo "User: $USER"
echo "Files with dangerous NOPASSWD ALL rules:"
sudo grep -l "NOPASSWD.*ALL" /etc/sudoers.d/*

# 5. REMOVE ALL dangerous files (backup first)
sudo mkdir -p /root/sudoers-backup
sudo cp -r /etc/sudoers.d/* /root/sudoers-backup/
echo "Backed up all sudoers files to /root/sudoers-backup/"

# CRITICAL: Handle specific dangerous Raspberry Pi file
echo "=== CHECKING FOR DANGEROUS DEFAULT FILES ==="
if [ -f /etc/sudoers.d/010_pi-nopasswd ]; then
    echo "🚨 FOUND: 010_pi-nopasswd (DEFAULT RASPBERRY PI VULNERABILITY)"
    echo "Contents:"
    sudo cat /etc/sudoers.d/010_pi-nopasswd
    echo "This file gives 'pi' user unrestricted root access!"
else
    echo "010_pi-nopasswd not found (good)"
fi

# Remove ALL files with dangerous configurations
echo "=== REMOVING ALL DANGEROUS FILES ==="
# Specific dangerous files
sudo rm -f /etc/sudoers.d/010_pi-nopasswd
sudo rm -f /etc/sudoers.d/*nopasswd*
sudo rm -f /etc/sudoers.d/*NOPASSWD*

# Any remaining files with NOPASSWD ALL
for file in $(sudo find /etc/sudoers.d/ -type f -exec grep -l "NOPASSWD.*ALL" {} \; 2>/dev/null); do
    echo "Removing dangerous file: $file"
    sudo rm "$file"
done

echo "=== DANGEROUS FILE CLEANUP COMPLETE ==="

# 6. Create single secure configuration file
cat << EOF | sudo tee /etc/sudoers.d/$USER-secure
# SECURE CONFIGURATION - Replace dangerous (ALL) NOPASSWD: ALL

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

# 7. Clean main sudoers file
# CAREFULLY edit /etc/sudoers to remove any (ALL) NOPASSWD: ALL
sudo visudo
# Look for and DELETE these dangerous lines:
# username ALL=(ALL) NOPASSWD: ALL
# pi ALL=(ALL) NOPASSWD: ALL
# %wheel ALL=(ALL) NOPASSWD: ALL
# %sudo ALL=(ALL) NOPASSWD: ALL

# 8. Verify NO dangerous rules remain anywhere
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/
# This should return NOTHING - if it shows results, you missed some!

# 9. Test the cleaned configuration
sudo -l
# Should show ONLY:
# - Limited NOPASSWD commands (apt, systemctl, etc.)
# - (ALL) PASSWD: ALL for everything else
# - NO unrestricted NOPASSWD: ALL rules

# 10. Verify security works
sudo apt update        # Should work (no password)
sudo passwd root       # Should prompt for YOUR password
sudo su -             # Should work (emergency access)
su -                  # Should FAIL (if PAM was also fixed)

# 11. Final security audit
echo "=== FINAL SECURITY AUDIT ==="
echo "1. Dangerous sudo rules remaining:"
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/ || echo "NONE FOUND (EXCELLENT!)"

echo "2. Files in sudoers.d directory:"
ls -la /etc/sudoers.d/

echo "3. Checking for specific dangerous files:"
[ -f /etc/sudoers.d/010_pi-nopasswd ] && echo "WARNING: 010_pi-nopasswd still exists!" || echo "010_pi-nopasswd: REMOVED (good)"
[ -f /etc/sudoers.d/pi ] && echo "WARNING: pi file still exists!" || echo "pi file: not found (good)"

echo "4. Your current sudo permissions:"
sudo -l

echo "5. Testing restricted access:"
echo "   - apt update (should work without password)"
echo "   - passwd commands (should require password)"
echo "   - su - (should fail if PAM was fixed)"

echo "6. Default pi user status:"
id pi 2>/dev/null && echo "WARNING: Default 'pi' user still exists - consider removing!" || echo "Default 'pi' user: REMOVED (excellent!)"
```

#### Option 1: Specific Commands Only (Most Secure - RECOMMENDED)

List the password of specific users


```shell
sudo -l -U username
```

```bash
# Allow only specific required commands without password
cat << 'EOF' | sudo tee /etc/sudoers.d/ansible
# Minimal required commands only
ansible ALL=(ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get install *, /usr/bin/apt-get upgrade *, /usr/bin/apt-get dist-upgrade *, /usr/bin/apt-get autoremove, /usr/bin/systemctl start *, /usr/bin/systemctl stop *, /usr/bin/systemctl restart *, /usr/bin/systemctl reload *, /usr/bin/systemctl enable *, /usr/bin/systemctl disable *, /usr/bin/systemctl status *, /usr/bin/ufw *, /bin/mkdir -p *, /bin/chown *, /bin/chmod *, /usr/bin/tee *, /bin/cp *

# Require password for everything else
ansible ALL=(ALL) PASSWD: ALL

# Security restrictions
Defaults:ansible !visiblepw
Defaults:ansible always_set_home
Defaults:ansible env_reset
Defaults:ansible secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults:ansible logfile="/var/log/sudo.log"
Defaults:ansible log_input
Defaults:ansible log_output
EOF
```

#### Option 2: Timeout-based Sudo (Moderate Security)
```bash
# Require password but cache it for 15 minutes
cat << 'EOF' | sudo tee /etc/sudoers.d/ansible
Defaults:ansible timestamp_timeout=15
ansible ALL=(ALL) ALL
EOF
```

#### Option 3: Use Ansible Vault for Sudo Password (Recommended)
```bash
# Standard sudo access (will prompt for password)
# No sudoers modification needed - ansible user is already in sudo group
# We'll configure Ansible to use encrypted password
```

### User Management and Root Security

**🔒 IMPORTANT: Complete user and root security hardening is documented in a separate guide:**

📄 **See:** [`raspberry-pi-user-security.md`](./raspberry-pi-user-security.md)

This separate guide covers:
- Critical vulnerability fixes (su - working without password)
- Emergency sudo cleanup procedures  
- PAM security configuration
- Root account hardening
- Default pi user removal
- Complete security audit procedures

**Before proceeding with Ansible setup, you MUST secure your user accounts first.**

#### Quick User Status Check
```bash
# List all users
cat /etc/passwd | cut -d: -f1

# Show users with sudo privileges  
getent group sudo

# Check for dangerous sudo configurations
sudo grep -r "NOPASSWD.*ALL" /etc/sudoers /etc/sudoers.d/

# Check for dangerous default files
ls -la /etc/sudoers.d/010_pi-nopasswd
```

**⚠️ If the above commands show dangerous configurations, STOP and complete the user security hardening first!**

### Get your Raspberry Pi's IP address

```bash
# On the Raspberry Pi
hostname -I
# or
ip addr show
# Should show "PermitRootLogin no"
```

**Verification and testing procedures:**
```bash
# Test 1: Verify root cannot login directly with password
su - root
# Expected result: "Authentication failure"

# Test 2: Verify root cannot login via SSH (from another machine)
ssh root@your_pi_ip
# Expected result: "Permission denied"

# Test 3: Verify emergency access still works
sudo su -
# Expected result: Success (you become root)
whoami  # Should show "root"
exit

# Test 4: Check account status
sudo passwd -S root
# Expected result: Shows "L" (locked)

# Test 5: Verify sudo users can still gain root access
sudo -i
# Expected result: Success (alternative way to become root)
whoami  # Should show "root"  
exit

# Test 6: Check who has sudo privileges
getent group sudo
# Review this list - only trusted users should have sudo access
```

**Why this security approach works:**
```bash
# The key insight: Different authentication methods
# 1. Direct root login: Uses root's password (blocked when locked)
# 2. SSH root login: Controlled by SSH config (disabled separately) 
# 3. su - root: Uses root's password (blocked when locked)
# 4. sudo su -: Uses YOUR sudo privileges (still works - this is intentional)

# This design provides:
# - Protection against password-based attacks on root
# - Protection against SSH brute force on root account  
# - Emergency access for legitimate administrators
# - Audit trail (sudo commands are logged)
```

**🚨 EMERGENCY RESPONSE TO USER'S VULNERABILITY:**

**Your Specific Issue:** You report that `su -` works even though `sudo passwd -S root` shows `root NP`. This is abnormal and dangerous.

**IMMEDIATE INVESTIGATION COMMANDS:**
```bash
# 1. CONFIRMED: PAM rootok vulnerability (USER REPORTED)
# User found: "auth sufficient pam_rootok.so" in /etc/pam.d/su
# THIS IS THE ROOT CAUSE!

cat /etc/pam.d/su
# USER'S CONFIRMED DANGEROUS CONFIGURATION:
# "auth sufficient pam_rootok.so"          <- IDENTIFIED: Allows su if "effectively root"
#                                              Combined with NOPASSWD: ALL = passwordless su

# OTHER DANGEROUS CONFIGURATIONS TO ALSO CHECK:
# "auth sufficient pam_wheel.so trust"     <- wheel group can su without password
# "auth sufficient pam_permit.so"          <- everyone can su without password
# "auth sufficient pam_group.so"           <- group-based authentication

# THE VULNERABILITY CHAIN EXPLAINED:
# 1. Your sudoers has (ALL) NOPASSWD: ALL
# 2. This gives you "effective root" privileges
# 3. pam_rootok.so checks if user is "effectively root"
# 4. Since you are, it allows su - without password
# 5. RESULT: su - works even with root NP status

# 2. Check your group memberships
groups $(whoami)
# If you see "wheel" in the output, this likely explains the vulnerability

# 3. Check wheel group configuration
getent group wheel
cat /etc/group | grep wheel

# 4. IMMEDIATE FIX - Secure PAM configuration
sudo cp /etc/pam.d/su /etc/pam.d/su.backup
sudo tee /etc/pam.d/su << 'EOF'
auth       required   pam_unix.so
auth       required   pam_wheel.so use_uid
account    required   pam_unix.so
session    required   pam_unix.so
session    optional   pam_xauth.so
EOF

# 5. Lock root account
sudo passwd -l root

# 6. CRITICAL TEST
su -
# Should now fail with "Authentication failure"

# 7. Verify emergency access still works
sudo su -
# Should work (proper emergency access)
```

**Best practices for root account management:**

1. **Account Locking Strategy:**
   ```bash
   # Always lock the root account after securing it
   sudo passwd -l root
   
   # Never leave root unlocked in production
   # Exception: Single-user recovery mode scenarios
   ```

2. **Sudo Access Management:**
   ```bash
   # Regularly audit sudo users
   getent group sudo
   
   # Remove sudo access from unnecessary users
   sudo deluser username sudo
   
   # Check individual user sudo permissions
   sudo -l -U username
   ```

3. **Monitoring and Logging:**
   ```bash
   # Enable comprehensive sudo logging
   echo 'Defaults logfile="/var/log/sudo.log"' | sudo tee -a /etc/sudoers.d/logging
   echo 'Defaults log_input,log_output' | sudo tee -a /etc/sudoers.d/logging
   
   # Monitor sudo usage
   sudo tail -f /var/log/sudo.log
   
   # Review recent root access
   sudo grep "sudo" /var/log/auth.log | tail -20
   ```

4. **Emergency Access Procedures:**
   ```bash
   # Method 1: Unlock root temporarily (if you set a password)
   sudo passwd -u root
   # Use root access as needed
   sudo passwd -l root  # Re-lock immediately
   
   # Method 2: Use existing sudo privileges
   sudo su -  # Recommended - maintains audit trail
   
   # Method 3: Physical access recovery
   # Boot to single user mode by editing GRUB:
   # Add 'single' or 'init=/bin/bash' to kernel parameters
   ```

5. **Security Validation Checklist:**
   ```bash
   # Run this checklist monthly
   echo "=== ROOT SECURITY AUDIT ==="
   
   # 1. Check root account status
   echo "Root account status:"
   sudo passwd -S root
   
   # 2. Verify SSH root login disabled
   echo "SSH root login setting:"
   sudo grep "PermitRootLogin" /etc/ssh/sshd_config
   
   # 3. Test direct root login (should fail)
   echo "Testing direct root login (should fail):"
   timeout 3 su - root || echo "GOOD: Root login blocked"
   
   # 4. Verify emergency access works
   echo "Testing emergency access:"
   sudo -n true && echo "GOOD: Sudo access available" || echo "WARNING: No sudo access"
   
   # 5. Check sudo group membership
   echo "Users with sudo access:"
   getent group sudo
   
   # 6. Review recent sudo usage
   echo "Recent sudo activity:"
   sudo tail -5 /var/log/auth.log | grep sudo || echo "No recent sudo activity"
   ```

**Common misconceptions and corrections:**

| Misconception | Reality | Correct Approach |
|--------------|---------|------------------|
| "Deleting root password prevents all access" | `sudo su -` still works | Lock account with `passwd -l` |
| "No root password = secure" | Sudo users can still become root | Audit sudo access regularly |
| "`passwd -d` makes system safer" | Only removes password authentication | Use `passwd -l` after setting/removing password |
| "Root SSH disabled = root secured" | Local access still possible | Disable SSH + lock account + audit sudo |
| "Only need to secure SSH" | Console/local access still works | Implement multi-layer security |

**Summary of proper root security:**

✅ **DO THIS:**
```bash
# Complete root security implementation
sudo passwd root          # Set emergency password
sudo passwd -l root        # Lock the account  
# Edit SSH config to have PermitRootLogin no
sudo systemctl restart ssh # Apply SSH changes
sudo passwd -S root        # Verify shows "L"
```

❌ **DON'T DO THIS:**
```bash
# Incomplete/ineffective approaches
sudo passwd -d root        # Alone - doesn't prevent sudo su -
# Leaving SSH root login enabled  
# Not auditing sudo access
# Assuming password deletion = security
```

**Quick verification command:**
```bash
# One-liner to check your root security status
echo "Root Status: $(sudo passwd -S root | cut -d' ' -f2), SSH Root: $(sudo grep PermitRootLogin /etc/ssh/sshd_config | grep -v '^#'), Sudo Test: $(sudo -n true 2>/dev/null && echo 'Available' || echo 'Blocked')"
```

#### Additional security hardening
```bash
# Disable unused user accounts
sudo usermod -L -s /bin/false username

# Remove user from sudo group
sudo deluser username sudo

# Check for users with empty passwords
sudo awk -F: '($2 == "" ) { print $1 }' /etc/shadow

# Set password expiry for users
sudo chage -M 90 username  # Password expires in 90 days
sudo chage -l username     # List password info
```

#### Ansible playbook for user security
```yaml
# Create user-security.yml
cat << 'EOF' > user-security.yml
---
- name: Secure User Accounts
  hosts: raspberrypi
  become: yes
  vars_prompt:
    - name: set_root_password
      prompt: "Set a root password before locking? (y/n)"
      default: "n"
      private: no

    - name: root_password
      prompt: "Enter root password (if setting one)"
      private: yes
      when: set_root_password == "y"

  tasks:
    - name: Set root password (if requested)
      user:
        name: root
        password: "{{ root_password | password_hash('sha512') }}"
      when: set_root_password == "y" and root_password is defined

    - name: Lock root account
      user:
        name: root
        password_lock: yes

    - name: Check root password status
      command: passwd -S root
      register: root_status
      changed_when: false

    - name: Display root account status
      debug:
        msg: "Root account status: {{ root_status.stdout }}"

    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        backup: yes
      notify: restart ssh

    - name: Disable password authentication (SSH key only)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        backup: yes
      notify: restart ssh

    - name: Check for users with empty passwords
      shell: "awk -F: '($2 == \"\" ) { print $1 }' /etc/shadow"
      register: empty_password_users
      changed_when: false

    - name: Report users with empty passwords
      debug:
        msg: "Users with empty passwords: {{ empty_password_users.stdout_lines }}"
      when: empty_password_users.stdout_lines | length > 0

  handlers:
    - name: restart ssh
      systemd:
        name: ssh
        state: restarted
EOF
```

### Get your Raspberry Pi's IP address
```bash
# On the Raspberry Pi
hostname -I
# or
ip addr show
```

### Configure Firewall (CRITICAL)
```bash
# On the Raspberry Pi - Configure UFW firewall
# Reset to clean state
sudo ufw --force reset

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH from local networks only
sudo ufw allow from 192.168.0.0/16 to any port 22
sudo ufw allow from 10.0.0.0/8 to any port 22
sudo ufw allow from 172.16.0.0/12 to any port 22

# Enable logging
sudo ufw logging on

# Enable firewall
sudo ufw --force enable

# Check status
sudo ufw status verbose
```

### Install and Configure Fail2ban
```bash
# Install fail2ban for intrusion detection
sudo apt update
sudo apt install fail2ban

# Configure fail2ban for SSH protection
sudo tee /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
EOF

# Start and enable fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Check fail2ban status
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

## Step 2: Install Ansible on Control Machine

### On macOS
```bash
# Using Homebrew
brew install ansible

# Using pip
pip3 install ansible
```

### On Ubuntu/Debian
```bash
sudo apt update
sudo apt install ansible
```

### On CentOS/RHEL/Fedora
```bash
# CentOS/RHEL
sudo yum install epel-release
sudo yum install ansible

# Fedora
sudo dnf install ansible
```

### Verify Installation
```bash
ansible --version
```

## Step 3: Set Up SSH Key Authentication

### Generate SSH key pair on control machine
```bash
# Generate SSH key (if you don't have one)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Copy public key to Raspberry Pi
ssh-copy-id ansible@192.168.x.x  # Replace with your Pi's IP

# Test SSH connection
ssh ansible@192.168.x.x
```

### Alternative: Manual key copy
```bash
# If ssh-copy-id doesn't work
cat ~/.ssh/id_rsa.pub | ssh ansible@192.168.x.x "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## Step 4: Create Ansible Project Structure

### Create project directory
```bash
mkdir raspberry-pi-ansible
cd raspberry-pi-ansible
```

### Create inventory file
```bash
# Create inventory.ini
cat << EOF > inventory.ini
[raspberrypi]
pi-1 ansible_host=192.168.x.x ansible_user=ansible

[raspberrypi:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
```

### Create ansible.cfg

#### For Option 1 (Specific Commands) or Option 2 (Timeout-based):
```bash
cat << EOF > ansible.cfg
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ansible
private_key_file = ~/.ssh/id_rsa

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
EOF
```

#### For Option 3 (Ansible Vault Password):
```bash
cat << EOF > ansible.cfg
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ansible
private_key_file = ~/.ssh/id_rsa
ask_become_pass = True

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
EOF
```

## Step 5: Test Ansible Connection

### Test basic connectivity
```bash
# Ping test
ansible all -m ping

# Expected output:
# pi-1 | SUCCESS => {
#     "ansible_facts": {
#         "discovered_interpreter_python": "/usr/bin/python3"
#     },
#     "changed": false,
#     "ping": "pong"
# }
```

### Run ad-hoc commands

#### For Options 1 & 2 (passwordless specific commands or timeout):
```bash
# Check uptime
ansible all -m command -a "uptime"

# Check disk space
ansible all -m command -a "df -h"

# Install package (will work without password prompt)
ansible all -m apt -a "name=htop state=present" --become
```

#### For Option 3 (with sudo password):
```bash
# Check uptime
ansible all -m command -a "uptime"

# Check disk space
ansible all -m command -a "df -h"

# Install package (will prompt for sudo password)
ansible all -m apt -a "name=htop state=present" --become --ask-become-pass

# Or create encrypted password file
echo 'your_sudo_password' | ansible-vault encrypt_string --stdin-name 'ansible_become_password'
# Add the output to group_vars/all.yml and use --ask-vault-pass
```

## Step 6: Create Your First Playbook

### Comprehensive Security Hardening Playbook
```yaml
# Create comprehensive-security.yml
cat << 'EOF' > comprehensive-security.yml
---
- name: Comprehensive Raspberry Pi Security Hardening
  hosts: raspberrypi
  become: yes
  vars:
    allowed_networks: "192.168.0.0/16"
    ssh_port: 22
    fail2ban_bantime: 3600
    fail2ban_maxretry: 3
    
  tasks:
    # System Updates
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Upgrade all packages
      apt:
        upgrade: dist

    # Security packages
    - name: Install security packages
      apt:
        name:
          - fail2ban
          - rkhunter
          - chkrootkit
          - aide
          - auditd
          - unattended-upgrades
          - ufw
          - lynis
        state: present

    # SSH Hardening
    - name: Configure SSH security settings
      blockinfile:
        path: /etc/ssh/sshd_config
        block: |
          # Security Configuration
          Protocol 2
          Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
          MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
          KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
          
          # Authentication
          PasswordAuthentication no
          PermitEmptyPasswords no
          ChallengeResponseAuthentication no
          PermitRootLogin no
          AllowUsers ansible
          MaxAuthTries 3
          MaxSessions 2
          LoginGraceTime 30
          
          # Network Security
          ClientAliveInterval 300
          ClientAliveCountMax 2
          X11Forwarding no
          AllowTcpForwarding no
          AllowAgentForwarding no
          
          # Logging
          SyslogFacility AUTH
          LogLevel VERBOSE
        backup: yes
      notify: restart ssh

    # Firewall Configuration
    - name: Reset UFW to defaults
      ufw:
        state: reset

    - name: Set UFW default policies
      ufw:
        direction: "{{ item.direction }}"
        policy: "{{ item.policy }}"
      loop:
        - { direction: 'incoming', policy: 'deny' }
        - { direction: 'outgoing', policy: 'allow' }

    - name: Allow SSH from local networks
      ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp
        src: "{{ item }}"
      loop:
        - 192.168.0.0/16
        - 10.0.0.0/8
        - 172.16.0.0/12

    - name: Enable UFW logging
      ufw:
        logging: on

    - name: Enable UFW
      ufw:
        state: enabled

    # Fail2ban Configuration
    - name: Configure fail2ban for SSH
      ini_file:
        path: /etc/fail2ban/jail.local
        section: sshd
        option: "{{ item.option }}"
        value: "{{ item.value }}"
        backup: yes
      loop:
        - { option: 'enabled', value: 'true' }
        - { option: 'port', value: "{{ ssh_port }}" }
        - { option: 'filter', value: 'sshd' }
        - { option: 'logpath', value: '/var/log/auth.log' }
        - { option: 'maxretry', value: "{{ fail2ban_maxretry }}" }
        - { option: 'bantime', value: "{{ fail2ban_bantime }}" }
      notify: restart fail2ban

    # System Hardening
    - name: Apply kernel security parameters
      sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        reload: yes
        sysctl_file: /etc/sysctl.d/99-security.conf
      loop:
        - { name: 'net.ipv4.conf.all.send_redirects', value: '0' }
        - { name: 'net.ipv4.conf.all.accept_redirects', value: '0' }
        - { name: 'net.ipv4.conf.all.accept_source_route', value: '0' }
        - { name: 'net.ipv4.conf.all.log_martians', value: '1' }
        - { name: 'net.ipv4.icmp_echo_ignore_broadcasts', value: '1' }
        - { name: 'net.ipv4.icmp_ignore_bogus_error_responses', value: '1' }
        - { name: 'net.ipv4.conf.all.rp_filter', value: '1' }
        - { name: 'net.ipv4.tcp_syncookies', value: '1' }
        - { name: 'kernel.dmesg_restrict', value: '1' }
        - { name: 'kernel.kptr_restrict', value: '2' }
        - { name: 'fs.suid_dumpable', value: '0' }

    # Automatic Security Updates
    - name: Configure unattended upgrades
      lineinfile:
        path: /etc/apt/apt.conf.d/20auto-upgrades
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        create: yes
      loop:
        - { regexp: '^APT::Periodic::Update-Package-Lists', line: 'APT::Periodic::Update-Package-Lists "1";' }
        - { regexp: '^APT::Periodic::Unattended-Upgrade', line: 'APT::Periodic::Unattended-Upgrade "1";' }
        - { regexp: '^APT::Periodic::AutocleanInterval', line: 'APT::Periodic::AutocleanInterval "7";' }

    # File Integrity Monitoring
    - name: Initialize AIDE database
      command: aide --init
      args:
        creates: /var/lib/aide/aide.db.new

    - name: Move AIDE database
      command: mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
      args:
        creates: /var/lib/aide/aide.db

    # Remove default user
    - name: Remove default pi user
      user:
        name: pi
        state: absent
        remove: yes
      failed_when: false

    # Disable unnecessary services
    - name: Disable unnecessary services
      systemd:
        name: "{{ item }}"
        state: stopped
        enabled: no
      loop:
        - bluetooth
        - avahi-daemon
        - triggerhappy
      failed_when: false

    # Start security services
    - name: Start and enable security services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - auditd
        - fail2ban

    # Security scan setup
    - name: Setup daily security scan
      cron:
        name: "Daily security scan"
        minute: "0"
        hour: "2"
        job: "/usr/bin/lynis audit system --quiet --cronjob > /var/log/lynis-daily.log 2>&1"

  handlers:
    - name: restart ssh
      systemd:
        name: ssh
        state: restarted

    - name: restart fail2ban
      systemd:
        name: fail2ban
        state: restarted
EOF
```

### Basic system update playbook
```yaml
# Create update-system.yml
cat << 'EOF' > update-system.yml
---
- name: Update Raspberry Pi System
  hosts: raspberrypi
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Upgrade all packages
      apt:
        upgrade: dist

    - name: Install essential packages
      apt:
        name:
          - curl
          - wget
          - git
          - vim
          - htop
          - tree
        state: present

    - name: Remove unnecessary packages
      apt:
        autoremove: yes

    - name: Check if reboot is required
      stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Notify about reboot
      debug:
        msg: "Reboot required!"
      when: reboot_required.stat.exists
EOF
```

### PostgreSQL installation playbook
```yaml
# Create postgres-setup.yml
cat << 'EOF' > postgres-setup.yml
---
- name: Install and Configure PostgreSQL
  hosts: raspberrypi
  become: yes
  vars:
    postgres_version: "15"
    postgres_password: "your_secure_password"
    allowed_networks: "192.168.0.0/16"

  tasks:
    - name: Install PostgreSQL
      apt:
        name:
          - postgresql
          - postgresql-contrib
          - python3-psycopg2
        state: present

    - name: Start and enable PostgreSQL
      systemd:
        name: postgresql
        state: started
        enabled: yes

    - name: Configure PostgreSQL to listen on all addresses
      lineinfile:
        path: "/etc/postgresql/{{ postgres_version }}/main/postgresql.conf"
        regexp: "^#?listen_addresses"
        line: "listen_addresses = '*'"
        backup: yes
      notify: restart postgresql

    - name: Configure pg_hba.conf for network access
      lineinfile:
        path: "/etc/postgresql/{{ postgres_version }}/main/pg_hba.conf"
        line: "host    all             all             {{ allowed_networks }}         md5"
        backup: yes
      notify: restart postgresql

    - name: Set PostgreSQL user password
      become_user: postgres
      postgresql_user:
        name: postgres
        password: "{{ postgres_password }}"

    - name: Open PostgreSQL port in UFW
      ufw:
        rule: allow
        port: "5432"
        proto: tcp
        src: "{{ allowed_networks }}"

  handlers:
    - name: restart postgresql
      systemd:
        name: postgresql
        state: restarted
EOF
```

## Step 7: Run Playbooks

### Execute comprehensive security hardening (RUN FIRST)
```bash
ansible-playbook comprehensive-security.yml
```

### Execute system update
```bash
ansible-playbook update-system.yml
```

### Execute PostgreSQL setup
```bash
ansible-playbook postgres-setup.yml
```

### Run with specific tags or limits
```bash
# Run only on specific host
ansible-playbook update-system.yml --limit pi-1

# Check what would change (dry run)
ansible-playbook postgres-setup.yml --check

# Verbose output
ansible-playbook update-system.yml -v
```

## Step 8: Advanced Configuration

### Group variables
```bash
# Create group_vars directory
mkdir group_vars

# Create group_vars/raspberrypi.yml
cat << 'EOF' > group_vars/raspberrypi.yml
---
# Common variables for all Raspberry Pis
timezone: "America/New_York"
swap_size: 1024
enable_i2c: true
enable_spi: true

# PostgreSQL settings
postgres_version: "15"
postgres_max_connections: 100
postgres_shared_buffers: "128MB"
EOF
```

### Host-specific variables
```bash
# Create host_vars directory
mkdir host_vars

# Create host_vars/pi-1.yml
cat << 'EOF' > host_vars/pi-1.yml
---
# Specific settings for pi-1
postgres_password: "pi1_secure_password"
additional_packages:
  - nginx
  - nodejs
  - npm
EOF
```

### Roles structure
```bash
# Create roles directory
mkdir -p roles/common/tasks
mkdir -p roles/postgres/tasks
mkdir -p roles/postgres/handlers
mkdir -p roles/postgres/templates

# Example role task
cat << 'EOF' > roles/common/tasks/main.yml
---
- name: Update system packages
  apt:
    update_cache: yes
    upgrade: dist

- name: Install common packages
  apt:
    name: "{{ common_packages }}"
    state: present
EOF
```

## Step 9: Useful Commands and Tips

### Inventory management
```bash
# List all hosts
ansible all --list-hosts

# View inventory variables
ansible-inventory --list

# Test specific group
ansible raspberrypi -m setup
```

### Debugging
```bash
# Debug connection issues
ansible all -m setup -vvv

# Check gathered facts
ansible all -m setup | grep ansible_distribution

# Test with different user
ansible all -m ping -u pi
```

### Security considerations

#### Method 1: Encrypt sudo password with Ansible Vault
```bash
# Create encrypted password file
mkdir -p group_vars
ansible-vault create group_vars/all.yml

# Add this content when prompted:
# ansible_become_password: your_actual_sudo_password

# Run playbooks with vault password
ansible-playbook postgres-setup.yml --ask-vault-pass
```

#### Method 2: Use environment variable for vault password
```bash
# Create vault password file (secure this file!)
echo 'your_vault_password' > .vault_pass
chmod 600 .vault_pass

# Add to ansible.cfg
echo 'vault_password_file = .vault_pass' >> ansible.cfg

# Add .vault_pass to .gitignore
echo '.vault_pass' >> .gitignore
```

#### Method 3: Encrypt individual variables
```bash
# Encrypt single variables
ansible-vault encrypt_string 'your_sudo_password' --name 'ansible_become_password'

# Add output to group_vars/all.yml
```

#### System Hardening Configuration
```bash
# Apply kernel security parameters
sudo tee /etc/sysctl.d/99-security.conf << 'EOF'
# Network security
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1

# Process security
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1

# File system security
fs.suid_dumpable = 0
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
EOF

# Apply settings
sudo sysctl -p /etc/sysctl.d/99-security.conf

# Disable unnecessary services
sudo systemctl disable bluetooth
sudo systemctl disable avahi-daemon
sudo systemctl disable triggerhappy

# Install security tools
sudo apt install aide rkhunter chkrootkit auditd unattended-upgrades

# Configure automatic security updates
sudo tee /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
EOF
```

#### General security practices
```bash
# Always encrypt sensitive variables
ansible-vault create secrets.yml
ansible-vault edit secrets.yml

# Use specific sudo commands when possible
# Regularly rotate passwords
# Use SSH key authentication only
# Keep Ansible control machine secure
# Monitor logs regularly
# Run security audits with: sudo lynis audit system
```

## Step 10: Automation and Scheduling

### Create deployment script
```bash
cat << 'EOF' > deploy.sh
#!/bin/bash
set -e

echo "Deploying to Raspberry Pi..."

# Test connectivity
ansible all -m ping

# Run system updates
ansible-playbook update-system.yml

# Deploy applications
ansible-playbook postgres-setup.yml

echo "Deployment complete!"
EOF

chmod +x deploy.sh
```

### Schedule regular updates (optional)
```bash
# Add to crontab on control machine
# Run every Sunday at 2 AM
0 2 * * 0 /path/to/raspberry-pi-ansible/deploy.sh >> /var/log/ansible-deploy.log 2>&1
```

## Troubleshooting

### Common issues and solutions

**SSH connection refused:**
```bash
# Check SSH service on Pi
ssh ansible@192.168.x.x "sudo systemctl status ssh"
```

**Permission denied:**
```bash
# Verify sudo access
ssh ansible@192.168.x.x "sudo whoami"

# Check sudoers file
ssh ansible@192.168.x.x "sudo cat /etc/sudoers.d/ansible"
```

**Python interpreter issues:**
```bash
# Specify Python path in inventory
ansible_python_interpreter=/usr/bin/python3
```

**Firewall blocking connections:**
```bash
# Check if UFW is blocking SSH
ansible all -m command -a "sudo ufw status" --ask-become-pass
```

## Security Monitoring and Maintenance

### Daily Security Checks
```bash
# Check fail2ban status
ansible all -m command -a "sudo fail2ban-client status sshd"

# Check firewall status
ansible all -m command -a "sudo ufw status verbose"

# Review security logs
ansible all -m command -a "sudo tail -20 /var/log/auth.log"

# Run security audit
ansible all -m command -a "sudo lynis audit system --quick"
```

### Weekly Security Tasks
```bash
# File integrity check
ansible all -m command -a "sudo aide --check"

# Rootkit scan
ansible all -m command -a "sudo rkhunter --check --skip-keypress"

# Check for security updates
ansible all -m command -a "apt list --upgradable | grep -i security"
```

### Security Incident Response
```bash
# If compromise suspected:
# 1. Check active connections
ansible all -m command -a "ss -tuln"
ansible all -m command -a "who -a"

# 2. Check recent logins
ansible all -m command -a "last -20"

# 3. Check processes
ansible all -m command -a "ps aux --sort=-%cpu | head -20"

# 4. Check fail2ban logs
ansible all -m command -a "sudo tail -50 /var/log/fail2ban.log"
```

## Next Steps

### Security Priority Order:
1. **CRITICAL**: Run comprehensive-security.yml playbook first
2. **HIGH**: Set up monitoring and log review procedures
3. **MEDIUM**: Implement network segmentation if possible
4. **LOW**: Advanced monitoring and alerting

### Ongoing Security Tasks:
1. Weekly security scans with Lynis
2. Monthly review of fail2ban logs
3. Quarterly password rotation
4. Regular backup testing
5. Monitor security advisories for installed packages

This hardened setup provides enterprise-grade security for your Raspberry Pi while maintaining Ansible automation capabilities!