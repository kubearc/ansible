# Ansible Variables — Practice Q&A

**Topics:** `vars` · `vars_files` · `group_vars` / `host_vars` · `register` · `facts` · `default()` · `vars_prompt` · precedence

Read each question, write the playbook yourself, then expand the answer to check.

---

## Playbook Variables (`vars:`)

**Q1. Define a variable `pkg_name` set to `httpd` and print `"Installing httpd"` using the variable.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Print package name
  hosts: all
  vars:
    pkg_name: httpd
  tasks:
    - name: Show install message
      debug:
        msg: "Installing {{ pkg_name }}"
```
</details>

---

**Q2. Define `pkg_name: httpd` and `pkg_version: "2.4"`, and print both in one message.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Print package name and version
  hosts: all
  vars:
    pkg_name: httpd
    pkg_version: "2.4"
  tasks:
    - name: Show install message
      debug:
        msg: "Installing {{ pkg_name }} version {{ pkg_version }}"
```
</details>

---

**Q3. Define a list variable `packages` containing `httpd`, `vim`, `git` and print each one with a loop.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Print each package
  hosts: all
  vars:
    packages:
      - httpd
      - vim
      - git
  tasks:
    - name: Show package name
      debug:
        msg: "Package: {{ item }}"
      loop: "{{ packages }}"
```
</details>

---

**Q4. Define a dictionary variable `user_info` with `name`, `uid`, and `shell`, and print all three in one message.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Print user info
  hosts: all
  vars:
    user_info:
      name: alice
      uid: 1050
      shell: /bin/bash
  tasks:
    - name: Show user info
      debug:
        msg: "User {{ user_info.name }} has uid {{ user_info.uid }} and shell {{ user_info.shell }}"
```
</details>

---

## External Variable Files (`vars_files:`)

**Q5. Load `vars/app_vars.yml` (containing `app_name`, `app_port`, `app_env`) and print `"myapp is running on port 8080 in production"`.**

<details>
<summary>Answer</summary>

`vars/app_vars.yml`
```yaml
app_name: myapp
app_port: 8080
app_env: production
```

Playbook:
```yaml
---
- name: Show app info
  hosts: all
  vars_files:
    - vars/app_vars.yml
  tasks:
    - name: Print app summary
      debug:
        msg: "{{ app_name }} is running on port {{ app_port }} in {{ app_env }}"
```
</details>

---

**Q6. Load both `vars/app_vars.yml` and `vars/db_vars.yml` (containing `db_name`, `db_user`) and print a combined message using variables from each file.**

<details>
<summary>Answer</summary>

`vars/db_vars.yml`
```yaml
db_name: appdb
db_user: appuser
```

Playbook:
```yaml
---
- name: Show app and db info
  hosts: all
  vars_files:
    - vars/app_vars.yml
    - vars/db_vars.yml
  tasks:
    - name: Print combined summary
      debug:
        msg: "{{ app_name }} connects to database {{ db_name }} as {{ db_user }}"
```
</details>

---

**Q7. Define `app_env` both in `vars_files` and directly under `vars:` in the playbook, with different values. Which one wins?**

<details>
<summary>Answer</summary>

```yaml
---
- name: Precedence check
  hosts: all
  vars_files:
    - vars/app_vars.yml   # app_env: production
  vars:
    app_env: staging       # this wins
  tasks:
    - name: Show winning value
      debug:
        msg: "app_env is {{ app_env }}"
```

> `vars_files` are loaded first, then `vars:` is applied on top — so the value
> defined directly under `vars:` in the playbook wins. Output: `app_env is staging`.
</details>

---

## Host and Group Variables

**Q8. Set `motd_message: "Welcome to node1"` in `host_vars/node1.yml` and print it for all hosts.**

<details>
<summary>Answer</summary>

`host_vars/node1.yml`
```yaml
motd_message: "Welcome to node1"
```

Playbook:
```yaml
---
- name: Show motd message
  hosts: all
  tasks:
    - name: Print motd
      debug:
        msg: "{{ motd_message | default('No message set for this host') }}"
```
</details>

---

**Q9. Set `http_port: 80` in `group_vars/web.yml` and `db_port: 5432` in `group_vars/db.yml`. Print the correct port for each host depending on its group.**

<details>
<summary>Answer</summary>

`group_vars/web.yml`
```yaml
http_port: 80
```

