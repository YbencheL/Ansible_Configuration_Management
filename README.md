# Ansible Configuration Management

Automated infrastructure provisioning and deployment for Ubuntu servers using Ansible. This playbook configures system packages, deploys Nginx with HTTPS/SSL, and deploys website applications.

**Target Environment:** Azure Linux VMs (Ubuntu)  
**Server:** 135.225.128.201  
**Primary Protocol:** HTTPS (TLS 1.3)

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Project Structure](#project-structure)
3. [Prerequisites](#prerequisites)
4. [Configuration](#configuration)
5. [Deployment](#deployment)
6. [Roles Overview](#roles-overview)
7. [Nginx Configuration](#nginx-configuration)
8. [Security](#security)
9. [Troubleshooting](#troubleshooting)

---

## Quick Start

```bash
# 1. Install Ansible
sudo apt install ansible

# 2. Edit inventory and credentials
nano inventory.ini
ansible-vault edit group_vars/myserver/vault.yml

# 3. Run the playbook
ansible-playbook -i inventory.ini setup.yml --ask-vault-pass
```

---

## Project Structure

```
Ansible_Configuration_Management/
├── inventory.ini                 # Server inventory & variables
├── setup.yml                     # Main playbook (entry point)
├── Readme.md                     # This documentation
├── group_vars/
│   └── myserver/
│       └── vault.yml             # Encrypted secrets (Ansible Vault)
└── roles/
    ├── base/                     # System initialization
    │   └── tasks/main.yml        # Updates, security tools, fail2ban
    ├── nginx/                    # Web server setup
    │   ├── tasks/main.yml        # Installation, SSL certificate generation
    │   └── files/nginx.conf      # Nginx configuration with HTTPS and restarting
    ├── app/                      # Website deployment
    │   ├── tasks/main.yml        # Copy & extract website files
    │   └── files/website.zip     # Website tarball (Feane template)
    └── ssh/                      # SSH key-based authentication
        └── tasks/main.yml        # SSH key setup
```

---

## Prerequisites

### Control Machine (where you run Ansible)
- **Ansible** 2.9+ (`pip install ansible` or `apt install ansible`)
- **Python 3.6+**
- **SSH client** (for connecting to target)
- **ansible-vault** (included with Ansible)

### Target Machine (remote server)
- **Ubuntu 20.04 LTS** or later
- **SSH access** with username/password or keys
- **Sudo privileges** (for package installation & service management)
- **Port 22** (SSH) open for connections
- **Ports 80 & 443** (HTTP/HTTPS) open in firewall

### Azure Cloud Requirements
- Security Group must allow:
  - **Inbound: Port 443** (HTTPS)
  - **Inbound: Port 22** (SSH) - from your IP
  - **Outbound: All** (for package downloads)

---

## Configuration

### 1. Update Inventory (`inventory.ini`)

```ini
[myserver]
135.225.128.201          # Your server's public IP

[myserver:vars]
ansible_user=myserver    # SSH username
ansible_ssh_port=22      # SSH port (default: 22)
```

### 2. Set Up Vault (`group_vars/myserver/vault.yml`)

Create encrypted vault file with credentials:

```bash
ansible-vault create group_vars/myserver/vault.yml
```

Add required variables:

```yaml
# SSH password for initial setup
ssh_password: "your_secure_password_here"

# Sudo password (if different from SSH password)
ansible_become_password: "your_sudo_password_here"
```

### 3. Configure Nginx (`roles/nginx/files/nginx.conf`)

Current configuration:
- **Protocol:** HTTPS only (TLS 1.3)
- **Port:** 443
- **Root Directory:** `/var/www/html/feane-1.0.0`
- **Default Page:** `index.html`
- **SSL Certificate:** `/etc/nginx/ssl/inception.crt`
- **SSL Key:** `/etc/nginx/ssl/inception.key`

To modify:
1. Edit `roles/nginx/files/nginx.conf`
2. Update `server_name`, `root`, or SSL paths as needed
3. Run playbook to apply changes

---

## Deployment

### Option 1: Interactive Vault Password (Recommended for Testing)

```bash
ansible-playbook -i inventory.ini setup.yml --ask-vault-pass
```

### Option 2: Vault Password File

Create password file:
```bash
echo "your_vault_password" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

Run playbook:
```bash
ansible-playbook setup.yml --vault-password-file ~/.vault_pass
```

### Option 3: Vault Password from Environment

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook -i inventory.ini setup.yml
```

### Run Specific Roles Only

```bash
# Run only base role
ansible-playbook -i inventory.ini setup.yml --tags base --ask-vault-pass

# Run only nginx
ansible-playbook -i inventory.ini setup.yml --tags nginx --ask-vault-pass

# Run only app deployment
ansible-playbook -i inventory.ini setup.yml --tags app --ask-vault-pass
```

### Verify Playbook Without Applying

```bash
ansible-playbook -i inventory.ini setup.yml --ask-vault-pass --check
```

---

## Roles Overview

### **base** - System Initialization
**Purpose:** Prepare the server for application deployment

**Tasks:**
- Update apt package lists (`apt update`)
- Upgrade all packages (`apt upgrade -y`)
- Install essential tools:
  - `curl` - HTTP client
  - `git` - Version control
  - `fail2ban` - Brute-force protection

**Services:**
- Enables and starts `fail2ban`

**Location:** roles/base/tasks/main.yml

### **nginx** - Web Server Setup
**Purpose:** Install and configure Nginx with HTTPS support

**Tasks:**
1. Install Nginx package
2. Enable and start Nginx service
3. Generate self-signed SSL certificates (CN=135.225.128.201)
4. Copy custom Nginx configuration
5. Restart Nginx to apply config

**SSL Certificate Details:**
- **Type:** Self-signed X.509 certificate
- **Duration:** 365 days
- **Algorithm:** RSA
- **Cipher:** TLS 1.3 only
- **Location:** `/etc/nginx/ssl/`

**Location:** roles/nginx/tasks/main.yml  
**Config:** roles/nginx/files/nginx.conf

### **app** - Website Deployment
**Purpose:** Deploy website files from tarball to web root

**Tasks:**
1. Copy `website.zip` from control machine to `/home/myserver/`
2. Extract archive to `/var/www/html/`
3. Remove temporary zip file
4. Website accessible at `https://135.225.128.201/`

**Expected Structure:**
```
/var/www/html/
└── feane-1.0.0/
    ├── index.html
    ├── css/
    ├── js/
    └── ...
```

**Location:** roles/app/tasks/main.yml

### **ssh** - SSH Key Authentication
**Purpose:** Configure passwordless SSH access

**Tasks:**
- Install `sshpass` utility
- Copy SSH public key to server's `authorized_keys`
- Enable key-based authentication

**Requirements:**
- `ssh_password` variable in vault.yml
- SSH public key in control machine (`~/.ssh/id_rsa.pub`)

**Location:** roles/ssh/tasks/main.yml

---

## Nginx Configuration

### Current Setup

```
Server Block: myserver
├── Listen: 443 (HTTPS)
├── Server Name: 135.225.128.201
├── Document Root: /var/www/html/feane-1.0.0
├── SSL Protocol: TLSv1.3
├── Certificate: /etc/nginx/ssl/inception.crt
├── Private Key: /etc/nginx/ssl/inception.key
└── Location /
    ├── try_files: $uri $uri/ =404
    └── error_page 404: /index.html
```

### How It Works

1. **HTTPS Only:** Requests only accepted on port 443
2. **File Resolution:** First tries exact file, then directory
3. **SPA Support:** 404 errors redirected to `index.html` (single-page app)
4. **Certificate Warning:** Self-signed cert will trigger browser warning
   - Click "Advanced" → "Proceed" to continue

### Access URL

```
https://135.225.128.201/
```

⚠️ **Note:** You'll see certificate warnings since it's self-signed (not signed by certificate authority)

---

## Security

### Vault Best Practices

✅ **DO:**
- Encrypt all sensitive data in vault.yml
- Use strong vault passwords (20+ characters)
- Restrict vault password file permissions: `chmod 600 ~/.vault_pass`
- Store vault password securely (not in version control)
- Rotate vault passwords periodically

❌ **DON'T:**
- Commit vault passwords to Git
- Share vault passwords via email/chat
- Use weak passwords
- Store passwords in plaintext files
- Run playbooks with `--vault-password-file` in production scripts

### SSL/TLS Security

- **Self-signed certificates:** OK for testing/internal use only
- **Production:** Use Let's Encrypt or commercial CA certificates
- **Certificate update:** Modify `roles/nginx/files/nginx.conf` → rerun playbook
- **TLS 1.3:** Current standard (TLS 1.0/1.1/1.2 deprecated)

---

## Troubleshooting

### Connection Issues

**SSH Connection Refused**
```
fatal: [135.225.128.201]: UNREACHABLE!
```
**Solutions:**
- Verify IP address in `inventory.ini`
- Check SSH port (default 22)
- Ensure firewall allows port 22
- Test: `ssh -v myserver@135.225.128.201`

**Authentication Failed**
```
fatal: [myserver]: FAILED! => {"msg": "unsudoers"}
```
**Solutions:**
- Verify `ssh_password` in vault.yml is correct
- Ensure user has sudo privileges: `sudo usermod -aG sudo myserver`
- Test: `ssh -v myserver@135.225.128.201`

### Vault Issues

**Vault Password Not Matching**
```
fatal: [myserver]: FAILED! => {"msg": "Vault password did not match"}
```
**Solutions:**
- Re-enter vault password correctly
- Or use password file: `--vault-password-file ~/.vault_pass`
- Check file wasn't corrupted

**Vault File View**
```bash
# View encrypted content without decrypting
ansible-vault view group_vars/myserver/vault.yml --ask-vault-pass

# Edit vault
ansible-vault edit group_vars/myserver/vault.yml --ask-vault-pass
```

### Nginx/HTTPS Issues

**Port 443 Not Responding**
```bash
# Check if nginx is listening
sudo ss -tlnp | grep 443

# Check nginx status
sudo systemctl status nginx

# View nginx error log
sudo tail -50 /var/log/nginx/error.log
```

**Certificate Not Found**
```bash
# Verify SSL files exist
sudo ls -la /etc/nginx/ssl/
```

**Invalid Configuration**
```bash
# Test nginx config syntax
sudo nginx -t

# Reload (if valid)
sudo systemctl reload nginx
```

### Website Not Displaying

**404 Errors**
```bash
# Check website files exist
ls -la /var/www/html/feane-1.0.0/

# Check permissions
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

**Test Locally on Server**
```bash
curl -kv https://localhost/
curl -kv https://127.0.0.1/
```

### Azure Firewall Issues

If HTTPS times out:
1. Login to Azure Portal
2. Navigate to your VM's **Network** settings
3. Check **Inbound port rules**
4. Ensure port 443 is allowed from "Any" or your IP
5. Wait 1-2 minutes for changes to apply

## Useful Commands

```bash
# List all hosts
ansible-inventory --list

# Test connectivity
ansible -i inventory.ini all -m ping --ask-vault-pass

# Run ad-hoc command
ansible -i inventory.ini myserver -m shell -a "uptime" --ask-vault-pass

# Display facts about hosts
ansible -i inventory.ini myserver -m setup --ask-vault-pass

# Dry run (check mode)
ansible-playbook -i inventory.ini setup.yml --check --ask-vault-pass

# Increase verbosity
ansible-playbook -i inventory.ini setup.yml -vvv --ask-vault-pass

# List tasks without running
ansible-playbook -i inventory.ini setup.yml --list-tasks
```

---

## Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Vault Guide](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [HTTPS Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)
- [Azure VM Security](https://docs.microsoft.com/en-us/azure/virtual-machines/security-recommendations)

---

## Project insperation URL

```
https://roadmap.sh/projects/configuration-management
```

## License

This project is provided as-is for infrastructure automation purposes.

**Last Updated:** May 5, 2026
