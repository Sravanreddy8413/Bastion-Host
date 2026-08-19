

# Bastion-Host

Here is a complete, production-ready README.md file tailored for this project, including setup steps, configuration details, and stretch goal guides.

Markdown
# Secure Bastion Host Architecture & Configuration

A comprehensive guide to deploying and configuring a secure Bastion Host (Jump Box) to manage administrative SSH access to isolated private infrastructure without exposing target servers directly to the public internet.

---

## Architecture Overview

                  +-------------------+
                  |   Local Machine   |
                  +---------+---------+
                            |
               SSH Tunnel   | (ProxyJump / Port 22)
                            v
            +---------------+---------------+
            |         Public Subnet         |
            |  +-------------------------+  |
            |  |       Bastion Host      |  |
            |  |   (Public IP: X.X.X.X)  |  |
            |  +------------+------------+  |
            +---------------|---------------+
                            |
                 Internal   | SSH (Port 22)
                 Subnet     v
            +---------------+---------------+
            |         Private Subnet        |
            |  +-------------------------+  |
            |  |      Private Server     |  |
            |  |  (Private IP: 10.0.X.X) |  |
            |  +-------------------------+  |
            +-------------------------------+

* **Bastion Host:** Located in a public subnet, exposed to Port 22 with hardened SSH configurations and access controls.
* **Private Server:** Located in an isolated private subnet with no public IP. Accepts incoming SSH connections **only** from the Bastion Host's private IP address.

---

## Prerequisites

* Active cloud provider account (AWS, DigitalOcean, GCP, Azure, etc.)
* `OpenSSH` client installed on local machine
* Generated SSH Keypair (`ssh-keygen -t ed25519`)
* Basic familiarity with Linux CLI and networking concepts

---

## Step-by-Step Implementation

### Step 1: Network & Provisioning Setup

1. **Provision Virtual Machines:**
   * **Bastion Host:** Attach a Public IP address.
   * **Private Server:** Assign to the same Virtual Private Cloud (VPC) / Network, but give it **only** a Private IP address.

2. **Configure Security Groups / Firewalls:**
   * **Bastion Security Group:**
     * Inbound: Allow SSH (`Port 22`) from your trusted local IP address (`your-home-ip/32`).
     * Outbound: Allow SSH (`Port 22`) to the private subnet range.
   * **Private Server Security Group:**
     * Inbound: Allow SSH (`Port 22`) strictly from the **Bastion Host's internal IP / Security Group**.
     * Outbound: Block all unnecessary inbound traffic.

---

### Step 2: SSH Key Setup & Server Hardening

1. **Deploy Public Key:**
   Copy your SSH public key to both servers:
   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519.pub <user>@<bastion-public-ip>
Disable Password Authentication (Both Servers):
Edit /etc/ssh/sshd_config:

Ini, TOML
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
Restart the SSH daemon:

Bash
sudo systemctl restart sshd
Step 3: Local Client ProxyJump Configuration
Configure your local SSH client to seamlessly jump through the bastion host without manually chaining SSH commands or placing private keys on the intermediate bastion server.

Add the following block to ~/.ssh/config on your Local Machine:

Bash
Host bastion
    HostName <BASTION-PUBLIC-IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

Host private-server
    HostName <PRIVATE-SERVER-PRIVATE-IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion
Usage Instructions
Connect directly to the Bastion Host:

Bash
ssh bastion
Connect to the Private Server (via Bastion ProxyJump):

Bash
ssh private-server
Transfer Files to Private Server via SCP:

Bash
scp -J bastion ./local-file.txt private-server:/tmp/
Monitoring & Intrusion Prevention
Setting up Fail2ban on Bastion Host
To automatically block IPs that display suspicious behavior (e.g., repeated failed SSH attempts):

Install Fail2ban:

Bash
sudo apt update && sudo apt install fail2ban -y
Create Local Configuration:

Bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
Configure SSH Jail:
Edit /etc/fail2ban/jail.local:

Ini, TOML
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1h
findtime = 10m
Enable and Start Service:

Bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
Stretch Goals & Advanced Security Hardening
1. Enable MFA (Google Authenticator) on Bastion
Install the PAM module:

Bash
sudo apt install libpam-google-authenticator -y
Run generator for your user:

Bash
google-authenticator
Enable PAM in /etc/pam.d/sshd:

Ini, TOML
auth required pam_google_authenticator.so
Update /etc/ssh/sshd_config:

Ini, TOML
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
Restart SSH service:

Bash
sudo systemctl restart ssh
2. Granular Traffic Filtering with iptables
Restrict forwarding traffic strictly to the target private server on port 22:

Bash
# Set default drop policy for forwarding
sudo iptables -P FORWARD DROP

# Allow established connections
sudo iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow forwarding from local machine subnet to private server on port 22
sudo iptables -A FORWARD -p tcp -i eth0 -o eth1 -d <PRIVATE-SERVER-IP> --dport 22 -j ACCEPT
3. Automated Deployment Options
Infrastructure deployments for this project can be automated using:

Terraform: To declare VPC, subnets, Security Groups, and EC2/Compute instances.

Ansible: To provision sshd_config, users, keys, and install fail2ban.
