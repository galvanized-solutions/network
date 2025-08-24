# YubiKey Configuration Guide for macOS

## Introduction

YubiKey is a hardware security key that provides strong two-factor authentication (2FA) and multi-factor authentication (MFA) across various applications and services. This guide covers comprehensive YubiKey setup and configuration for macOS across all supported applications.

## Prerequisites

### YubiKey Models Supported on macOS
- **YubiKey 5 Series**: Latest with USB-A, USB-C, NFC, and Lightning variants
- **YubiKey 4 Series**: Legacy but still supported
- **Security Key Series**: FIDO2/WebAuthn only, budget-friendly option

### Required Software Installation

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install YubiKey Manager
brew install --cask yubico-yubikey-manager

# Install YubiKey Personalization Tool
brew install --cask yubico-yubikey-personalization-gui

# Install command-line tools
brew install ykman yubico-piv-tool

# Install PAM module for system authentication
brew install pam-u2f

# Install additional utilities
brew install oath-toolkit libfido2
```

## Core YubiKey Applications and Configuration

### 1. YubiKey Manager (Primary Configuration Tool)

**What it is**: Official GUI application for managing YubiKey settings, OTP, PIV certificates, and FIDO2 credentials.

**When to use**: 
- Initial YubiKey setup and configuration
- Managing OTP applications
- Configuring PIV certificates
- Viewing and managing FIDO2 credentials
- Firmware updates

**Configuration Steps**:
```bash
# Launch YubiKey Manager
open /Applications/YubiKey\ Manager.app

# Or via command line
ykman info  # Display YubiKey information
ykman config usb --enable-all  # Enable all USB interfaces
ykman config nfc --enable-all  # Enable all NFC interfaces (if supported)
```

**Key Features**:
- **Applications Tab**: Configure OTP, FIDO2, PIV, OATH
- **Interfaces Tab**: Enable/disable USB and NFC interfaces
- **Configuration Tab**: Set device-wide settings

### 2. OATH (One-Time Password) Configuration

**What it is**: Stores TOTP/HOTP secrets for 2FA codes, replacing apps like Google Authenticator.

**When to use**: 
- Store 2FA codes for websites and services
- Backup solution for authenticator apps
- Corporate environments requiring hardware-based TOTP

**Setup Process**:
```bash
# Add OATH credential manually
ykman oath accounts add "Service Name" "SECRET_KEY_BASE32"

# Add credential with period (default 30 seconds)
ykman oath accounts add -p 60 "Service:user@example.com" "SECRET"

# List all OATH accounts
ykman oath accounts list

# Generate code for specific account
ykman oath accounts code "Service Name"

# Generate codes for all accounts
ykman oath accounts code
```

**GUI Configuration**:
1. Open YubiKey Manager
2. Click "Applications" → "OATH"
3. Click "Add account"
4. Enter service name and secret key (or scan QR code)
5. Test code generation

### 3. PIV (Personal Identity Verification) for Digital Certificates

**What it is**: Smart card functionality for storing X.509 certificates and private keys for authentication, encryption, and digital signing.

**When to use**:
- Corporate PKI environments
- Client certificate authentication
- Code signing
- Email encryption/signing (S/MIME)
- SSH key authentication

**Initial PIV Setup**:
```bash
# Reset PIV application (WARNING: destroys existing certificates)
ykman piv reset

# Set new PIN and PUK
ykman piv access change-pin -P 123456 -n NEW_PIN
ykman piv access change-puk -p 12345678 -n NEW_PUK

# Set management key
ykman piv access change-management-key -m NEW_MGMT_KEY

# Generate key and self-signed certificate
ykman piv keys generate -a RSA2048 9a /tmp/public.pem
ykman piv certificates generate -s "CN=YubiKey User" 9a /tmp/public.pem

