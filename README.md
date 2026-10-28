# 🔒 VPS — Best Effort Max Security

> Harden a fresh Ubuntu 24.04 / Debian 12 VPS to the highest practical baseline.  
> Follow every section in order. Do **not** skip or reorder steps 1–5.

---

## ⚡ Golden Rule

```
Updates → Sudo user → SSH keys → Harden SSH → Firewall
→ Fail2Ban → Kernel → Auto-updates → Monitoring → Advanced
```

Never close your root session until you have verified key-based login from a **second terminal**.

---

## 1 — System Updates

```bash
apt update && apt upgrade -y && apt dist-upgrade -y
apt autoremove -y && apt autoclean
```

Set timezone and sync time:

```bash
timedatectl set-timezone UTC
timedatectl set-ntp true

# Higher accuracy (optional)
apt install chrony -y
systemctl enable --now chrony
```

---

## 2 — Sudo User + Lock Root

```bash
adduser chidev
usermod -aG sudo chidev

# Test sudo BEFORE locking root
su - chidev
sudo whoami          # must return: root
exit

# Lock root
passwd -l root
passwd -S root       # must show: root L ...
```

Strong password policy:

```bash
apt install libpam-pwquality -y
```

`/etc/security/pwquality.conf`:

```ini
minlen    = 16
minclass  = 3
maxrepeat = 3
dcredit   = -1
ucredit   = -1
lcredit   = -1
ocredit   = -1
```

Sudo hardening (`visudo`):

```
Defaults    timestamp_timeout=5
Defaults    logfile="/var/log/sudo.log"
Defaults    log_input, log_output
Defaults    requiretty
```

---

## 3 — SSH: Keys Only, Root Blocked

### On your LOCAL machine

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/vps_ed25519
ssh-copy-id -i ~/.ssh/vps_ed25519.pub chidev@YOUR_VPS_IP
```

**Open a new terminal and test the key login before touching anything else.**

```bash
ssh -i ~/.ssh/vps_ed25519 chidev@YOUR_VPS_IP   # must work
```

### Harden `/etc/ssh/sshd_config`

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

```
Port 2222                              # move off port 22
Protocol 2

PermitRootLogin                 no
PasswordAuthentication          no
PubkeyAuthentication            yes
AuthorizedKeysFile              .ssh/authorized_keys
PermitEmptyPasswords            no
ChallengeResponseAuthentication no
GSSAPIAuthentication            no
UsePAM                          yes

MaxAuthTries     3
MaxSessions      5
LoginGraceTime   30
AllowUsers       chidev

X11Forwarding        no
AllowAgentForwarding no
AllowTcpForwarding   no
PermitTunnel         no

ClientAliveInterval  300
ClientAliveCountMax  2

SyslogFacility AUTH
LogLevel       VERBOSE

# Crypto — modern only
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
KexAlgorithms  curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers        chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs           hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
```

```bash
sshd -t                  # validate — zero errors required
systemctl restart sshd
ss -tlnp | grep sshd     # confirm port 2222
```

**Test new port from second terminal before closing root session.**

Local `~/.ssh/config`:

```
Host vps
    HostName        YOUR_VPS_IP
    User            chidev
    Port            2222
    IdentityFile    ~/.ssh/vps_ed25519
    ServerAliveInterval 60
```

---

## 4 — Firewall (UFW)

```bash
apt install ufw -y

ufw default deny incoming
ufw default allow outgoing

ufw allow 2222/tcp  comment 'SSH'
ufw limit 2222/tcp              # rate-limit brute force
ufw allow 80/tcp    comment 'HTTP'
ufw allow 443/tcp   comment 'HTTPS'

# Lock SSH to your static IP only (recommended)
# ufw delete allow 2222/tcp
# ufw allow from YOUR.HOME.IP to any port 2222 proto tcp

ufw enable
ufw status verbose
```

Allow Docker → host (adjust subnet to match `docker network inspect`):

```bash
ufw allow from 172.16.1.0/24 to any port 3000 comment 'Docker → Next.js'
ufw allow from 172.16.1.0/24 to any port 3026 comment 'Docker → App'
ufw reload
```

Enable logging:

```bash
ufw logging on
ufw logging medium
```

---

## 5 — Fail2Ban

```bash
apt install fail2ban -y
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

`/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime   = 3600
findtime  = 600
maxretry  = 3
ignoreip  = 127.0.0.1/8 ::1 YOUR.HOME.IP
banaction = ufw

[sshd]
enabled  = true
port     = 2222
maxretry = 3
bantime  = 86400
```

```bash
systemctl enable --now fail2ban
fail2ban-client status sshd
```

---

## 6 — Kernel Hardening

`/etc/sysctl.d/99-security.conf`:

```ini
# Network
net.ipv4.ip_forward                    = 0
net.ipv4.tcp_syncookies                = 1
net.ipv4.icmp_echo_ignore_broadcasts   = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter            = 1
net.ipv4.conf.default.rp_filter        = 1
net.ipv4.conf.all.accept_redirects     = 0
net.ipv4.conf.all.send_redirects       = 0
net.ipv4.conf.all.accept_source_route  = 0
net.ipv4.conf.all.log_martians         = 1
net.ipv4.tcp_rfc1337                   = 1
net.ipv6.conf.all.accept_redirects     = 0
net.ipv6.conf.all.accept_source_route  = 0

# Memory / exploit mitigations
kernel.randomize_va_space = 2
kernel.kptr_restrict      = 2
kernel.dmesg_restrict     = 1
kernel.yama.ptrace_scope  = 1
kernel.sysrq              = 0
fs.suid_dumpable          = 0

# Filesystem
fs.protected_hardlinks = 1
fs.protected_symlinks  = 1
fs.protected_fifos     = 2
fs.protected_regular   = 2
```

