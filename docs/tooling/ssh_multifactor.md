 Here's how to enable YubiKey MFA for SSH authentication:

  1. Install Required Software

  # Ubuntu/Debian
  sudo apt update
  sudo apt install libpam-u2f pamu2fcfg

  # CentOS/RHEL/Fedora
  sudo dnf install pam-u2f pamu2fcfg

  # macOS
  brew install pam-u2f

  2. Register YubiKey

  # Create U2F directory
  mkdir -p ~/.config/Yubico

  # Register your YubiKey (touch it when it blinks)
  pamu2fcfg > ~/.config/Yubico/u2f_keys

  # For multiple YubiKeys, append additional registrations
  pamu2fcfg -n >> ~/.config/Yubico/u2f_keys

  3. Configure PAM

  Edit /etc/pam.d/sshd:
  # Add this line for required MFA (must have YubiKey)
  auth required pam_u2f.so

  # OR add this line for sufficient MFA (YubiKey OR password)  
  auth sufficient pam_u2f.so

  # Place before or after other auth lines depending on desired flow

  4. Configure SSH Server

  Edit /etc/ssh/sshd_config:
  # Enable challenge-response authentication
  KbdInteractiveAuthentication yes
  UsePAM yes

  # Require BOTH key AND YubiKey
  AuthenticationMethods publickey,keyboard-interactive

  # OR require key OR password+YubiKey
  #AuthenticationMethods publickey keyboard-interactive:pam

  5. Advanced Configuration Options

  System-wide YubiKey registration:
  # Register for all users (as root)
  sudo pamu2fcfg -u username > /etc/u2f_mappings

  # Configure PAM to use system file
  # In /etc/pam.d/sshd:
  auth required pam_u2f.so authfile=/etc/u2f_mappings

  Additional PAM options:
  # Debug mode
  auth required pam_u2f.so debug

  # Custom prompt
  auth required pam_u2f.so prompt="Please touch YubiKey"

  # Timeout (default 15 seconds)
  auth required pam_u2f.so timeout=30

  6. Test Configuration

  # Restart SSH daemon
  sudo systemctl restart sshd

  # Test authentication (YubiKey should blink)
  ssh user@hostname

  # Check logs for debugging
  sudo tail -f /var/log/auth.log

  7. Hardware Key Support (FIDO2/WebAuthn)

  For newer YubiKeys with FIDO2:
  # Generate SSH key on YubiKey
  ssh-keygen -t ed25519-sk -f ~/.ssh/id_ed25519_sk

  # Copy public key to server
  ssh-copy-id -i ~/.ssh/id_ed25519_sk.pub user@server

  # SSH with hardware key (will require touch)
  ssh -i ~/.ssh/id_ed25519_sk user@server

  This setup requires both SSH key authentication AND YubiKey touch for secure
  MFA.