# Or import existing certificate
ykman piv certificates import 9a certificate.pem
```

**PIV Slot Usage**:
- **9a**: PIV Authentication (SSH, client auth)
- **9c**: Digital Signature (code signing, email)
- **9d**: Key Management (encryption)
- **9e**: Card Authentication (physical access)

### 4. FIDO2/WebAuthn Configuration

**What it is**: Modern passwordless authentication standard supported by major web services.

**When to use**:
- Passwordless login to websites
- Modern 2FA without SMS/codes
- Corporate Single Sign-On (SSO)
- Developer authentication (GitHub, GitLab)

**Setup Process**:
1. Visit supported website (Google, Microsoft, GitHub, etc.)
2. Navigate to Security Settings
3. Add Security Key
4. Follow prompts to register YubiKey
5. Test authentication

**Supported Services**:
- Google Accounts
- Microsoft Accounts
- GitHub/GitLab
- Facebook
- Twitter
- Dropbox
- 1Password
- Bitwarden

### 5. Static Password Configuration

**What it is**: Programs YubiKey to output a static password when touched.

**When to use**:
- Legacy systems requiring long, complex passwords
- Shared workstation access
- Quick password entry for frequently used accounts

**Configuration**:
```bash
# Set static password (Slot 2)
ykman otp static -l 38 2  # Generate 38-character password
# Or set custom password
echo "MySecretPassword123!" | ykman otp static -P 2
```

## Application-Specific Integrations

### 1. SSH Authentication

**Purpose**: Use YubiKey for SSH key storage and authentication.

**Configuration**:
```bash
# Generate SSH key on YubiKey (FIDO2)
ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk

# Or use PIV-stored key
ssh-keygen -D /usr/local/lib/libykcs11.dylib

# Add public key to servers
ssh-copy-id -i ~/.ssh/id_ed25519_sk.pub user@server

# SSH with YubiKey (requires touch)
ssh -i ~/.ssh/id_ed25519_sk user@server
```

### 2. macOS Login (System Authentication)

**Purpose**: Use YubiKey for macOS user login and sudo authentication.

**Configuration**:
```bash
# Register YubiKey for current user
mkdir -p ~/.config/Yubico
pamu2fcfg > ~/.config/Yubico/u2f_keys

# Configure PAM (requires admin privileges)
sudo vim /etc/pam.d/authorization
# Add: auth sufficient pam_u2f.so

# For sudo authentication
sudo vim /etc/pam.d/sudo
# Add: auth sufficient pam_u2f.so
```

### 3. Password Manager Integration

#### 1Password
**Setup**:
1. Open 1Password → Preferences → Security
2. Enable "Unlock using Touch ID" 
3. Add YubiKey as security key in 1Password account online
4. Test authentication

#### Bitwarden
**Setup**:
1. Log into Bitwarden web vault
2. Settings → Two-step Login
3. Enable FIDO2 WebAuthn
4. Register YubiKey
5. Test login on all devices

### 4. Web Browser Configuration

#### Safari
**Setup**:
1. Safari → Preferences → Passwords
2. Enable "AutoFill user names and passwords"
3. Websites automatically prompt for YubiKey when configured

#### Chrome
**Setup**:
1. Chrome → Settings → Privacy and Security → Security Keys
2. Manage security keys for WebAuthn
3. Register YubiKey with supported sites

#### Firefox
**Setup**:
1. Firefox → Preferences → Privacy & Security
2. Enable "Use a security key" for 2FA
3. about:config → security.webauth.u2f → true

### 5. Email Client Configuration (S/MIME)

#### Apple Mail
**Setup**:
```bash
# Export certificate from YubiKey
ykman piv certificates export 9c cert.pem

# Import to Keychain
security import cert.pem -k ~/Library/Keychains/login.keychain

# Configure in Mail → Preferences → Accounts → Advanced
# Set Signing Certificate and Encryption Certificate
```

### 6. Code Signing and Development

#### Git Commit Signing
**Setup**:
```bash
# Configure Git to use YubiKey certificate
git config --global user.signingkey "Certificate Subject"
git config --global commit.gpgsign true
git config --global gpg.program gpgsm

# Sign commits
git commit -S -m "Signed commit"
```

#### macOS Code Signing
**Setup**:
```bash
# Export code signing certificate
ykman piv certificates export 9c code-sign.pem