`group_vars/db.yml`
```yaml
db_port: 5432
```

Playbook:
```yaml
---
- name: Show service port
  hosts: all
  tasks:
    - name: Print port
      debug:
        msg: "Port in use: {{ http_port | default(db_port) | default('unknown') }}"
```
</details>

---

**Q10. Set `org_name: "KubeArc"` in `group_vars/all.yml` and confirm it's visible on both `node1` and `node2`.**

<details>
<summary>Answer</summary>

`group_vars/all.yml`
```yaml
org_name: "KubeArc"
```

Playbook:
```yaml
---
- name: Show org name
  hosts: all
  tasks:
    - name: Print org name
      debug:
        msg: "This host belongs to {{ org_name }}"
```
</details>

---

**Q11. Add a group variable `web_root: /var/www/html` for the `web` group and use it in a `file` task to create that directory.**

<details>
<summary>Answer</summary>

`group_vars/web.yml`
```yaml
web_root: /var/www/html
```

Playbook:
```yaml
---
- name: Create web root using group var
  hosts: web
  become: yes
  tasks:
    - name: Create web root directory
      file:
        path: "{{ web_root }}"
        state: directory
        mode: '0755'
```
</details>

---

## Registered Variables & Facts

**Q12. Run `date` on the managed node, register the result as `date_output`, and print `date_output.stdout`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Show current date
  hosts: all
  tasks:
    - name: Get date
      command: date
      register: date_output

    - name: Print date
      debug:
        msg: "{{ date_output.stdout }}"
```
</details>

---

**Q13. Register the output of `whoami`, and only run a follow-up task if the result equals `ansible`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Conditional based on whoami
  hosts: all
  tasks:
    - name: Get current user
      command: whoami
      register: whoami_result

    - name: Run only if user is ansible
      debug:
        msg: "Confirmed running as ansible user"
      when: whoami_result.stdout == "ansible"
```
</details>

---

**Q14. Print the managed node's `ansible_distribution`, `ansible_distribution_version`, and `ansible_default_ipv4.address` in one debug message.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Show host facts
  hosts: all
  tasks:
    - name: Print OS and IP info
      debug:
        msg: "{{ ansible_distribution }} {{ ansible_distribution_version }} - IP: {{ ansible_default_ipv4.address }}"
```
</details>

---

**Q15. Install `httpd` in check mode, register the result, and print whether `.changed` is true or false.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Check install status
  hosts: all
  become: yes
  tasks:
    - name: Simulate httpd install
      yum:
        name: httpd
        state: present
      check_mode: yes
      register: install_result

    - name: Show changed status
      debug:
        msg: "Change needed: {{ install_result.changed }}"
```
</details>

---

**Q16. Register the result of a `stat` task checking whether `/etc/httpd/conf/httpd.conf` exists, and print a message accordingly.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Check if config file exists
  hosts: all
  tasks:
    - name: Check httpd.conf
      stat:
        path: /etc/httpd/conf/httpd.conf
      register: conf_check

    - name: Print result
      debug:
        msg: "Config file exists: {{ conf_check.stat.exists }}"
```
</details>

---

## Variable Precedence

**Q17. Set `greeting: "Hello from group_vars"` in `group_vars/web.yml` and `greeting: "Hello from playbook vars"` under `vars:` in the playbook. Which one wins?**

<details>
<summary>Answer</summary>

```yaml
---
- name: Precedence test
  hosts: web
  vars:
    greeting: "Hello from playbook vars"
  tasks:
    - name: Print greeting
      debug:
        msg: "{{ greeting }}"
```

> Play `vars:` has higher precedence than `group_vars`, so the output is
> `Hello from playbook vars`.
</details>

---

**Q18. Using the playbook from Q17, override `greeting` from the command line with `-e`. What wins now?**

<details>
<summary>Answer</summary>

```bash
ansible-playbook task17.yml -e "greeting='Hello from CLI'"
```

> Extra vars (`-e`) always have the **highest** precedence in Ansible — they
> override `vars:`, `vars_files`, `group_vars`, `host_vars`, and even registered
> variables. Output: `Hello from CLI`.
</details>

---

**Q19. Set `env_name: dev` in `group_vars/all.yml` and `env_name: staging` in `host_vars/node1.yml`. What value does `node1` see, and why?**

<details>
<summary>Answer</summary>

```yaml
---
- name: Show env name
  hosts: all
  tasks:
    - name: Print environment
      debug:
        msg: "Environment: {{ env_name }}"
