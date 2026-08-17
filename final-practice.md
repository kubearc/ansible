# RHCE 9 Practice Exam — Full Solutions

All paths assume control node work is done as `student` in `/home/student/ansible/`
unless a task explicitly says otherwise (task 5 and 16 mention `/home/admin/ansible/` —
adjust the base dir to match whatever your exam gives you; the content is identical).

---

## 1. Install & Configure Ansible

```bash
sudo dnf install -y ansible-core
mkdir -p /home/student/ansible/{roles,mycollection}
cd /home/student/ansible
```

**`/home/student/ansible/inventory`**
```ini
[dev]
node1

[test]
node2

[prod]
node3
node4

[balancers]
node5

[webservers:children]
prod
```

**`/home/student/ansible/ansible.cfg`**
```ini
[defaults]
inventory       = /home/student/ansible/inventory
roles_path      = /home/student/ansible/roles
collections_path = /home/student/ansible/mycollection
```

---

## 2. `yum_repo.yml`

```yaml
---
- name: Configure internal repositories
  hosts: all
  become: true
  tasks:
    - name: Add baseos-internal repo
      ansible.builtin.yum_repository:
        name: baseos-internal
        description: Baseos
        baseurl: http://content/rhel9.0/x86_64/dvd/
        gpgcheck: true
        gpgkey: http://content.example.com/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: true

    - name: Add appstream-internal repo
      ansible.builtin.yum_repository:
        name: appstream-internal
        description: App Description
        baseurl: http://content/rhel9.0/x86_64/dvd/AppStream
        gpgcheck: true
        gpgkey: http://content.example.com/rhel9.0/x86_64/dvd/RPM-GPG-KEY-redhat-release
        enabled: true
```

---

## 3. Galaxy roles via `requirements.yml`

```bash
mkdir -p /home/student/ansible/roles
```

**`/home/student/ansible/roles/requirements.yml`**
```yaml
---
roles:
  - name: balancer
    src: http://content.example.com/Rhce/balancer.tgz
  - name: phpinfo
    src: http://content.example.com/Rhce/phpinfo.tgz
```

```bash
ansible-galaxy role install -r /home/student/ansible/roles/requirements.yml \
  -p /home/student/ansible/roles
```

---

## 4. Offline `apache` role

```bash
cd /home/student/ansible/roles
ansible-galaxy role init apache
```

**`roles/apache/tasks/main.yml`**
```yaml
---
- name: Install httpd
  ansible.builtin.dnf:
    name: httpd
    state: present

- name: Deploy index.html from template
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html

- name: Start and enable httpd
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

**`roles/apache/templates/index.html.j2`**
```
Welcome to {{ ansible_fqdn }} on {{ ansible_default_ipv4.address }}
```

**`apache_role.yml`**
```yaml
---
- name: Deploy apache role
  hosts: webservers
  become: true
  roles:
    - apache
```

---

## 5. `roles.yml` — balancer + phpinfo

```yaml
---
- name: Configure load balancer
  hosts: balancers
  become: true
  roles:
    - balancer

- name: Configure PHP info page
  hosts: webservers
  become: true
  roles:
    - phpinfo
```

Run with:
```bash
ansible-playbook roles.yml
```

---

## 6. Install a Content Collection from a tarball

```bash
ansible-galaxy collection install \
  http://rhls.lab.example.com/materials/<collection-file>.tar.gz \
  -p /home/student/ansible/mycollection
```
(Replace `<collection-file>` with whatever the exam's URL actually points to.)

---

## 7. `packages.yml` — multi-group package installs

```yaml
---
- name: Install php and mariadb-server on dev and test
  hosts: dev,test
  become: true
  tasks:
    - name: Install packages
      ansible.builtin.dnf:
        name:
          - php
          - mariadb-server
        state: present

- name: Install RPM Development Tools on prod
  hosts: prod
  become: true
  tasks:
    - name: Install group package
      ansible.builtin.dnf:
        name: "@RPM Development Tools"
        state: present

- name: Update all packages on dev
  hosts: dev
  become: true
  tasks:
    - name: Update all packages
      ansible.builtin.dnf:
        name: "*"
        state: latest