```bash
sysctl --system
```

---

## 7 — Automatic Security Updates

```bash
apt install unattended-upgrades apt-listchanges -y
```

`/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

`/etc/apt/apt.conf.d/20auto-upgrades`:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

```bash
unattended-upgrade --dry-run
systemctl enable --now unattended-upgrades
```

---

## 8 — System Hardening

```bash
# Secure /dev/shm
echo "tmpfs /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0" >> /etc/fstab
mount -o remount /dev/shm

# Secure umask for all users
echo "umask 027" >> /etc/profile
echo "umask 027" >> /etc/bash.bashrc

# File permissions
chmod 640 /etc/shadow
chmod 600 /etc/ssh/sshd_config
chmod 700 /root

# Disable unnecessary services
systemctl disable --now avahi-daemon cups bluetooth ModemManager 2>/dev/null || true

# Find world-writable files (investigate everything)
find / -xdev -type f -perm -0002 -ls 2>/dev/null

# Find SUID/SGID binaries (audit the list)
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -ls 2>/dev/null
```

---

## 9 — Monitoring & Logging

```bash
# Logwatch — daily digest
apt install logwatch -y
echo "0 6 * * * root /usr/sbin/logwatch --output mail --mailto you@example.com --detail med" \
  > /etc/cron.d/logwatch

# AIDE — filesystem integrity
apt install aide aide-common -y
aideinit
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
echo "0 3 * * * root /usr/bin/aide --check" > /etc/cron.d/aide

# Lynis — security audit
apt install lynis -y
lynis audit system     # target score: 75+ (85+ = excellent)
echo "0 4 * * 0 root /usr/bin/lynis audit system --quiet" > /etc/cron.d/lynis
```

Log retention (`/etc/logrotate.conf`):

```
rotate 52
weekly
compress
delaycompress
```

Make logs append-only (attackers can't erase tracks):

```bash
chattr +a /var/log/auth.log
chattr +a /var/log/syslog
```

---

## 10 — Advanced (Maximum Security)

### SSH Two-Factor Authentication

```bash
apt install libpam-google-authenticator -y
su - chidev
google-authenticator   # scan QR with Authenticator app
```

`/etc/pam.d/sshd` — add at top:

```
auth required pam_google_authenticator.so
```

`sshd_config`:

```
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

```bash
systemctl restart sshd
```

### AppArmor

```bash
apt install apparmor-profiles apparmor-utils -y
aa-enforce /etc/apparmor.d/*
aa-status
```

### Rootkit Detection

```bash
apt install rkhunter -y
rkhunter --update
rkhunter --check --skip-keypress
echo "0 5 * * 0 root /usr/bin/rkhunter --check --skip-keypress --report-warnings-only" \
  > /etc/cron.d/rkhunter
```

### Block Unnecessary Kernel Modules

`/etc/modprobe.d/blacklist-security.conf`:

```
blacklist usb-storage
blacklist dccp
blacklist sctp
blacklist rds
blacklist tipc
blacklist cramfs
blacklist hfs
blacklist hfsplus
blacklist udf
```

```bash
update-initramfs -u
```

### Process Accounting

```bash
apt install acct -y
systemctl enable --now acct
lastcomm      # every command run by every user
```

---

## ✅ Final Verification

```bash
# SSH on correct port, not 22
ss -tlnp | grep sshd

# Root login rejected
ssh root@localhost -p 2222
# → Permission denied (publickey)

# Firewall
ufw status verbose

# Fail2Ban
fail2ban-client status sshd

# Kernel
sysctl net.ipv4.tcp_syncookies     # → 1
sysctl kernel.randomize_va_space   # → 2
sysctl kernel.dmesg_restrict       # → 1
sysctl net.ipv4.ip_forward         # → 0

# Root locked
passwd -S root   # → root L ...

# File permissions
stat /etc/shadow          # → 640
stat /etc/ssh/sshd_config # → 600

# Integrity baseline clean
aide --check

# Audit score
lynis audit system --quick   # → 75+

# External scan (from another machine)
nmap -sV -p 1-1024 YOUR_VPS_IP   # → only 80, 443, 2222 visible
```

---

## ❌ Never Do These

| Mistake | Consequence |
|---|---|
| Modify `sshd_config` without testing in a new terminal | Locked out permanently |
| Enable UFW before allowing your SSH port | Instant lockout |
| `PermitRootLogin no` before testing sudo user | No recovery path |
| Skip AIDE rebuild after `apt upgrade` | False alarms you'll start ignoring |
| Use RSA < 3072-bit or DSA keys | Weak cryptography |
| Leave private key without a passphrase | Stolen laptop = compromised server |
| Expose app ports (3000, 3026) publicly | Bypasses nginx entirely |
| Treat hardening as a one-time task | Threat landscape changes constantly |

---

> Security is a practice, not a checkbox.  
> Run `lynis audit system` quarterly · Review Fail2Ban bans weekly · Rebuild AIDE after every planned change.
