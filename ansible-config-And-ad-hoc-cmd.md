# Ansible Basics: ansible.cfg Setup & Ad-Hoc Commands

A hands-on guide for students to configure Ansible and run their first ad-hoc commands.

---

## Prerequisites

- Ansible installed on the control node
- SSH access to managed nodes
- Python installed on all nodes

> **Install Ansible (if not already):**
> ```bash
> sudo apt update && sudo apt install ansible -y        # Debian/Ubuntu
> sudo dnf install ansible -y                           # RHEL/CentOS/Fedora
> pip install ansible                                   # via pip
> ```

---

## Step 1: Create the `ansible.cfg` File

T

### Create the file in your project directory:

```bash
mkdir ~/ansible && cd ~/ansible
vim ansible.cfg
```

### Paste the following configuration:

```ini
[defaults]
# Path to your inventory file
inventory       = 'path of your host file'

# Default remote user for SSH connections
remote_user     = admin

# Disable SSH host key checking (okay for labs, not for production)
host_key_checking = False

# Path to your private SSH key
private_key_file = ~/.ssh/id_rsa

# Output format — 'yaml' is more readable; use 'json' for scripting
stdout_callback = yaml

# Number of parallel connections
forks           = 10

# Log file (optional)
# log_path      = ./ansible.log

[privilege_escalation]
# Allow privilege escalation (sudo)
become          = True
become_method   = sudo
become_user     = root
become_ask_pass = False
```

> **Tip:** Run `ansible --version` to confirm which `ansible.cfg` Ansible is picking up.

---

## Step 2: Create the Inventory File

The inventory file lists the hosts (managed nodes) Ansible will connect to.

```bash
vim hosts
```

```ini
# Single host
192.168.1.10

# Named host
webserver ansible_host=192.168.1.11

# Group of hosts
[webservers]
web1 ansible_host=192.168.1.20
web2 ansible_host=192.168.1.21

[dbservers]
db1 ansible_host=192.168.1.30

# Group of groups
[all_servers:children]
webservers
dbservers

# Group variables
[webservers:vars]
ansible_user=ec2-user
```
**NOTE** This is the example of the Hosts file.

> **Verify your inventory:**
> ```bash
> ansible-inventory --list
> ansible-inventory --graph
> ```

---

## Step 3: Test Connectivity

Before running any commands, verify Ansible can reach your hosts:

```bash
ansible all -m ping
```

**Expected output:**
```yaml
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## Step 4: Ad-Hoc Commands

Ad-hoc commands let you run a single Ansible task without writing a playbook. The syntax is:

```
ansible <host-pattern> -m <module> -a "<arguments>"
```

---

### 🔹 System Information

```bash
# Check uptime on all hosts
ansible all -m command -a "uptime"

# Check disk usage
ansible all -m command -a "df -h"

# Check memory usage
ansible all -m command -a "free -m"

# Check OS details
ansible all -m setup -a "filter=ansible_distribution*"

# Gather all facts about a host
ansible web1 -m setup
```

---

### 🔹 File Operations

```bash
# Create a directory
ansible all -m file -a "path=/tmp/demo state=directory mode=0755"

# Create an empty file
ansible all -m file -a "path=/tmp/demo/hello.txt state=touch"

# Copy a file from control node to managed nodes
ansible all -m copy -a "src=./hello.txt dest=/tmp/hello.txt"

# Copy content directly (no local file needed)
ansible all -m copy -a "content='Hello from Ansible!\n' dest=/tmp/message.txt"

# Fetch a file from managed nodes to control node
ansible all -m fetch -a "src=/etc/hostname dest=./hostnames/ flat=no"

# Delete a file
ansible all -m file -a "path=/tmp/hello.txt state=absent"
```

---

### 🔹 Package Management

```bash
# Install a package (RHEL/CentOS)
ansible all -m yum -a "name=httpd state=present"

# Install a package (Debian/Ubuntu)
ansible all -m apt -a "name=nginx state=present"

# Remove a package
ansible all -m yum -a "name=httpd state=absent"

# Update all packages
ansible all -m yum -a "name=* state=latest"
```

---

### 🔹 Service Management

```bash
# Start a service
ansible all -m service -a "name=httpd state=started"

# Stop a service
ansible all -m service -a "name=httpd state=stopped"

# Restart a service
ansible all -m service -a "name=httpd state=restarted"

# Enable a service to start on boot
ansible all -m service -a "name=httpd enabled=true"

# Check service status
ansible all -m command -a "systemctl status httpd"
```

---

### 🔹 User Management

```bash
# Create a user
ansible all -m user -a "name=john state=present"

# Create a user with a home directory and shell
ansible all -m user -a "name=john shell=/bin/bash home=/home/john state=present"

# Delete a user
ansible all -m user -a "name=john state=absent remove=yes"

# Add user to a group
ansible all -m user -a "name=john groups=wheel append=yes"
```

---

### 🔹 Running Shell Commands

```bash
# Run a shell command (supports pipes, redirects)
ansible all -m shell -a "echo $HOSTNAME"

# Run a command as a different user
ansible all -m shell -a "whoami" --become --become-user=root

# Run command only on webservers group
ansible webservers -m command -a "uptime"
```

> **`command` vs `shell` module:**
> - `command` — safer, does **not** use the shell, no pipes/redirects
> - `shell` — uses `/bin/sh`, supports pipes, variables, redirects

---

### 🔹 Limiting Hosts

```bash
# Target a single host
ansible web1 -m ping

# Target a group
ansible webservers -m ping

# Target all hosts
ansible all -m ping

# Exclude a host
ansible all --limit '!web2' -m ping

# Limit to first 2 hosts in a group
ansible webservers --limit 'webservers[0:1]' -m ping
```

---

### 🔹 Useful Flags

| Flag | Description |
|------|-------------|
| `-m` | Module name |
| `-a` | Module arguments |
| `-i` | Specify inventory file |
| `-u` | Remote user |
| `-b` | Become (sudo) |
| `--become-user` | User to become |
| `-k` | Ask for SSH password |
| `-K` | Ask for sudo password |
| `-v / -vv / -vvv` | Verbosity (more `v` = more detail) |
| `--check` | Dry run (no changes made) |
| `--diff` | Show file diffs |

---

## Quick Reference Card

```bash
# Ping all hosts
ansible all -m ping

# Run a command
ansible all -m command -a "uptime"

# Copy a file
ansible all -m copy -a "src=file.txt dest=/tmp/file.txt"

# Install a package
ansible all -m yum -a "name=vim state=present" -b

# Start a service
ansible all -m service -a "name=sshd state=started" -b

# Gather facts
ansible all -m setup

# Check which config file is active
ansible --version
```

---

## Common Errors & Fixes

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `UNREACHABLE` | SSH not reachable | Check IP, firewall, SSH service |
| `Permission denied` | Wrong SSH key or user | Check `remote_user` and `private_key_file` |
| `Missing sudo password` | `become_ask_pass` not set | Set `become_ask_pass = False` or use `-K` |
| `No hosts matched` | Wrong host pattern | Check inventory with `ansible-inventory --list` |
| `Module not found` | Typo in module name | Run `ansible-doc -l` to list modules |

---

## Learn More

- 📖 [Ansible Documentation](https://docs.ansible.com/)
- 📦 [Ansible Galaxy](https://galaxy.ansible.com/) — community roles & collections
- 🔍 [`ansible-doc <module>`](https://docs.ansible.com/ansible/latest/collections/index_module.html) — built-in module help

```bash
# Get help on any module directly in the terminal
ansible-doc copy
ansible-doc yum
ansible-doc service
```

---

*Happy automating! 🚀*