```

---

## 8. `webcontent.yml`

```yaml
---
- name: Configure dev web content
  hosts: dev
  become: true
  tasks:
    - name: Ensure develops group exists
      ansible.builtin.group:
        name: develops
        state: present

    - name: Create /devweb directory
      ansible.builtin.file:
        path: /devweb
        state: directory
        group: develops
        mode: '2775'          # rwxrwsr-x -> user=rwx group=rwx+setgid others=r-x
        # task literally says user=TWX -> rwx, group=rwx, others=rx, plus setgid
        # mode '2775' covers rwxrwxr-x + setgid bit

    - name: Set SELinux context type on /devweb
      community.general.sefcontext:
        target: '/devweb(/.*)?'
        setype: httpd_sys_content_t
        state: present

    - name: Apply SELinux context now
      ansible.builtin.command: restorecon -Rv /devweb

    - name: Create index.html
      ansible.builtin.copy:
        content: "Developement"
        dest: /devweb/index.html

    - name: Link /devweb under docroot
      ansible.builtin.file:
        src: /devweb
        dest: /var/www/html/devweb
        state: link
```

> Note: the task literally says permission `user=TWX` — read as `rwx` for owner
> (the T is presumably a typo for the sticky/special bit note). `mode: '2775'`
> gives owner rwx, group rwx, others r-x, and applies setgid (the "group special
> permission").

---

## 9. `hwreport.yml`

```yaml
---
- name: Generate hardware report
  hosts: all
  become: true
  tasks:
    - name: Download empty hwreport template
      ansible.builtin.get_url:
        url: http://content.example.com/Rhce/hwreport.empty
        dest: /root/hwreport.txt
        mode: '0644'

    - name: Gather facts
      ansible.builtin.setup:

    - name: Populate hwreport.txt
      ansible.builtin.copy:
        dest: /root/hwreport.txt
        content: |
          #hwreport
          HOSTNAME={{ ansible_hostname | default('NONE') }}
          MEMORY={{ ansible_memtotal_mb | default('NONE') }}
          BIOS={{ ansible_bios_version | default('NONE') }}
          CPU={{ ansible_processor[2] | default('NONE') }}
          DISK_SIZE_VDA={{ ansible_devices.vda.size | default('NONE') }}
          DISK_SIZE_VDB={{ ansible_devices.vdb.size | default('NONE') }}
```

---

## 10. `issue.yml`

```yaml
---
- name: Set /etc/issue per environment
  hosts: all
  become: true
  tasks:
    - name: Set issue for dev
      ansible.builtin.copy:
        content: "Developement"
        dest: /etc/issue
      when: "'dev' in group_names"

    - name: Set issue for test
      ansible.builtin.copy:
        content: "Test"
        dest: /etc/issue
      when: "'test' in group_names"

    - name: Set issue for prod
      ansible.builtin.copy:
        content: "Production"
        dest: /etc/issue
      when: "'prod' in group_names"
```

---

## 11. `hosts.yml` (uses `myhosts.j2`)

```bash
mkdir -p /home/student/ansible/templates
curl -o /home/student/ansible/templates/myhosts.j2 \
  http://content.example.com/Rhce/myhosts.j2
```

Append the loop to the downloaded template so it ends like:

**`templates/myhosts.j2`**
```
127.0.0.1 localhost.localdomain localhost
192.168.0.1 localhost.localdomain localhost
{% for host in groups['all'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }} {{ hostvars[host]['ansible_fqdn'] }} {{ hostvars[host]['ansible_hostname'] }}
{% endfor %}
```

**`hosts.yml`**
```yaml
---
- name: Build /etc/myhosts
  hosts: dev
  become: true
  tasks:
    - name: Gather facts from all hosts
      ansible.builtin.setup:
      delegate_to: "{{ item }}"
      delegate_facts: true
      loop: "{{ groups['all'] }}"

    - name: Deploy myhosts file
      ansible.builtin.template:
        src: myhosts.j2
        dest: /etc/myhosts
        mode: '0644'
```

---

## 12. `vault.yml`

```bash
echo 'P@sswOrd' > /home/student/ansible/secret.txt
chmod 600 /home/student/ansible/secret.txt
```

```bash
ansible-vault create --vault-password-file=secret.txt vault.yml
```
Contents to type in the editor:
```yaml
pw_developer: lamdev
pw_manager: lammgr
```

---

## 13. `users.yml`

```bash
curl -o /home/student/ansible/user_list.yml \
  http://content.example.com/Rhce/user_list.yml
