# Nmap Cheat Sheet

## Basic Scanning

### Host Discovery
```bash
# Ping scan (discover live hosts)
nmap -sn 192.168.1.0/24

# Skip host discovery (treat all hosts as online)
nmap -Pn target

# ARP ping (local network)
nmap -PR 192.168.1.0/24
```

### Port Scanning
```bash
# TCP SYN scan (default)
nmap -sS target

# TCP connect scan
nmap -sT target

# UDP scan
nmap -sU target

# Scan specific ports
nmap -p 22,80,443 target

# Scan port ranges
nmap -p 1-1000 target

# Scan all ports
nmap -p- target

# Top 1000 ports (default)
nmap --top-ports 1000 target
```

### Scan Types
```bash
# TCP SYN stealth scan
nmap -sS target

# TCP ACK scan (firewall detection)
nmap -sA target

# TCP Window scan
nmap -sW target

# TCP Maimon scan
nmap -sM target
```

## Advanced Scanning

### Service Detection
```bash
# Service version detection
nmap -sV target

# OS detection
nmap -O target

# Aggressive scan (OS, version, script, traceroute)
nmap -A target

# Script scanning
nmap -sC target
nmap --script=default target
```

### Timing and Performance
```bash
# Timing templates (0-5, paranoid to insane)
nmap -T4 target

# Custom timing
nmap --min-rate 1000 target
nmap --max-rate 5000 target

# Parallel host scanning
nmap --min-hostgroup 50 target
```

### Output Options
```bash
# Normal output
nmap -oN scan.txt target

# XML output
nmap -oX scan.xml target

# Grepable output
nmap -oG scan.gnmap target

# All formats
nmap -oA scan target
```

### Evasion Techniques
```bash
# Fragment packets
nmap -f target

# Decoy scanning
nmap -D RND:10 target

# Source port spoofing
nmap --source-port 53 target

# Randomize target order
nmap --randomize-hosts target

# Spoof MAC address
nmap --spoof-mac 0 target
```

## NSE Scripts

### Common Script Categories
```bash
# Default scripts
nmap --script=default target

# Vulnerability scripts
nmap --script=vuln target

# Authentication scripts
nmap --script=auth target

# Brute force scripts
nmap --script=brute target

# Discovery scripts
nmap --script=discovery target

# Safe scripts
nmap --script=safe target
```

### Specific Useful Scripts
```bash
# HTTP enumeration
nmap --script=http-enum target

# SMB enumeration
nmap --script=smb-enum-shares target

# SSL certificate info
nmap --script=ssl-cert target

# DNS enumeration
nmap --script=dns-brute target

# Check for common vulnerabilities
nmap --script=vulners target
```

## Target Specification

```bash
# Single IP
nmap 192.168.1.1

# Multiple IPs
nmap 192.168.1.1,2,3

# IP range
nmap 192.168.1.1-10

# CIDR notation
nmap 192.168.1.0/24

# Hostname
nmap example.com

# Input from file
nmap -iL targets.txt

# Exclude hosts
nmap 192.168.1.0/24 --exclude 192.168.1.1
```

## Common Scan Combinations

### Quick Network Discovery
```bash
# Fast network sweep
nmap -sn --min-rate 5000 192.168.1.0/24

# Quick port scan of live hosts
nmap -sS -T4 --top-ports 1000 192.168.1.0/24
```

### Comprehensive Host Scan
```bash
# Full TCP port scan with service detection
nmap -sS -sV -O -A -T4 -p- target

# UDP scan of common ports
nmap -sU -sV --top-ports 1000 target
```

### Stealth Scanning
```bash
# Slow stealth scan
nmap -sS -T1 -f -D RND:10 target

# Firewall evasion
nmap -sA -T2 --source-port 53 target
```

---

# Useful Nmap Aliases for .zshrc

Add these aliases to your `~/.zshrc` file:

```bash
# Basic nmap aliases
alias nmap-quick='nmap -T4 -F'
alias nmap-intense='nmap -T4 -A -v'
alias nmap-comprehensive='nmap -sS -sV -O -A -T4 -p-'

# Host discovery
alias nmap-ping='nmap -sn'
alias nmap-discover='nmap -sn --min-rate 5000'

# Port scanning
alias nmap-tcp='nmap -sS -T4'
alias nmap-udp='nmap -sU --top-ports 1000'
alias nmap-all-ports='nmap -p- -T4'

# Service detection
alias nmap-services='nmap -sV -T4'
alias nmap-os='nmap -O -T4'

# Script scanning
alias nmap-vuln='nmap --script=vuln -T4'
alias nmap-safe='nmap --script=safe -T4'
alias nmap-default='nmap -sC -sV -T4'

# Stealth scanning
alias nmap-stealth='nmap -sS -T1 -f'
alias nmap-decoy='nmap -sS -D RND:10 -T4'

# Network ranges
alias nmap-local='nmap -sS -T4 192.168.1.0/24'
alias nmap-class-c='function _nmap_class_c(){ nmap -sS -T4 "$1".0/24 }; _nmap_class_c'

# Output options
alias nmap-save='function _nmap_save(){ nmap -oA "scan-$(date +%Y%m%d-%H%M%S)" "$@" }; _nmap_save'

# Quick scans with common options
alias nmap-web='nmap -p 80,443,8000,8080,8443 -sV -T4'
alias nmap-ssh='nmap -p 22 -sV --script=ssh-auth-methods -T4'
alias nmap-ftp='nmap -p 21 -sV --script=ftp-anon -T4'
alias nmap-smb='nmap -p 139,445 -sV --script=smb-enum-shares -T4'
```

## Usage Examples with Aliases

```bash
# Quick scan of common ports
nmap-quick 192.168.1.1

# Discover live hosts on local network
nmap-discover 192.168.1.0/24

# Comprehensive scan with all features
nmap-comprehensive example.com

# Check for vulnerabilities
nmap-vuln target.com

# Scan web services
nmap-web 192.168.1.0/24

# Save scan results with timestamp
nmap-save -sS -sV target.com
```

Remember to reload your shell configuration after adding aliases:
```bash
source ~/.zshrc
```