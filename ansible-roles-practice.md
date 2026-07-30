# Ansible Roles — Practice Tasks (Basic → Execution)

Complete each task in order. Try solving it yourself first, then expand the solution to check your work.

**Prerequisites:** Comfortable with playbooks, tasks, variables, and modules like `dnf`, `copy`, `template`, `service`.

---

## Part A — Building the Role

### Task 1: Scaffold Your First Role

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

### Task 2: Install and Start a Service

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

### Task 3: Deploy a Templated Config with a Handler

Add a task that deploys a Jinja2 template to `/etc/httpd/conf.d/vhost.conf`, and make sure Apache restarts **only when the config actually changes**.

<details>
<summary>Solution</summary>

```yaml
# roles/apache_basic/tasks/main.yml (append)
- name: Deploy vhost config
  ansible.builtin.template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/vhost.conf
  tags: config
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

**Check yourself:** Run the playbook twice (see Part B). On the second run, the handler should NOT fire.

</details>

---

### Task 4: Overridable vs Fixed Variables

You need two variables:
- `http_port` — defaults to `80`, easy for users of the role to override
- `apache_package_name` — defaults to `httpd`, should NOT be casually overridden

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

**Why:** `defaults/` is the lowest-precedence location. `vars/` sits much higher in precedence, resisting accidental overrides.

</details>

---

## Part B — Executing the Role

### Task 5: Write the Playbook and Run It

Create `site.yml` that applies `apache_basic` to hosts in group `webservers`, then run it.

<details>
<summary>Solution</summary>

```yaml
# site.yml
- hosts: webservers
  roles:
    - apache_basic
```

```bash
ansible-playbook site.yml
```

**Check yourself:** Run the same command a second time immediately after. The recap should show `changed=0` for the tasks that already applied correctly — this confirms idempotency.

</details>

---

### Task 6: Override a Variable at Runtime

Run the playbook so `http_port` becomes `8080` **without editing any files**.

<details>
<summary>Solution</summary>

```bash
ansible-playbook site.yml -e "http_port=8080"
```

Works because `-e` (extra-vars) has higher precedence than `defaults/`.

</details>

---

### Task 7: Prove the Precedence Rule

Now try overriding `apache_package_name` the same way:

```bash
ansible-playbook site.yml -e "apache_package_name=nginx"
```

What happens, and why?

<details>
<summary>Solution</summary>

`httpd` still gets installed — `nginx` is ignored. `vars/main.yml` sits at a higher precedence than `-e` extra-vars, so it cannot be overridden this way. This is the opposite of `http_port`, which lives in `defaults/` (lowest precedence) and *can* be overridden by `-e`.

</details>

---

### Task 8: Run Only Part of the Role Using Tags

Using the `tags: config` you added in Task 3, run **only** the config-deployment task — skip the install and service tasks entirely.

<details>
<summary>Solution</summary>

```bash
ansible-playbook site.yml --tags config
```

Useful for quickly testing a template change without re-running the full role.

</details>

---

### Task 9: Limit Execution to One Host

Your inventory has `node1` and `node2` under `webservers`. Run the role against `node1` only.

<details>
<summary>Solution</summary>

```bash
ansible-playbook site.yml --limit node1
```

</details>

---

### Task 10: Dry-Run Before Applying

Before actually changing anything, show what the role *would* change on `node2`.

<details>
<summary>Solution</summary>

```bash
ansible-playbook site.yml --limit node2 --check --diff
```

`--check` simulates the run without making changes; `--diff` shows the before/after content for any file that would change (like the template). Good habit: always dry-run against production-like hosts before a real apply.

</details>

---

### Task 11: Conditional Role Execution

Modify the playbook so `apache_basic` only applies when a variable `install_webserver` is `true`. Which role-inclusion method must you use?

<details>
<summary>Solution</summary>

```yaml
# site.yml
- hosts: webservers
  tasks:
    - name: Apply apache_basic conditionally
      ansible.builtin.include_role:
        name: apache_basic
      when: install_webserver | bool
```

```bash
ansible-playbook site.yml -e "install_webserver=true"
```

`include_role` is required (not `import_role`) because it's evaluated dynamically at runtime, so `when` conditions are respected. `import_role` is resolved at parse time, before conditionals are checked.

</details>

---

## Part C — Capstone

### Task 12: Full Build and Execution — nginx_basic

Build a complete role named `nginx_basic` that:
1. Installs `nginx`
2. Deploys a templated `index.html` using a variable `site_title` (overridable, sensible default)
3. Ensures the service is enabled and started
4. Restarts nginx only when the config changes

Then:
- Run it normally
- Run it again to confirm idempotency (0 changes)
- Run it a third time overriding `site_title` via `-e`

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

```yaml
# site.yml
- hosts: webservers
  roles:
    - nginx_basic
```

```bash
ansible-playbook site.yml
ansible-playbook site.yml                              # confirm changed=0
ansible-playbook site.yml -e "site_title=Kubearc Labs"  # confirm override + restart fires
```

</details>

---

## Bonus Challenge: Spot the Bug

A student wrote this role and running `ansible-playbook site.yml` does nothing at all.

```
roles/mysql_basic/
├── task/main.yml
├── handlers/main.yml
```

```bash
ansible-galaxy run mysql_basic
```

<details>
<summary>Solution</summary>

Two bugs:

1. **Folder typo:** `task/` should be `tasks/`. Ansible auto-loads folders only by exact name — a misspelled folder fails silently with no error.
2. **Wrong execution command:** `ansible-galaxy run` doesn't exist. `ansible-galaxy` is only for scaffolding (`init`) and installing roles/collections — never for execution. Roles are always run through `ansible-playbook`, referencing the role in a playbook's `roles:` or `include_role`/`import_role`.

</details>