# Import to Keychain for Xcode
security import code-sign.pem -k ~/Library/Keychains/login.keychain

# Configure Xcode to use certificate
# Xcode → Preferences → Accounts → Manage Certificates
```

### 7. VPN and Network Authentication

#### OpenVPN with Client Certificates
**Setup**:
```bash
# Export client certificate and configure OpenVPN
echo "pkcs11-providers /usr/local/lib/libykcs11.dylib" >> client.ovpn
echo "pkcs11-id 'YubiKey PIV #X/SLOT/CERTIFICATE'" >> client.ovpn
```

### 8. GPG (GNU Privacy Guard) Integration

**What it is**: Use YubiKey's OpenPGP applet for GPG operations including encryption, decryption, and digital signing.

**When to use**:
- Email encryption and signing
- File encryption and decryption
- Git commit and tag signing
- Software package signing
- Secure communication and document authentication

#### Initial GPG Setup on macOS

```bash
# Install GPG and related tools
brew install gnupg yubikey-personalization hopenpgp-tools ykman pinentry-mac

# Configure GPG to use pinentry-mac
echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf

# Restart GPG agent
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent
```

#### Method 1: Generate GPG Keys Directly on YubiKey (Recommended)

```bash
# Enable OpenPGP application on YubiKey
ykman openpgp reset  # WARNING: This deletes existing OpenPGP data

# Start GPG key generation process
gpg --card-edit
```

**GPG Interactive Commands**:
```
gpg/card> admin
gpg/card> generate
Make off-card backup of encryption key? (Y/n) n
What keysize do you want? (3072) 4096
Key is valid for? (0) 2y
Real name: Your Name
Email address: your.email@example.com
Comment: YubiKey GPG
```

#### Method 2: Import Existing GPG Keys to YubiKey

```bash
# Generate master key offline (air-gapped system recommended)
gpg --expert --full-gen-key
# Choose: (1) RSA and RSA, 4096 bits, 2 years validity

# Generate subkeys for daily use
gpg --expert --edit-key YOUR_KEY_ID
```

**GPG Interactive Subkey Creation**:
```
gpg> addkey
What kind of key do you want? (6) RSA (encrypt only)
What keysize do you want? (3072) 4096
Key is valid for? (0) 1y

gpg> addkey  
What kind of key do you want? (4) RSA (sign only)
What keysize do you want? (3072) 4096
Key is valid for? (0) 1y

gpg> addkey
What kind of key do you want? (8) RSA (set your own capabilities)
Current allowed actions: Sign Encrypt 
Toggle: (A) Authenticate capability
What keysize do you want? (3072) 4096
Key is valid for? (0) 1y

gpg> save
```

**Transfer Subkeys to YubiKey**:
```bash
# Select and move subkeys to YubiKey
gpg --edit-key YOUR_KEY_ID

gpg> key 1  # Select first subkey (encryption)
gpg> keytocard
Your selection? 2  # Encryption slot

gpg> key 1  # Deselect
gpg> key 2  # Select second subkey (signing)
gpg> keytocard
Your selection? 1  # Signature slot

gpg> key 2  # Deselect  
gpg> key 3  # Select third subkey (authentication)
gpg> keytocard
Your selection? 3  # Authentication slot

gpg> save
```

#### GPG YubiKey Configuration

```bash
# Set YubiKey PINs (default PIN: 123456, Admin PIN: 12345678)
gpg --card-edit
```

**Change PINs**:
```
gpg/card> admin
gpg/card> passwd
1 - change PIN (user PIN for daily operations)
2 - unblock PIN  
3 - change Admin PIN (for administrative operations)
Q - quit
```

**Configure Touch Policies**:
```bash
# Require touch for signing operations
ykman openpgp keys set-touch sig on

# Require touch for encryption operations  
ykman openpgp keys set-touch enc on

# Require touch for authentication operations
ykman openpgp keys set-touch auth on

# Cached touch (touch once, valid for 15 seconds)
ykman openpgp keys set-touch sig cached
```

#### GPG Daily Usage

**Encryption and Decryption**:
```bash
# Encrypt file for recipient
gpg --encrypt --armor --recipient recipient@example.com document.txt

