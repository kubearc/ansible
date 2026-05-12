# Ansible Setup on RHEL — 1 Controller + 4 Managed Nodes

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![RHEL](https://img.shields.io/badge/RHEL-CC0000?style=for-the-badge&logo=redhat&logoColor=white)

A step-by-step guide to install and configure Ansible on Red Hat Enterprise Linux (RHEL) with **one Control Node** and **four Managed Nodes**.

---

## 📋 Lab Environment

| Role            | Hostname         | IP Address       |
|-----------------|------------------|------------------|
| Control Node    | controller.lab   | 192.168.1.10     |
| Managed Node 1  | node1.lab        | 192.168.1.11     |
| Managed Node 2  | node2.lab        | 192.168.1.12     |
| Managed Node 3  | node3.lab        | 192.168.1.13     |
| Managed Node 4  | node4.lab        | 192.168.1.14     |

> ⚠️ Replace the hostnames and IP addresses above with your actual values.

---

## Prerequisites

- RHEL 8 or RHEL 9 installed on all nodes
- A non-root sudo user on all nodes (or root access)
- All nodes can communicate with each other over the network
- Active RHEL subscription or access to required repos

---

## Part 1 — Install Ansible on the Control Node

> Run the following steps **only on the Controller node**.

### Step 1: Register and Subscribe the System (if not already done) or Configure yum localy 

```bash
sudo subscription-manager register --username <your-rhn-username> --password <your-rhn-password>
sudo subscription-manager attach --auto
```

### Step 2: Enable the Ansible Repository

**On RHEL 8:**
```bash
sudo subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms
```

**On RHEL 9:**
```bash
sudo subscription-manager repos --enable ansible-automation-platform-2.4-for-rhel-9-x86_64-rpms
```

> Alternatively, if you have EPEL access:
> ```bash
> sudo dnf install -y epel-release
> ```

### Step 3: Install Ansible

```bash
sudo dnf install -y ansible
```

### Step 4: Verify the Installation

```bash
ansible --version
```

Expected output example:
```
ansible [core 2.14.x]
  config file = /etc/ansible/ansible.cfg
  python version = 3.x.x
  ...
```

---

## Part 2 — Prepare All Managed Nodes

> Run the following steps on **each of the 4 managed nodes** (node1 through node4).

### Step 5: Ensure Python is Installed

Ansible requires Python on the managed nodes.

```bash
sudo dnf install -y python3
```

### Step 6: Create an Ansible User (Optional but Recommended)

```bash
sudo useradd admin
sudo passwd admin
```

Grant sudo privileges:
```bash
echo "admin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
```

---

## Part 3 — Configure SSH Key-Based Authentication

> Run the following steps on the **Control Node**.

### Step 7: Generate an SSH Key Pair from **admin** user

```bash
ssh-keygen 
```

Press `Enter` to accept the default location (`~/.ssh/id_rsa`) and set a passphrase (or leave blank).

### Step 8: Copy the Public Key to Each Managed Node

```bash
ssh-copy-id admin@192.168.1.11   # node1
ssh-copy-id admin@192.168.1.12   # node2
ssh-copy-id admin@192.168.1.13   # node3
ssh-copy-id admin@192.168.1.14   # node4
```

### Step 9: Test SSH Connectivity

```bash
ssh admin@192.168.1.11
ssh admin@192.168.1.12
ssh admin@192.168.1.13
ssh admin@192.168.1.14
```

You should be able to log in to each node **without a password**.

---

## Part 4 — Configure the Ansible Inventory

> Run on the **Control Node**.

### Step 10: Edit the Default Inventory File

```bash
sudo vim /etc/ansible/hosts
```

Add the following content:

```ini
[controller]
controller.lab ansible_host=192.168.1.10

[managed_nodes]
node1.lab ansible_host=192.168.1.11
node2.lab ansible_host=192.168.1.12
node3.lab ansible_host=192.168.1.13
node4.lab ansible_host=192.168.1.14

```

---


## Part 5 — Verify the Setup

### Step 11: Ping All Managed Nodes

```bash
ansible all -m ping
```

Expected output:

```
node1.lab | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node2.lab | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node3.lab | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node4.lab | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Step 12: Run an Ad-Hoc Command

Check uptime on all managed nodes:

```bash
ansible managed_nodes -m command -a "uptime" -b 
```

Check free memory:

```bash
ansible managed_nodes -m command -a "free -h" -b 
```

---

## Part 7 — Test with a Simple Playbook

### Step 13: Create a Test Playbook

```bash
vim ~/test-playbook.yml
```

```yaml
---
- name: Test Ansible Setup
  hosts: managed_nodes
  become: yes

  tasks:
    - name: Ensure the latest version of wget is installed
      ansible.builtin.dnf:
        name: wget
        state: latest

    - name: Print a success message
      ansible.builtin.debug:
        msg: "Ansible is working correctly on {{ inventory_hostname }}"
```

### Step 15: Run the Playbook

```bash
ansible-playbook ~/test-playbook.yml -b 
```

---

## 📁 Directory Structure (Reference)

```
/etc/ansible/
├── ansible.cfg       # Main configuration file
├── hosts             # Inventory file
└── roles/            # Roles directory (optional)

~/
└── test-playbook.yml # Your test playbook
```

---

## 🛠️ Troubleshooting

| Issue | Solution |
|-------|----------|
| `Permission denied (publickey)` | Re-run `ssh-copy-id` to the affected node |
| `Host key verification failed` | Set `host_key_checking = False` in `ansible.cfg` |
| `Python not found` | Install Python 3 on the managed node: `dnf install -y python3` |
| `sudo: no tty present` | Ensure `NOPASSWD` is set in `/etc/sudoers.d/ansible` |
| Node not reachable | Check firewall rules: `firewall-cmd --list-all` |

---

## 📚 References

- [Ansible Official Documentation](https://docs.ansible.com/)
- [Red Hat Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible)
- [RHEL System Roles](https://access.redhat.com/articles/3050101)


---

> ✅ **Setup Complete!** Your Ansible control node is now managing 4 RHEL nodes. You're ready to start writing playbooks and automating your infrastructure.
