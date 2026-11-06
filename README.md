
# Linux Security Home Lab | Fedora Asahi & Splunk

![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)
![Linux](https://img.shields.io/badge/OS-Fedora%20Asahi%2045-blue)
![Splunk](https://img.shields.io/badge/SIEM-Splunk-green)
![Architecture](https://img.shields.io/badge/Architecture-ARM64-orange)

## 📌 Project Overview

This project documents the deployment, configuration, and security hardening of a dedicated Linux cybersecurity home lab using a repurposed Apple Mac mini M1.

The goal is to develop practical experience with Linux system administration, security engineering, network configuration, log management, and threat detection.

The environment consists of a Fedora Asahi Remix Server running on Apple Silicon and a separate Windows 11 workstation hosting Splunk Enterprise.

The lab will be used to collect and analyze security logs, develop detection rules, automate security tasks, and simulate security events.

---

## 🖥️ Lab Environment

| Component | Configuration |
|---|---|
| Server | Apple Mac mini M1 (2020) |
| CPU Architecture | ARM64 / aarch64 |
| RAM | 8GB |
| Internal Storage | 256GB SSD |
| External Storage | 2TB Western Digital Drive |
| Operating System | Fedora Asahi Remix 45 Server |
| Kernel | 7.2.12-402.asahi.fc45.aarch64+16k |
| SIEM | Splunk Enterprise (Windows 11) |
| Forwarder | Splunk Universal Forwarder 10.4.3 |
| Remote Administration | OpenSSH |
| Firewall | firewalld |
| Networking | Private Wi-Fi Network |

### Network Architecture

```text
                  Home Network
                       |
                 Wi-Fi Router
                       |
             +---------+---------+
             |                   |
      Windows 11 PC         Mac mini M1
             |                   |
     Splunk Enterprise     Fedora Asahi Server
             |                   |
      TCP Port 9997        Splunk Universal
      Receiving            Forwarder
             ^                   |
             |                   |
             +-------------------+
                Log Forwarding
                   TCP 9997

        Windows PC ---- SSH ----> Fedora
                       TCP 22
```

The Fedora server establishes outbound connections to Splunk Enterprise for centralized log collection.

SSH provides remote administrative access from the Windows workstation.

---

## 🔧 Phase 1 — Linux Server Deployment

Installed Fedora Asahi Remix Server on an Apple M1 Mac mini.

### Configuration

- Installed Fedora Asahi Remix 45 Server.
- Allocated approximately 120GB of internal storage to Linux.
- Preserved required macOS and Apple Silicon recovery partitions.
- Configured Wi-Fi connectivity.
- Created a dedicated Linux user account.
- Configured administrative permissions using the `wheel` group.

### Administrator Permissions

Initially, the standard Linux user could not execute commands using `sudo`.

The issue was resolved by adding the user to the administrative `wheel` group:

```bash
su -
usermod -aG wheel <username>
```

Verified administrative access:

```bash
sudo whoami
```

**Expected Result:**

```text
root
```

**Lessons Learned:**

- `sudo` allows authorized users to execute commands with elevated privileges.
- `wheel` is the administrative group commonly used by Fedora.
- `usermod -aG` adds a user to a supplementary group without removing existing memberships.
- Group membership changes require a new login session to take effect.

---

## 🔐 Phase 2 — SSH Configuration and Hardening

Configured OpenSSH to enable secure remote administration of the Fedora server.

### Enable SSH

```bash
sudo systemctl enable --now sshd
```

Verified service status:

```bash
sudo systemctl status sshd
```

Connected from Windows PowerShell:

```powershell
ssh <username>@<server-ip>
```

### SSH Key Authentication

Generated an Ed25519 SSH key pair on the Windows workstation:

```powershell
ssh-keygen -t ed25519 -C "windows-to-asahi-server"
```

Installed the public key on the Fedora server.

The private key remains on Windows and is protected by a passphrase.

### SSH Hardening

Created an SSH configuration file:

```text
/etc/ssh/sshd_config.d/50-key-auth.conf
```

Applied the following settings:

```ini
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

Validated the configuration:

```bash
sudo sshd -t
```

Inspected effective settings:

```bash
sudo sshd -T
```

Reloaded the SSH service:

```bash
sudo systemctl reload sshd
```

### Security Validation

Successfully authenticated using an SSH key.

Verified that password-only SSH authentication was rejected.

**Security Improvements:**

- ✅ Enabled public-key authentication.
- ✅ Disabled password-based authentication.
- ✅ Disabled keyboard-interactive authentication.
- ✅ Disabled direct root SSH login.
- ✅ Protected the private key with a passphrase.
- ✅ Verified remote access after applying changes.

**Lessons Learned:**

SSH provides encrypted remote administration.

Public-key authentication uses a cryptographic key pair to authenticate a client without transmitting the account password.

Disabling password authentication reduces exposure to password-guessing attacks.

---

## 💾 Phase 3 — External Storage Configuration

Configured an external 2TB Western Digital drive as dedicated cybersecurity lab storage.

### Identify Storage Devices

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,MODEL
```

The external drive was identified as `/dev/sda`, with its primary data partition at `/dev/sda2`.

### Filesystem Configuration

After confirming existing data was no longer needed, formatted the external data partition using ext4:

```bash
sudo mkfs.ext4 -L lab-storage /dev/sda2
```

> **Warning:** Formatting permanently destroys data on the selected partition. Device names must be verified before executing this command.

Created a mount directory:

```bash
sudo mkdir -p /srv/lab-storage
```

Mounted the filesystem:

```bash
sudo mount /dev/sda2 /srv/lab-storage
```

### Persistent Mounting

Configured `/etc/fstab` to automatically mount the external storage during system startup.

Used the filesystem UUID rather than the device name to identify the partition.

Example:

```text
UUID=<filesystem-uuid> /srv/lab-storage ext4 defaults,nofail,x-systemd.device-timeout=10s 0 2
```

Validated the configuration:

```bash
sudo mount -a
findmnt /srv/lab-storage
```

Confirmed that the external storage remained mounted after reboot.

### Storage Directory Structure

```text
/srv/lab-storage/
├── backups/
├── containers/
├── datasets/
├── logs/
├── pcaps/
└── projects/
```

| Directory | Purpose |
|---|---|
| backups | Configuration and project backups |
| containers | Container application storage |
| datasets | Synthetic security datasets |
| logs | Security and system logs |
| pcaps | Network packet captures |
| projects | Python scripts and cybersecurity projects |

**Lessons Learned:**

- ext4 is a Linux filesystem supporting Unix permissions and ownership.
- Mounting makes a filesystem accessible through a directory.
- `/etc/fstab` configures persistent filesystem mounts.
- UUIDs provide more stable filesystem identification than device names.

---

## 🛡️ Phase 4 — Firewall Configuration

Verified that Fedora's `firewalld` service was enabled and running.

### Service Verification

```bash
sudo systemctl status firewalld
```

### Active Firewall Zones

```bash
sudo firewall-cmd --get-active-zones
```

### Review Firewall Rules

```bash
sudo firewall-cmd --list-all
```

### Observed Configuration

| Setting | Result |
|---|---|
| Active Zone | FedoraServer |
| Network Interface | wld0 |
| SSH | Allowed |
| Cockpit | Allowed |
| DHCPv6 Client | Allowed |
| Additional Explicit Ports | None |
| IPv4 Forwarding | Disabled |

Verified IPv4 forwarding:

```bash
sysctl net.ipv4.ip_forward
```

**Result:**

```text
net.ipv4.ip_forward = 0
```

Reviewed persistent firewall rules:

```bash
sudo firewall-cmd --permanent --zone=FedoraServer --list-all
```

**Lessons Learned:**

- `firewalld` manages network traffic permissions using zones.
- Firewall services define allowed network traffic.
- Runtime and permanent configurations can differ.
- Firewall forwarding settings and kernel IP forwarding are separate configurations.

---

## 🔄 Phase 5 — System Updates

Installed DNF5's automatic update plugin.

```bash
sudo dnf install dnf5-plugin-automatic
```

### Update Configuration

Configured automatic update downloads while retaining manual installation control.

```ini
[commands]
apply_updates = no
download_updates = yes
```

Enabled the update timer:

```bash
sudo systemctl enable --now dnf5-automatic.timer
```

Verified the schedule:

```bash
systemctl list-timers dnf5-automatic.timer
```

**Result:**

- ✅ Automatic update checks enabled.
- ✅ Update downloads enabled.
- ✅ Manual installation retained.
- ✅ No automatic reboot configured.

**Lessons Learned:**

- `systemd` manages background services and scheduled timers.
- Automatic update downloads improve update readiness.
- Manual installation allows greater control over system changes.

---

## 📊 Phase 6 — Splunk Universal Forwarder Deployment

Installed Splunk Universal Forwarder on the Fedora ARM64 server to support centralized security log collection.

### Installation

Downloaded and installed Splunk Universal Forwarder 10.4.3 for Linux ARM64.

```bash
sudo dnf install ./splunkforwarder-10.4.3-4174a2deda5d.aarch64.rpm
```

Verified the version:

```bash
/opt/splunkforwarder/bin/splunk version
```

**Result:**

```text
Splunk Universal Forwarder 10.4.3
```

### Service Account Configuration

Configured Splunk to operate using the dedicated `splunkfwd` service account.

Started the forwarder:

```bash
sudo systemctl start SplunkForwarder
```

Verified:

```bash
sudo systemctl status SplunkForwarder --no-pager
```

**Result:** `active (running)`

### Configure Forwarding Destination

Created the following configuration file:

```text
/opt/splunkforwarder/etc/system/local/outputs.conf
```

Configuration:

```ini
[tcpout]
defaultGroup = home_lab

[tcpout:home_lab]
server = <splunk-server-ip>:9997
```

Verified TCP connectivity between the Fedora server and Splunk Enterprise:

```bash
nc -zv -w 5 <splunk-server-ip> 9997
```

Inspected Splunk internal logs:

```bash
sudo grep -Ei 'TcpOutputProc|Connected to idx' \
/opt/splunkforwarder/var/log/splunk/splunkd.log | tail -n 20
```

**Result:** Successful connections to the Splunk Enterprise receiving port.

### Configure Journal Permissions

Initially, the `splunkfwd` service account could not read system-wide SSH authentication logs.

Added the account to the `systemd-journal` group:

```bash
sudo usermod -aG systemd-journal splunkfwd
```

Verified membership:

```bash
id splunkfwd
```

Tested journal access:

```bash
sudo -u splunkfwd journalctl -u sshd -n 10 --no-pager
```

**Result:** The service account successfully retrieved SSH authentication events.

Restarted the forwarder to apply the updated group membership.

### SSH Journal Input

Created a journal input configuration in:

```text
/opt/splunkforwarder/etc/system/local/inputs.conf
```

Configured collection of SSH authentication events.

Restarted the Splunk Universal Forwarder successfully.

---

## 🧠 Skills Developed

### Linux Administration
- Fedora Linux installation and configuration
- User and group management
- File ownership and permissions
- Package management using DNF
- Filesystem formatting and mounting
- systemd service administration
- Scheduled system updates

### Security Engineering
- SSH hardening
- Public-key authentication
- Least-privilege service accounts
- Firewall configuration review
- Security logging permissions
- Network service verification

### Security Monitoring
- Splunk Universal Forwarder installation
- SIEM forwarding configuration
- Linux authentication log collection preparation
- Network connectivity troubleshooting
- Splunk internal log analysis

---

## 🚀 Future Improvements

- [ ] Create detection rules for repeated failed login attempts.
- [ ] Configure monitoring for sudo and privileged activity.
- [ ] Add additional Linux system log sources.
- [ ] Generate synthetic authentication events for detection testing.
- [ ] Build Python scripts for authentication log analysis.
- [ ] Create dashboards for Linux security monitoring.
- [ ] Document detection logic and investigation workflows.