# Decrypt file (YubiKey will prompt for PIN and touch)
gpg --decrypt document.txt.asc

# Encrypt for yourself
gpg --encrypt --armor --recipient your.email@example.com secret.txt
```

**Digital Signing**:
```bash
# Sign file (detached signature)
gpg --detach-sign --armor document.txt

# Sign and encrypt
gpg --sign --encrypt --armor --recipient recipient@example.com document.txt

# Verify signature
gpg --verify document.txt.asc document.txt

# Clear-sign document (sign inline)
gpg --clear-sign document.txt
```

#### Git Integration with YubiKey GPG

```bash
# Configure Git to use YubiKey GPG key
gpg --list-secret-keys --keyid-format LONG
# Copy the key ID from the output

git config --global user.signingkey YOUR_GPG_KEY_ID
git config --global commit.gpgsign true
git config --global tag.gpgSign true
git config --global gpg.program gpg

# Sign commits and tags
git commit -S -m "Signed commit message"
git tag -s v1.0.0 -m "Signed tag"

# Verify signatures
git log --show-signature
git tag -v v1.0.0
```

#### Email Client Integration

**Apple Mail with GPG Suite**:
```bash
# Install GPG Suite for macOS
brew install --cask gpg-suite

# Configure in Mail preferences:
# Mail → Preferences → GPG Suite → Select your YubiKey key
```

**Thunderbird with Enigmail/OpenPGP**:
1. Install Thunderbird
2. Enable built-in OpenPGP support
3. Account Settings → End-to-End Encryption
4. Import your public key or configure YubiKey

#### Advanced GPG Configuration

**Backup Strategy**:
```bash
# Export public key for sharing
gpg --export --armor your.email@example.com > public.asc

# Export secret key stubs (safe to backup)
gpg --export-secret-keys your.email@example.com > secret-stubs.gpg

# Export master key (keep offline!)
gpg --export-secret-key --armor YOUR_MASTER_KEY_ID > master-key.asc

# Backup revocation certificate
cp ~/.gnupg/openpgp-revocs.d/YOUR_KEY_ID.rev ~/backup/
```

**Multiple YubiKey Setup**:
```bash
# Clone GPG configuration to second YubiKey
# First, backup subkeys from original YubiKey
gpg --card-status  # Ensure original YubiKey is connected

# Then prepare second YubiKey
ykman openpgp reset  # On second YubiKey
# Repeat key generation or import process
```

**GPG Agent Configuration** (`~/.gnupg/gpg-agent.conf`):
```bash
# Cache settings
default-cache-ttl 28800
max-cache-ttl 86400

# PIN entry
pinentry-program /opt/homebrew/bin/pinentry-mac

# SSH agent support
enable-ssh-support

# Logging
log-file ~/.gnupg/gpg-agent.log
debug-level basic
```

#### Troubleshooting GPG with YubiKey

**Common Issues**:

**YubiKey not detected**:
```bash
# Check card status
gpg --card-status

# Kill and restart GPG agent
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent

# Check for conflicts with other smart card services
sudo launchctl list | grep -i smart
```

**PIN entry problems**:
```bash
# Test PIN entry
echo "test" | gpg --sign --armor

# Reset PIN entry program
echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf
gpgconf --kill gpg-agent
```

**Touch not working**:
```bash
# Check touch settings
ykman openpgp info

# Reset touch policy
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch enc on  
ykman openpgp keys set-touch auth on
```

### 9. Database and Enterprise Applications

#### PostgreSQL Client Certificate Authentication
**Setup**:
```bash
# Extract certificate and key for PostgreSQL
ykman piv certificates export 9a ~/.postgresql/postgresql.crt
# Configure connection string to use client certificate
```

## Advanced Configuration

### 1. Multiple YubiKey Management

```bash
# List all connected YubiKeys
ykman list

# Work with specific YubiKey by serial number
ykman --device 12345678 info
ykman --device 12345678 oath accounts list

