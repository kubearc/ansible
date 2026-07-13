# Ansible Vault Practice Lab

## Objective

In this lab, you will learn how to secure sensitive information using **Ansible Vault**.

By the end of this exercise, you should be able to:

- Create encrypted files
- Encrypt existing files
- View and edit encrypted files
- Rekey encrypted files
- Encrypt individual variables
- Use encrypted variable files in playbooks
- Execute playbooks using Ansible Vault

> **Security tip:** Any Vault password file you create in this lab (e.g. `vault_password.txt`) should be locked down with `chmod 600` and never committed to Git. Add it to your `.gitignore` before you start:
> ```text
> vault_password.txt
> *.retry
> ```

> **Note:** Solutions use a sample password of `RedHat@123` — replace it with your own during practice. Try each task yourself before expanding the solution.

---

# Prerequisites

- RHEL / CentOS / Rocky Linux
- Ansible installed
- One managed node configured
- Passwordless SSH configured
- Basic knowledge of Ansible Playbooks

Verify Ansible installation:

```bash
ansible --version
```

Verify connectivity:

```bash
ansible all -m ping
```

---

# Lab Directory Structure

Create the following directory.

```text
vault-lab/
├── inventory
├── playbook.yml
├── secret.yml
├── database.yml
├── vars.yml
└── vault_password.txt
```

---

# Task 1 - Create an Encrypted File

Create a new encrypted file named **secret.yml**.

Store the following variables:

- username
- password
- department

Verify that the file is encrypted.

<details>
<summary>Solution</summary>

Create a new encrypted file.

```bash
ansible-vault create secret.yml
```

Example content:

```yaml
username: admin
password: RedHat@123
department: IT
```

Verify:

```bash
cat secret.yml
```

Expected Output:

```text
$ANSIBLE_VAULT;1.1;AES256
xxxxxxxxxxxxxxxxxxxxxxxx
```

</details>

---

# Task 2 - View an Encrypted File

View the contents of **secret.yml** without decrypting it permanently.

<details>
<summary>Solution</summary>

```bash
ansible-vault view secret.yml
```

Expected Output

```yaml
username: admin
password: RedHat@123
department: IT
```

</details>

---

# Task 3 - Edit an Encrypted File

Modify the following values:

- Change the password
- Add a new variable named **location**

Save the changes.

<details>
<summary>Solution</summary>

```bash
ansible-vault edit secret.yml
```

Modify it to:

```yaml
username: admin
password: NewPassword@123
department: IT
location: Ludhiana
```

</details>

---

# Task 4 - Encrypt an Existing File

Create a file named **database.yml**.

Add the following variables:

- db_name
- db_user
- db_password

Encrypt the file.

Verify that the contents are no longer readable.

<details>
<summary>Solution</summary>

Create the file.

database.yml

```yaml
db_name: employee
db_user: root
db_password: RedHat@123
```

Encrypt it.

```bash
ansible-vault encrypt database.yml
```

Verify.

```bash
cat database.yml
```

</details>

---

# Task 5 - Decrypt a File

Decrypt **database.yml**.

Verify that the original content is restored.

<details>
<summary>Solution</summary>

```bash
ansible-vault decrypt database.yml
```

Verify

```bash
cat database.yml
```

Output

```yaml
db_name: employee
db_user: root
db_password: RedHat@123
```

</details>

---

# Task 6 - Change the Vault Password

Change the password used for **secret.yml**.

Verify that the old password no longer works.

<details>
<summary>Solution</summary>

```bash
ansible-vault rekey secret.yml
```

It asks for

```
Current password:
New password:
Confirm password:
```

</details>

---

# Task 7 - Encrypt a Single Variable

Encrypt only the variable below.

```
api_key
```

Store it inside **vars.yml**.

Do not encrypt the complete file.

<details>
<summary>Solution</summary>

```bash
ansible-vault encrypt_string 'ABCD1234' --name api_key
```

Output

```yaml
api_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          3138393339...
```

Save it in

```text
vars.yml
```

</details>

---

# Task 8 - Use Vault in a Playbook

Create a playbook that loads variables from **secret.yml**.

Display the following using the debug module.

- username
- department
- location

Run the playbook using the Vault password.

<details>
<summary>Solution</summary>

Inventory

```text
localhost ansible_connection=local
```

secret.yml

```yaml
username: admin
password: RedHat@123
department: IT
location: Ludhiana
```

Encrypt

```bash
ansible-vault encrypt secret.yml
```

playbook.yml

```yaml
---
- name: Vault Demo
  hosts: localhost
  gather_facts: false

  vars_files:
    - secret.yml

  tasks:

    - name: Show Username
      debug:
        var: username

    - name: Show Department
      debug:
        var: department

    - name: Show Location
      debug:
        var: location
```

Run

```bash
ansible-playbook -i inventory playbook.yml --ask-vault-pass
```

</details>

---

# Task 9 - Password File

Create a password file named **vault_password.txt**.

Restrict its permissions so only you can read it.

Run the playbook using the password file instead of entering the password manually.

<details>
<summary>Solution</summary>

Create

```text
vault_password.txt
```

Content

```text
RedHat123
```

Lock down the permissions so only you can read it:

```bash
chmod 600 vault_password.txt
```

Run

```bash
ansible-playbook \
-i inventory playbook.yml \
--vault-password-file vault_password.txt
```

</details>

---

# Task 10 - Environment Variable

Configure Ansible so that it automatically reads the Vault password from the password file.

Run the playbook without providing any Vault-related options.

