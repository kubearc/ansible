# Ansible Roles — Practice Tasks

Complete each task in order. Try solving it yourself first, then expand the solution to check your work.

**Prerequisites:** You should already be comfortable with playbooks, tasks, variables, and modules like `dnf`, `copy`, `template`, and `service`.

---

## Task 1: Scaffold Your First Role

Create a role named `apache_basic` using the correct Ansible tool, and list the generated directory structure.

<details>
<summary>Solution</summary>

```bash
ansible-galaxy init roles/apache_basic
tree roles/apache_basic
```

Expected structure:
```
roles/apache_basic/
├── tasks/main.yml
├── handlers/main.yml
├── templates/
├── files/
├── vars/main.yml
├── defaults/main.yml
├── meta/main.yml
└── tests/
```

</details>

---

## Task 2: Install and Start a Service

Inside `apache_basic`, write the tasks to:
1. Install `httpd`
2. Ensure the service is started and enabled at boot

<details>
<summary>Solution</summary>

```yaml
# roles/apache_basic/tasks/main.yml
- name: Install httpd
  ansible.builtin.dnf:
    name: httpd
    state: present

- name: Ensure httpd is running and enabled
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

</details>

---

## Task 3: Deploy a Templated Config with a Handler

Add a task that deploys a Jinja2 template to `/etc/httpd/conf.d/vhost.conf`, and make sure Apache restarts **only when the config actually changes**.

<details>
<summary>Solution</summary>

```yaml
# roles/apache_basic/tasks/main.yml (append)
- name: Deploy vhost config
  ansible.builtin.template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/vhost.conf
  notify: Restart httpd
```

```yaml
# roles/apache_basic/handlers/main.yml
- name: Restart httpd
  ansible.builtin.service:
    name: httpd
    state: restarted
```

```jinja2
{# roles/apache_basic/templates/vhost.conf.j2 #}
<VirtualHost *:{{ http_port }}>
    ServerName {{ server_name }}
    DocumentRoot /var/www/html
</VirtualHost>
```

**Check yourself:** Run the playbook twice. On the second run, the handler should NOT fire (0 changes, service not restarted).

</details>

---

## Task 4: Overridable vs Fixed Variables

You need two variables:
- `http_port` — should default to `80` but be easy for anyone using this role to override
- `apache_package_name` — should default to `httpd` and should generally NOT be overridden casually

Decide where each belongs, and write the files.

<details>
<summary>Solution</summary>

```yaml
# roles/apache_basic/defaults/main.yml
http_port: 80
server_name: "localhost"
```

```yaml
# roles/apache_basic/vars/main.yml
apache_package_name: httpd
```

**Why:** `defaults/` is the lowest-precedence location, meant to be freely overridden by whoever uses the role. `vars/` sits much higher in precedence, so it resists accidental overrides — appropriate for values the role author considers fixed.

</details>

---

## Task 5: Prove the Precedence

Run your role two ways and observe the difference:

```bash
ansible-playbook site.yml -e "http_port=8080"
ansible-playbook site.yml -e "apache_package_name=nginx"
```

What happens in each case? Why?

<details>
<summary>Solution</summary>

- `-e "http_port=8080"` → **works**, port changes to 8080, because `defaults/` is low precedence and `-e` (extra-vars) overrides it.
- `-e "apache_package_name=nginx"` → **does NOT work**, `httpd` still gets installed, because `vars/` sits at a higher precedence than `-e` extra-vars.

This is the single most-confused concept for beginners — if you got both right, you understand role variable precedence.

</details>

---

## Task 6: `import_role` vs `include_role`

Write a playbook task that applies `apache_basic` **only when** a variable `install_webserver` is `true`. Which role-inclusion method must you use, and why?

<details>
<summary>Solution</summary>

```yaml
- hosts: webservers
  tasks:
    - name: Apply apache_basic conditionally
      ansible.builtin.include_role:
        name: apache_basic
      when: install_webserver | bool
```

**Why `include_role` and not `import_role`:** `import_role` is static and resolved at playbook parse time, before any `when` conditions are evaluated at runtime. `include_role` is dynamic and evaluates conditionals like `when` at execution time, so it correctly skips the role when the condition is false.

</details>

---

## Task 7: Full Build — nginx_basic (Capstone)

Build a complete role named `nginx_basic` from scratch that:
1. Installs `nginx`
2. Deploys a templated `index.html` using a variable `site_title` (overridable, sensible default)
3. Ensures the service is enabled and started
4. Restarts nginx only when the config changes
5. Uses correct `vars/` vs `defaults/` placement throughout

<details>
<summary>Solution</summary>

```
roles/nginx_basic/
├── tasks/main.yml
├── handlers/main.yml
├── templates/index.html.j2
└── defaults/main.yml
```

```yaml
# defaults/main.yml
site_title: "Welcome to Kubearc Academy"
```

```yaml
# tasks/main.yml
- name: Install nginx
  ansible.builtin.dnf:
    name: nginx
    state: present

- name: Deploy index page
  ansible.builtin.template:
    src: index.html.j2
    dest: /usr/share/nginx/html/index.html
  notify: Restart nginx

- name: Ensure nginx running and enabled
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

```yaml
# handlers/main.yml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

```jinja2
{# templates/index.html.j2 #}
<h1>{{ site_title }}</h1>
```

</details>

---

## Bonus Challenge: Spot the Bug

A student wrote this role and it silently does nothing when they run it. Find the mistake.

```
roles/mysql_basic/
├── task/main.yml
├── handlers/main.yml
```

<details>
<summary>Solution</summary>

The folder is named `task/` instead of `tasks/` (missing the "s"). Ansible only auto-loads folders matching its exact naming convention — a misspelled folder name fails silently with no error, and the role simply does nothing. Always double-check folder names against the standard structure: `tasks/`, `handlers/`, `templates/`, `files/`, `vars/`, `defaults/`, `meta/`.

</details>