# Backup and restore OATH accounts between YubiKeys
ykman oath accounts code > backup.txt
# Manually re-add to second YubiKey
```

### 2. Custom Touch Policies

```bash
# Require touch for PIV operations
ykman piv keys generate --touch-policy always 9a public.pem

# Cached touch (touch once, valid for 15 seconds)
ykman piv keys generate --touch-policy cached 9a public.pem
```

### 3. PIN Complexity Policies

```bash
# Set PIN complexity requirements (6-8 digits)
ykman piv access change-pin --new-pin 12345678

# Enable PIN complexity policy (enterprise feature)
ykman piv config set-pin-policy complex
```

## Security Best Practices

### 1. Backup Strategy
- **Purchase 2+ YubiKeys**: Configure identically for redundancy
- **Store backup codes**: Save service-specific backup codes
- **Document configuration**: Keep record of all configured services
- **Secure storage**: Store backup YubiKey in separate location

### 2. PIN and Key Management
- **Use strong PINs**: 8-digit PINs for PIV
- **Don't share management keys**: Keep PIV management keys secret
- **Regular rotation**: Rotate certificates annually
- **Access logging**: Monitor YubiKey usage logs

### 3. Physical Security
- **Attach to keychain**: Prevent loss
- **Never leave unattended**: Physical access = compromise
- **Register serial numbers**: For asset tracking
- **Consider bio-metric backup**: Touch ID as fallback

## Troubleshooting

### Common Issues

#### YubiKey Not Detected
```bash
# Check USB interfaces
ykman config usb --list

# Enable all interfaces
ykman config usb --enable-all

# Check system permissions
ls -la /dev/cu.usbmodem*
```

#### PIV PIN Blocked
```bash
# Check PIN retry counter
ykman piv info

# Unlock with PUK
ykman piv access unblock-pin

# Reset if PUK also blocked (destroys data)
ykman piv reset
```

#### OATH Time Sync Issues
```bash
# Check system time
date

# Manually sync if needed
sudo sntp -sS time.apple.com
```

### Logging and Debugging

```bash
# Enable verbose logging
export YKMAN_LOG_LEVEL=DEBUG
ykman oath accounts list

# Check system logs
log show --predicate 'subsystem == "com.yubico"' --info --last 1h

# PIV operations logging
export OPENSC_DEBUG=3
```

## Integration Scripts

### Automated Setup Script
```bash
#!/bin/bash
# YubiKey macOS Setup Script

echo "Setting up YubiKey for macOS..."

# Install required software
if ! command -v brew &> /dev/null; then
    echo "Installing Homebrew..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

brew install --cask yubico-yubikey-manager
brew install ykman pam-u2f

# Basic configuration
echo "Enabling all YubiKey interfaces..."
ykman config usb --enable-all 2>/dev/null
ykman config nfc --enable-all 2>/dev/null

echo "YubiKey setup complete!"
echo "Next steps:"
echo "1. Configure OATH accounts"
echo "2. Set up PIV certificates"
echo "3. Register FIDO2 credentials"
```

### Backup Script
```bash
#!/bin/bash
# Backup YubiKey Configuration

BACKUP_DIR="$HOME/yubikey-backup-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

echo "Backing up YubiKey configuration..."

# Export PIV certificates
for slot in 9a 9c 9d 9e; do
    ykman piv certificates export "$slot" "$BACKUP_DIR/piv-$slot.pem" 2>/dev/null
done

# List OATH accounts
ykman oath accounts list > "$BACKUP_DIR/oath-accounts.txt"

# Export device info
ykman info > "$BACKUP_DIR/device-info.txt"

echo "Backup saved to: $BACKUP_DIR"
```

## Conclusion

YubiKey provides robust hardware-based security for macOS users across numerous applications and services. This guide covers the essential configurations for maximum security benefit. Regular practice with backup YubiKeys and keeping configuration documentation updated ensures seamless operation and recovery capabilities.

For enterprise deployments, consider additional features like centralized management, bulk configuration tools, and integration with identity providers like Active Directory or LDAP.