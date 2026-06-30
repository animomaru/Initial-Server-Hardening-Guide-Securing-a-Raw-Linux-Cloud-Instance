# Initial Server Hardening Guide: Securing a Raw Linux Cloud Instance

When you provision a raw Linux instance on a cloud provider like DigitalOcean or Linode, it is immediately exposed to the public internet. Within minutes, automated bots will begin scanning your IP address for vulnerabilities, attempting brute-force SSH attacks on your `root` account.

Leaving a cloud instance in its default state presents a significant security risk. This guide provides a step-by-step blueprint to secure a fresh Ubuntu server. You will implement standard infrastructure security practices: disabling root access, enforcing cryptographic authentication, isolating network entry points, and establishing automated intrusion prevention.

---

## Prerequisites

Before beginning this guide, ensure you have the following:

* A newly deployed cloud instance running **Ubuntu 24.04 LTS** (or 22.04 LTS).
* Initial `root` access to the instance via your cloud provider's console or default credentials.
* An SSH key pair generated on your local machine.

---

## Step 1 — Creating a Non-Root User with Sudo Privileges

Operating entirely as the `root` user is dangerous; any accidental command or compromised process has absolute authority over your system. We will create a dedicated, limited user account and grant it administrative escalation privileges via `sudo`.

First, log in to your new cloud instance as `root`:

```bash
ssh root@your_server_ip

```

Create a new user account. Replace `user_account` with your preferred username:

```bash
adduser user_account

```

*You will be prompted to set a strong password and enter optional user details. Ensure this password is secure, as it will authorize administrative commands.*

Add the newly created user to the `sudo` system group to grant administrative permissions:

```bash
usermod -aG sudo user_account

```

> **Security Note:** While this user can execute administrative tasks using the `sudo` prefix, they will be forced to input their password to verify identity before any root-level command is executed. This prevents unauthorized ambient privilege execution.

---

## Step 2 — Enforcing SSH-Key-Only Authentication

Passwords can be cracked via brute-force or dictionary attacks. Cryptographic SSH key pairs are virtually impossible to brute-force. We will copy your local public key to the new user and disable password authentication entirely.

On your **local machine** (not the cloud server), copy your public key to the new server using the `ssh-copy-id` utility:

```bash
ssh-copy-id user_account@your_server_ip

```

*Enter the password you created for the user in Step 1 to complete the transfer.*

Now, test that you can log in **without a password** from your local machine:

```bash
ssh user_account@your_server_ip

```

Once successfully logged in as your new user, open the SSH daemon configuration file to disable password logins across the system:

```bash
sudo nano /etc/ssh/sshd_config

```

Scroll through the file or use `Ctrl+W` to locate the following directives. Modify them to match the configurations below:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

```

> **Security Context:** Disabling `PermitRootLogin` completely shuts down the primary target for automated internet bots. Disabling `PasswordAuthentication` ensures that anyone attempting to connect without an authorized cryptographic key pair is immediately rejected at the handshake level.

Save and close the file (`Ctrl+O`, then `Enter`, then `Ctrl+X`).

Test the syntax of your configuration before restarting the service to avoid locking yourself out:

```bash
sudo sshd -t

```

*If this returns no output, the syntax is valid.*

Restart the SSH daemon to apply the new rules:

```bash
sudo systemctl restart ssh

```

---

## Step 3 — Constructing an Aggressive Firewall Configuration via UFW

An unconfigured server exposes internal application ports to the public internet. We will use UFW (Uncomplicated Firewall) to establish a "Default Deny" network posture—blocking all incoming traffic except for explicitly whitelisted services.

By default, ensure UFW is set to block all incoming traffic and allow all outgoing traffic:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

```

Explicitly allow SSH connections so your current session does not drop when the firewall initializes:

```bash
sudo ufw allow OpenSSH

```

*If you host web services later, you would open web traffic using `sudo ufw allow proto tcp from any to any port 80,443`.*

Enable the firewall rules:

```bash
sudo ufw enable

```

*Press `y` and `Enter` when warned that the command may disrupt existing SSH connections.*

### Verification

Verify that your firewall policy is active and inspect the open surface area:

```bash
sudo ufw status verbose

```

**Expected Output:**

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere                  
22/tcp (OpenSSH) (v6)      ALLOW IN    Anywhere (v6)

```

---

## Step 4 — Implementing Active Intrusion Prevention with Fail2Ban

While your firewall protects unutilized ports, your SSH port (22) must remain open. To prevent automated bots from spamming your SSH port and consuming system resources, we will install Fail2Ban. Fail2Ban monitors system logs in real-time and dynamically updates firewall rules to block IPs showing malicious intent.

Install the Fail2Ban package:

```bash
sudo apt update
sudo apt install fail2ban -y

```

Fail2Ban uses a `.conf` configuration file, but updates should be written to a `.local` file to prevent updates from overwriting your custom rules. Copy the default configuration:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

```

Open the new configuration file:

```bash
sudo nano /etc/fail2ban/jail.local

```

Scroll down to the `[DEFAULT]` block and configure the global ban settings to be aggressively defensive:

```text
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3

```

* **`bantime`**: The duration an offending IP address is blocked (set to 1 hour).
* **`findtime`**: The historical window Fail2Ban looks at to count failed attempts (10 minutes).
* **`maxretry`**: The number of failures permitted within the `findtime` before a ban triggers (3 attempts).

Next, ensure the SSH protection loop is explicitly enabled. Scroll down to the `[sshd]` section and add the `enabled = true` directive:

```text
[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s

```

Save and close the file. Enable and start the background service tracking system:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

```

### Verification

Check the operational status of your intrusion prevention engine to confirm it is actively monitoring the SSH traffic pipeline:

```bash
sudo fail2ban-client status sshd

```

**Expected Output:**

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:   

```

---

## Conclusion

Your cloud instance has moved from a default, vulnerable configuration to a hardened, defense-in-depth security state. You have successfully:

1. Eliminated raw `root` command execution vulnerabilities.
2. Swapped weak password mechanics for un-brutable cryptographic handshakes.
3. Locked down peripheral network boundaries using a default-deny firewall posture.
4. Installed an automated active-defense system to drop scanners looking for access points.
