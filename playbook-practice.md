# Ansible Playbook — Practice Q&A

**Modules:** `yum` · `user` · `file` · `copy` · `lineinfile`

Read each question, write the playbook yourself, then expand the answer to check.

---

## yum Module

**Q1. Install the `httpd` package.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Install httpd
  hosts: all
  become: yes
  tasks:
    - name: Install httpd
      yum:
        name: httpd
        state: present
```
</details>

---

**Q2. Install `httpd` at the latest version.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Install httpd latest
  hosts: all
  become: yes
  tasks:
    - name: Install httpd at latest version
      yum:
        name: httpd
        state: latest
```
</details>

---

**Q3. Install `git`, `vim`, and `curl` in one task.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Install multiple packages
  hosts: all
  become: yes
  tasks:
    - name: Install git, vim, curl
      yum:
        name:
          - git
          - vim
          - curl
        state: present
```
</details>

---

**Q4. Remove the `telnet` package.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Remove telnet
  hosts: all
  become: yes
  tasks:
    - name: Remove telnet
      yum:
        name: telnet
        state: absent
```
</details>

---

**Q5. Install `nginx` and `firewalld`, and remove `vsftpd`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Manage packages
  hosts: all
  become: yes
  tasks:
    - name: Install nginx
      yum:
        name: nginx
        state: present

    - name: Install firewalld
      yum:
        name: firewalld
        state: present

    - name: Remove vsftpd
      yum:
        name: vsftpd
        state: absent
```
</details>

---

## user Module

**Q6. Create a user `alice` with shell `/bin/bash`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create user alice
  hosts: all
  become: yes
  tasks:
    - name: Create alice
      user:
        name: alice
        shell: /bin/bash
        state: present
```
</details>

---

**Q7. Create user `bob` and add him to the `developers` group without removing his existing groups.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create user bob
  hosts: all
  become: yes
  tasks:
    - name: Create bob
      user:
        name: bob
        shell: /bin/bash
        groups: developers
        append: yes
        state: present
```

> `append: yes` makes sure bob keeps his existing groups.
</details>

---

**Q8. Create user `deployer` with shell `/bin/bash` and add to group `wheel`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create deployer
  hosts: all
  become: yes
  tasks:
    - name: Create deployer user
      user:
        name: deployer
        shell: /bin/bash
        groups: wheel
        append: yes
        state: present
```
</details>

---

**Q9. Delete user `testuser` and remove their home directory.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Delete testuser
  hosts: all
  become: yes
  tasks:
    - name: Remove testuser
      user:
        name: testuser
        state: absent
        remove: yes
```
</details>

---

## file Module

**Q10. Create a directory `/opt/myapp`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create directory
  hosts: all
  become: yes
  tasks:
    - name: Create /opt/myapp
      file:
        path: /opt/myapp
        state: directory
```
</details>

---

**Q11. Create `/opt/project` with owner `alice`, group `developers`, mode `0775`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create project directory
  hosts: all
  become: yes
  tasks:
    - name: Create /opt/project
      file:
        path: /opt/project
        state: directory
        owner: alice
        group: developers
        mode: '0775'
```
</details>

---

**Q12. Change permissions on `/etc/hosts` to `0644` without changing its content.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Fix permissions
  hosts: all
  become: yes
  tasks:
    - name: Set /etc/hosts permissions
      file:
        path: /etc/hosts
        mode: '0644'
```
</details>

---

**Q13. Create a symlink at `/home/bob/project` pointing to `/opt/project`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create symlink
  hosts: all
  become: yes
  tasks:
    - name: Symlink for bob
      file:
        src: /opt/project
        dest: /home/bob/project
        state: link
```
</details>

---

**Q14. Delete the file `/tmp/oldfile.txt`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Delete file
  hosts: all
  become: yes
  tasks:
    - name: Remove oldfile.txt
      file:
        path: /tmp/oldfile.txt
        state: absent
```
</details>

---

## copy Module

**Q15. Copy local file `files/app.conf` to `/etc/myapp/app.conf` on the managed node.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Copy config file
  hosts: all
  become: yes
  tasks:
    - name: Copy app.conf
      copy:
        src: files/app.conf
        dest: /etc/myapp/app.conf
```
</details>

---

**Q16. Copy `files/nginx.conf` to `/etc/nginx/nginx.conf` with owner `root` and mode `0644`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Deploy nginx config
  hosts: all
  become: yes
  tasks:
    - name: Copy nginx.conf
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
```
</details>

---

**Q17. Write `"Authorised access only."` directly to `/etc/motd` without using a local file.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Set MOTD
  hosts: all
  become: yes
  tasks:
    - name: Write login banner
      copy:
        content: "Authorised access only.\n"
        dest: /etc/motd
```
</details>

---

**Q18. Copy `files/index.html` to `/var/www/html/index.html` with owner `deployer` and mode `0644`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Deploy index.html
  hosts: all
  become: yes
  tasks:
    - name: Copy index.html
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: deployer
        mode: '0644'
```
</details>

---

## lineinfile Module