```

```yaml
---
- name: Create developer accounts
  hosts: dev,test
  become: true
  vars_files:
    - user_list.yml
    - vault.yml
  tasks:
    - name: Ensure opsdev group exists
      ansible.builtin.group:
        name: opsdev
        state: present

    - name: Create developer users
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: opsdev
        append: true
        password: "{{ pw_developer | password_hash('sha512') }}"
      loop: "{{ users }}"
      when: item.job == 'developer'

- name: Create manager accounts
  hosts: prod
  become: true
  vars_files:
    - user_list.yml
    - vault.yml
  tasks:
    - name: Ensure opsmgr group exists
      ansible.builtin.group:
        name: opsmgr
        state: present

    - name: Create manager users
      ansible.builtin.user:
        name: "{{ item.name }}"
        groups: opsmgr
        append: true
        password: "{{ pw_manager | password_hash('sha512') }}"
      loop: "{{ users }}"
      when: item.job == 'manager'
```

Run with the vault password file:
```bash
ansible-playbook users.yml --vault-password-file=secret.txt
```

> Adjust `users` / `item.name` / `item.job` to whatever keys `user_list.yml`
> actually uses once you open it — Red Hat typically ships it as a list of
> dicts like `{ name: ..., job: ... }`.

---

## 14. Rekey `solaris.yml`

```bash
curl -o /home/student/ansible/solaris.yml \
  http://content.example.com/Rhce/solaris.yml

echo 'cisco' > old_pass.txt
echo 'redhat' > new_pass.txt

ansible-vault rekey \
  --vault-password-file=old_pass.txt \
  --new-vault-password-file=new_pass.txt \
  solaris.yml
```

---

## 15. `crontab.yml`

```yaml
---
- name: Create cron job for natasha
  hosts: all
  become: true
  tasks:
    - name: Ensure natasha exists
      ansible.builtin.user:
        name: natasha
        state: present

    - name: Add cron job
      ansible.builtin.cron:
        name: "EX294 progress logger"
        user: natasha
        minute: "*/2"
        job: 'logger "EX294 in progress"'
```

---

## 16. `lv.yml` — logical volume with error handling

```yaml
---
- name: Create logical volume with fallback size
  hosts: all
  become: true
  tasks:
    - name: Check research volume group exists
      ansible.builtin.command: vgs research
      register: vg_check
      failed_when: false
      changed_when: false

    - name: Fail cleanly if volume group missing
      ansible.builtin.debug:
        msg: "volume group does not exist"
      when: vg_check.rc != 0

    - name: Attempt to create 1500 MiB logical volume
      block:
        - name: Create data LV at 1500 MiB
          community.general.lvol:
            vg: research
            lv: data
            size: 1500m
      rescue:
        - name: Report size failure
          ansible.builtin.debug:
            msg: "Could not create logical volume of that size"

        - name: Create data LV at 800 MiB instead
          community.general.lvol:
            vg: research
            lv: data
            size: 800m
      when: vg_check.rc == 0

    - name: Format LV with ext4 (no mount)
      community.general.filesystem:
        fstype: ext4
        dev: /dev/research/data
      when: vg_check.rc == 0
```

---

## 18. `selinux.yml`

```yaml
---
- name: Enforce SELinux on all nodes
  hosts: all
  become: true
  tasks:
    - name: Set SELinux to enforcing
      ansible.posix.selinux:
        policy: targeted
        state: enforcing
```

Requires the `ansible.posix` collection:
```bash
ansible-galaxy collection install ansible.posix -p /home/student/ansible/mycollection
```

---

## Quick run reference

```bash
cd /home/student/ansible

ansible-playbook yum_repo.yml
ansible-playbook apache_role.yml
ansible-playbook roles.yml
ansible-playbook packages.yml
ansible-playbook webcontent.yml
ansible-playbook hwreport.yml
ansible-playbook issue.yml
ansible-playbook hosts.yml
ansible-playbook users.yml --vault-password-file=secret.txt
ansible-playbook crontab.yml
ansible-playbook lv.yml
ansible-playbook selinux.yml
```
