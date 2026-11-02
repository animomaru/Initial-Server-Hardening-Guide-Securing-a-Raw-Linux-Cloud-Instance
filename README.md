# Initial Server Hardening Guide: Securing a Raw Linux Cloud Instance

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Linux Platform](https://img.shields.io/badge/OS-Ubuntu%20%7C%20Debian%20%7C%20RHEL-orange.svg)](https://www.kernel.org/)
[![Security Posture](https://img.shields.io/badge/Security-Hardened-brightgreen.svg)](#-core-security-pillars)

A production-ready framework and step-by-step operational blueprint for securing a newly provisioned Linux cloud instance (AWS EC2, GCP Compute Engine, DigitalOcean, Linode) against common attack vectors before exposing it to the public internet.

> 📖 **Project Wiki**: For in-depth tutorials, deep-dive explanations, and detailed configuration breakdowns, check out our [Project Wiki](../../wiki).

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Core Security Pillars](#-core-security-pillars)
- [Quick Start / Execution Checklist](#-quick-start--execution-checklist)
- [Repository Structure](#-repository-structure)
- [Verification \& Auditing](#-verification--auditing)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🛡️ Overview

Spinning up a raw cloud instance leaves it immediately vulnerable to automated scanners, credential brute-forcing, and unconfigured service defaults. Within minutes of provisioning, public IPv4 addresses are routinely targeted.

This repository provides a structured approach to transition a fresh OS deployment into a hardened, production-capable environment using battle-tested configuration standards and security best practices.

---

## 🔑 Core Security Pillars

This guide covers the critical initial setup across five main security layers:

```
                      ┌─────────────────────────────────────────┐
                      │    1. Access Control & Identity        │
                      │  (Key-only SSH, Disable Root Login)     │
                      └────────────────────┬────────────────────┘
                                           │
                      ┌────────────────────▼────────────────────┐
                      │    2. Network Perimeter Defense         │
                      │  (Default-Deny UFW / Firewalld Rules)   │
                      └────────────────────┬────────────────────┘
                                           │
                      ┌────────────────────▼────────────────────┐
                      │    3. Intrusion Prevention              │
                      │  (Fail2ban, Log Watchers, Rate Limit)  │
                      └────────────────────┬────────────────────┘
                                           │
                      ┌────────────────────▼────────────────────┐
                      │    4. OS & Kernel Hardening             │
                      │  (Sysctl tuning, Secure Shared Memory) │
                      └────────────────────┬────────────────────┘
                                           │
                      ┌────────────────────▼────────────────────┐
                      │    5. Patch & Audit Management          │
                      │  (Unattended Upgrades, Auditd Logs)     │
                      └─────────────────────────────────────────┘
```

1. **Access Control & Identity**
   * Absolute restriction of direct `root` login over SSH.
   * Disabling password authentication in favor of Ed25519 / RSA 4096-bit SSH keys.
   * Implementing custom SSH listening ports and restricting allowed SSH users/groups.
2. **Network Perimeter Defense**
   * Configuring strict incoming `DROP`/`REJECT` policies via `ufw` or `firewalld`.
   * Whitelisting only explicitly required application ports (e.g., `80/TCP`, `443/TCP`).
3. **Intrusion Prevention & Rate Limiting**
   * Deploying `fail2ban` to automatically dynamic-ban IP addresses exhibiting malicious behavior.
   * System log auditing with `auditd`.
4. **Kernel & OS Hardening**
   * Hardening networking stack parameters via sysctl (`/etc/sysctl.conf`) against SYN floods, IP spoofing, and ICMP redirects.
   * Securing shared memory mounts (`/dev/shm`) with `noexec,nosuid,nodev`.
5. **Patch Management**
   * Automated security updates via `unattended-upgrades`.

---

## 🚀 Quick Start / Execution Checklist

Follow these core terminal steps for an initial manual run on Debian/Ubuntu systems:

```bash
# 1. Update existing system packages
sudo apt update && sudo apt upgrade -y

# 2. Create a dedicated non-root administrative user
sudo adduser secureuser
sudo usermod -aG sudo secureuser

# 3. Import SSH keys for the new user
sudo mkdir -p /home/secureuser/.ssh
sudo cp ~/.ssh/authorized_keys /home/secureuser/.ssh/
sudo chown -R secureuser:secureuser /home/secureuser/.ssh
sudo chmod 700 /home/secureuser/.ssh
sudo chmod 600 /home/secureuser/.ssh/authorized_keys

# 4. Enforce SSH Hardening (/etc/ssh/sshd_config)
# Edit configuration settings:
#   PermitRootLogin no
#   PasswordAuthentication no
#   PubkeyAuthentication yes
sudo systemctl restart sshd

# 5. Configure Default Firewall (UFW)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp # Update if using a non-standard SSH port!
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# 6. Install & Activate Fail2ban
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
```

---

## 📁 Repository Structure

```text
.
├── configs/                  # Hardened baseline config templates
│   ├── sshd_config           # Hardened OpenSSH server configuration
│   ├── sysctl.conf           # Kernel parameter hardening (IP spoofing, SYN flood defense)
│   └── fail2ban.local        # Custom fail2ban jail definitions
├── scripts/                  # Automation scripts
│   ├── harden.sh             # Interactive Bash setup script
│   └── audit.sh              # Quick local health & configuration checker
├── checklists/               # Manual verification sheets
│   └── pre-flight.md         # Pre-deployment checklist
├── LICENSE                   # Open-source license details
└── README.md                 # Repository entry point
```

---

## 🔍 Verification & Auditing

After applying the configuration, validate your server's security posture with these auditing tools:

* **OpenSSH Security Audit:** Validate key exchanges and ciphers using [SSH-Audit](https://github.com/jtesta/ssh-audit).
  ```bash
  python3 ssh-audit.py your-server-ip
  ```
* **External Port Verification:** Run Nmap externally to confirm no unintended ports are listening.
  ```bash
  nmap -Pn -sT your-server-ip
  ```
* **Intrusion Prevention Status:** Check `fail2ban` active status and banned list.
  ```bash
  sudo fail2ban-client status sshd
  ```

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the [issues page](../../issues) if you'd like to report bugs or suggest enhancements.

---

## 📄 License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for more information.