**Q19. Ensure `PermitRootLogin no` is set in `/etc/ssh/sshd_config`. Replace any existing `PermitRootLogin` line.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Disable root login
  hosts: all
  become: yes
  tasks:
    - name: Set PermitRootLogin no
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present
```
</details>

---

**Q20. Ensure `PasswordAuthentication no` is set in `/etc/ssh/sshd_config`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Disable password auth
  hosts: all
  become: yes
  tasks:
    - name: Set PasswordAuthentication no
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
```
</details>

---

**Q21. Add `vm.swappiness=10` to `/etc/sysctl.conf`. Replace the line if it already exists.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Set swappiness
  hosts: all
  become: yes
  tasks:
    - name: Configure vm.swappiness
      lineinfile:
        path: /etc/sysctl.conf
        regexp: '^vm.swappiness'
        line: 'vm.swappiness=10'
        state: present
```
</details>

---

**Q22. Remove all lines containing the word `LEGACY` from `/etc/myapp/myapp.conf`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Remove LEGACY lines
  hosts: all
  become: yes
  tasks:
    - name: Remove LEGACY entries
      lineinfile:
        path: /etc/myapp/myapp.conf
        regexp: 'LEGACY'
        state: absent
```
</details>

---

**Q23. Add `ServerName myserver.local` to `/etc/nginx/nginx.conf`. Replace any existing `ServerName` line.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Set ServerName
  hosts: all
  become: yes
  tasks:
    - name: Set nginx ServerName
      lineinfile:
        path: /etc/nginx/nginx.conf
        regexp: '^ServerName'
        line: 'ServerName myserver.local'
        state: present
```
</details>

---

## Mixed Questions

**Q24. Create user `webadmin` and create `/var/www/html` owned by that user.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create webadmin and web root
  hosts: all
  become: yes
  tasks:
    - name: Create webadmin user
      user:
        name: webadmin
        shell: /bin/bash
        state: present

    - name: Create web root directory
      file:
        path: /var/www/html
        state: directory
        owner: webadmin
        mode: '0755'
```
</details>

---

**Q25. Install `nginx`, copy `files/nginx.conf` to `/etc/nginx/nginx.conf`, and set `ServerName myserver.local` in it.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Install and configure nginx
  hosts: all
  become: yes
  tasks:
    - name: Install nginx
      yum:
        name: nginx
        state: present

    - name: Deploy nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'

    - name: Set ServerName
      lineinfile:
        path: /etc/nginx/nginx.conf
        regexp: '^ServerName'
        line: 'ServerName myserver.local'
        state: present
```
</details>

---

## Capstone Playbooks

**Q26. Full web server setup** — Write one playbook that does all of the following:

1. Install `nginx`, `git`, and `firewalld`
2. Create user `webadmin` with shell `/bin/bash`, added to group `wheel`
3. Create `/var/www/html` — owner `webadmin`, mode `0755`
4. Copy `files/index.html` to `/var/www/html/index.html`
5. Set `ServerName mywebserver.local` in `/etc/nginx/nginx.conf`
6. Set `PermitRootLogin no` in `/etc/ssh/sshd_config`

<details>
<summary>Answer</summary>

```yaml
---
- name: Full web server setup
  hosts: all
  become: yes
  tasks:
    - name: Install packages
      yum:
        name:
          - nginx
          - git
          - firewalld
        state: present

    - name: Create webadmin user
      user:
        name: webadmin
        shell: /bin/bash
        groups: wheel
        append: yes
        state: present

    - name: Create web root
      file:
        path: /var/www/html
        state: directory
        owner: webadmin
        mode: '0755'

    - name: Deploy index.html
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: webadmin
        mode: '0644'

    - name: Set nginx ServerName
      lineinfile:
        path: /etc/nginx/nginx.conf
        regexp: '^ServerName'
        line: 'ServerName mywebserver.local'
        state: present

    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present
```
</details>

---

**Q27. Server hardening** — Write one playbook that does all of the following:

1. Remove `telnet` and `vsftpd`
2. Set `PermitRootLogin no` in `/etc/ssh/sshd_config`
3. Set `PasswordAuthentication no` in `/etc/ssh/sshd_config`
4. Set `MaxAuthTries 3` in `/etc/ssh/sshd_config`
5. Set `vm.swappiness=10` in `/etc/sysctl.conf`
6. Write a security warning to `/etc/motd`
7. Set permissions on `/etc/ssh/sshd_config` to `0600`

<details>
<summary>Answer</summary>

```yaml
---
- name: Server hardening
  hosts: all
  become: yes
  tasks:
    - name: Remove insecure packages
      yum:
        name:
          - telnet
          - vsftpd
        state: absent

    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present

    - name: Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present

    - name: Limit auth attempts
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^MaxAuthTries'
        line: 'MaxAuthTries 3'
        state: present

    - name: Set vm.swappiness
      lineinfile:
        path: /etc/sysctl.conf
        regexp: '^vm.swappiness'
        line: 'vm.swappiness=10'
        state: present

    - name: Set security banner
      copy:
        content: "Authorised access only. All activity is monitored.\n"
        dest: /etc/motd

    - name: Secure sshd_config permissions
      file:
        path: /etc/ssh/sshd_config
        mode: '0600'
```
</details>

---

*27 questions — 5 modules — 2 capstone playbooks*