<details>
<summary>Solution</summary>

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=vault_password.txt
```

Now simply execute

```bash
ansible-playbook -i inventory playbook.yml
```

</details>

---

# Task 11 - Multiple Encrypted Files

Create two encrypted files.

```
dev.yml
prod.yml
```

Each file should contain:

- username
- password
- environment

Create two playbooks (e.g. **deploy_dev.yml** and **deploy_prod.yml**).

One should use **dev.yml**.

The second should use **prod.yml**.

<details>
<summary>Solution</summary>

Create

```bash
ansible-vault create dev.yml
```

```yaml
username: devuser
password: Dev@123
environment: Development
```

Create

```bash
ansible-vault create prod.yml
```

```yaml
username: produser
password: Prod@123
environment: Production
```

deploy_dev.yml (Development Playbook)

```yaml
---
- hosts: localhost
  gather_facts: false

  vars_files:
    - dev.yml

  tasks:

    - debug:
        var: environment
```

deploy_prod.yml (Production Playbook)

```yaml
---
- hosts: localhost
  gather_facts: false

  vars_files:
    - prod.yml

  tasks:

    - debug:
        var: environment
```

Execute

```bash
ansible-playbook deploy_dev.yml --ask-vault-pass
```

or

```bash
ansible-playbook deploy_prod.yml --ask-vault-pass
```

</details>

---

# Task 12 - Complete Challenge

Create the following project.

```text
project/
│
├── inventory
├── playbook.yml
├── vars/
│   ├── users.yml
│   ├── database.yml
│   └── api.yml
└── vault_password.txt
```

Requirements:

- users.yml must be encrypted.
- database.yml must be encrypted.
- api.yml should contain only one encrypted variable.
- The playbook should load all variables.
- Display all variables using the debug module.
- Execute the playbook successfully using the vault password file.

<details>
<summary>Solution</summary>

Project Structure

```text
project/
│
├── inventory
├── playbook.yml
├── vars/
│   ├── users.yml
│   ├── database.yml
│   └── api.yml
└── vault_password.txt
```

users.yml

```yaml
username: admin
department: Linux
```

Encrypt

```bash
ansible-vault encrypt vars/users.yml
```

database.yml

```yaml
db_name: employee
db_user: root
db_password: RedHat@123
```

Encrypt

```bash
ansible-vault encrypt vars/database.yml
```

api.yml

```yaml
api_key: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      xxxxxxxxxxxxxxxxxxxxx
```

Generate using

```bash
ansible-vault encrypt_string 'TOKEN12345' --name api_key
```

playbook.yml

```yaml
---
- hosts: localhost
  gather_facts: false

  vars_files:
    - vars/users.yml
    - vars/database.yml
    - vars/api.yml

  tasks:

    - debug:
        var: username

    - debug:
        var: department

    - debug:
        var: db_name

    - debug:
        var: db_user

    - debug:
        var: db_password

    - debug:
        var: api_key
```

Lock down the vault password file:

```bash
chmod 600 vault_password.txt
```

Run

```bash
ansible-playbook \
-i inventory playbook.yml \
--vault-password-file vault_password.txt
```

</details>

---

# Bonus Challenge

Create a playbook named **apache.yml** that installs Apache.

The following values must come from an encrypted Vault file (e.g. **mysql.yml**).

- Server Administrator
- Server Name
- Document Root

Verify that the playbook runs successfully.

<details>
<summary>Solution</summary>

mysql.yml

```yaml
mysql_root_password: RedHat@123
server_admin: admin@example.com
server_name: webserver.example.com
document_root: /var/www/html
```

Encrypt

```bash
ansible-vault encrypt mysql.yml
```

apache.yml (Playbook)

```yaml
---
- hosts: localhost
  become: yes

  vars_files:
    - mysql.yml

  tasks:

    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Create Document Root
      file:
        path: "{{ document_root }}"
        state: directory

    - name: Create index.html
      copy:
        dest: "{{ document_root }}/index.html"
        content: |
          Welcome to {{ server_name }}

    - name: Start Apache
      service:
        name: httpd
        state: started
        enabled: yes
```

Execute

```bash
ansible-playbook apache.yml --ask-vault-pass
```

</details>

---

# Interview Practice

Answer the following questions.

1. What is Ansible Vault?
2. Why is Ansible Vault used?
3. What is the difference between encrypting a file and encrypting a string?
4. Which command is used to edit an encrypted file?
5. Which command changes the Vault password?
6. How do you execute a playbook that uses encrypted variables?
7. What is the purpose of a Vault password file?
8. Can multiple Vault files be used in one playbook?
9. Why should the Vault password file never be committed to Git?
10. What are the best practices for managing secrets in Ansible?

---

# Command Cheat Sheet

| Task | Command |
|------|---------|
| Create Vault | `ansible-vault create file.yml` |
| View | `ansible-vault view file.yml` |
| Edit | `ansible-vault edit file.yml` |
| Encrypt | `ansible-vault encrypt file.yml` |
| Decrypt | `ansible-vault decrypt file.yml` |
| Rekey | `ansible-vault rekey file.yml` |
| Encrypt String | `ansible-vault encrypt_string` |
| Ask Password | `--ask-vault-pass` |
| Password File | `--vault-password-file file` |
| Environment Variable | `export ANSIBLE_VAULT_PASSWORD_FILE=file` |

---
You have successfully practiced:

- Creating encrypted files
- Encrypting existing files
- Viewing encrypted files
- Editing encrypted files
- Decrypting files
- Rekeying Vault passwords
- Encrypting single variables
- Running playbooks with Vault
- Using Vault password files
- Managing multiple Vault files
- Securing and excluding vault password files from version control

----