```

> `node1` sees `staging`. Host-level variables (`host_vars`) are more specific
> than group-level variables (`group_vars`), and Ansible's precedence order
> always favors the more specific scope. Any other host without a `host_vars`
> entry still sees `dev` from `group_vars/all.yml`.
</details>

---

## Defaults & Conditionals with Variables

**Q20. Use the `default()` filter so that if `custom_port` isn't defined, it falls back to `8080`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Show port with fallback
  hosts: all
  tasks:
    - name: Print port
      debug:
        msg: "Using port {{ custom_port | default(8080) }}"
```
</details>

---

**Q21. Use `vars_prompt` to interactively ask for `deploy_env` at runtime, then print `"Deploying to <value>"`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Interactive deploy
  hosts: all
  vars_prompt:
    - name: deploy_env
      prompt: "Which environment are you deploying to?"
      private: no
  tasks:
    - name: Print deployment target
      debug:
        msg: "Deploying to {{ deploy_env }}"
```
</details>

---

**Q22. Only run a task if a variable `enable_firewall` is defined and set to `true`; skip cleanly if it's undefined.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Conditional firewall setup
  hosts: all
  become: yes
  tasks:
    - name: Enable firewalld
      service:
        name: firewalld
        state: started
        enabled: yes
      when: enable_firewall | default(false) | bool
```
</details>

---

## Mixed Questions

**Q23. Create a user whose name comes from a variable `new_user: deployer`, with shell `/bin/bash`, and print a debug confirmation after creation.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Create user from variable
  hosts: all
  become: yes
  vars:
    new_user: deployer
  tasks:
    - name: Create user
      user:
        name: "{{ new_user }}"
        shell: /bin/bash
        state: present

    - name: Confirm creation
      debug:
        msg: "User {{ new_user }} has been created"
```
</details>

---

**Q24. Load `app_port` from `vars_files`, then use it in a `lineinfile` task to set `Listen <app_port>` in `/etc/httpd/conf/httpd.conf`.**

<details>
<summary>Answer</summary>

```yaml
---
- name: Configure httpd listen port
  hosts: all
  become: yes
  vars_files:
    - vars/app_vars.yml
  tasks:
    - name: Set Listen directive
      lineinfile:
        path: /etc/httpd/conf/httpd.conf
        regexp: '^Listen'
        line: "Listen {{ app_port }}"
        state: present
```
</details>

---

**Q25. Copy `files/index.html` to a destination path built from a variable `web_root` (default `/var/www/html` if undefined).**

<details>
<summary>Answer</summary>

```yaml
---
- name: Deploy index page
  hosts: all
  become: yes
  tasks:
    - name: Copy index.html
      copy:
        src: files/index.html
        dest: "{{ web_root | default('/var/www/html') }}/index.html"
        owner: root
        mode: '0644'
```
</details>

---

## Capstone Playbook

**Q26. Full variables workflow** — Write one playbook that does all of the following:

1. Loads `app_name` and `app_port` from a `vars_files`
2. Overrides `app_env` using a `group_vars/web.yml` value
3. Registers the output of checking whether `/etc/nginx/nginx.conf` exists (`stat`)
4. Uses `default()` to safely handle an optional variable `max_clients` (fallback `200`)
5. Prints a final summary debug message combining all of the above

<details>
<summary>Answer</summary>

`vars/app_vars.yml`
```yaml
app_name: myapp
app_port: 8080
```

`group_vars/web.yml`
```yaml
app_env: production
```

Playbook:
```yaml
---
- name: Full variables workflow
  hosts: web
  become: yes
  vars_files:
    - vars/app_vars.yml
  tasks:
    - name: Check nginx config presence
      stat:
        path: /etc/nginx/nginx.conf
      register: nginx_conf_check

    - name: Print full summary
      debug:
        msg: >
          {{ app_name }} runs on port {{ app_port }} in {{ app_env }} environment.
          Nginx config present: {{ nginx_conf_check.stat.exists }}.
          Max clients: {{ max_clients | default(200) }}.
```
</details>

---

*26 questions — variables, vars_files, group_vars/host_vars, register, facts, precedence, defaults*
