# Ansible Loops — Practice

Hands-on tasks and Q&A covering `loop`, legacy `with_*` looping, looping over
dictionaries, nested loops, `loop_control`, and retry loops (`until`).

Work through the tasks first, then use the Q&A section to check your
understanding. Answers are hidden in collapsible blocks — try to answer
before expanding.

---

## Setup

All tasks assume a simple playbook skeleton:

```yaml
---
- name: Loops practice
  hosts: localhost
  gather_facts: false
  vars:
    packages:
      - httpd
      - vim
      - git
    users:
      alice:
        role: admin
      bob:
        role: developer
      carol:
        role: viewer
  tasks:
    # add each task below here, one at a time
```

Run each task with:

```bash
ansible-playbook loops.yml
```

---

## Part 1 — Hands-On Tasks

### Task 1: Basic `loop` over a list

Write a task that creates three files: `/tmp/file1.txt`, `/tmp/file2.txt`,
`/tmp/file3.txt`, using `loop` and the `file` module. Do not hardcode three
separate tasks.

<details>
<summary>Show solution</summary>

```yaml
- name: Create multiple files
  ansible.builtin.file:
    path: "/tmp/{{ item }}"
    state: touch
  loop:
    - file1.txt
    - file2.txt
    - file3.txt
```

</details>

---

### Task 2: Loop over a variable list

Using the `packages` variable defined in Setup, write a task that installs
all three packages with `dnf`, looping over the variable rather than an
inline list.

<details>
<summary>Show solution</summary>

```yaml
- name: Install packages
  ansible.builtin.dnf:
    name: "{{ item }}"
    state: present
  loop: "{{ packages }}"
```

</details>

---

### Task 3: Loop over a dictionary

Using the `users` variable, write a task that prints a debug message for
each user in the form:

```
alice has role admin
bob has role developer
carol has role viewer
```

<details>
<summary>Show solution</summary>

```yaml
- name: Show user roles
  ansible.builtin.debug:
    msg: "{{ item.key }} has role {{ item.value.role }}"
  loop: "{{ users | dict2items }}"
```

`dict2items` converts the dictionary into a list of `{key, value}` pairs so
`loop` can iterate over it.

</details>

---

### Task 4: Nested loop with `loop_control`

Write a task that, for every package in `packages`, prints a debug message
for every user in `users`, in the form:

```
Installing httpd for alice
Installing httpd for bob
...
```

Use `loop_control.loop_var` to avoid the `item` naming clash in the nested
loop.

<details>
<summary>Show solution</summary>

```yaml
- name: Nested loop over packages and users
  ansible.builtin.debug:
    msg: "Installing {{ pkg }} for {{ user_item.key }}"
  loop: "{{ packages }}"
  loop_control:
    loop_var: pkg
  vars:
    inner_loop: "{{ users | dict2items }}"
```

This only sets up the outer loop. A full nested loop needs `include_tasks`
with the inner loop in a separate file — plain nested `loop:` inside
`loop:` isn't supported directly. Minimal working pattern:

```yaml
- name: Outer loop over packages
  ansible.builtin.include_tasks: inner.yml
  loop: "{{ packages }}"
  loop_control:
    loop_var: pkg
```

`inner.yml`:

```yaml
- name: Inner loop over users
  ansible.builtin.debug:
    msg: "Installing {{ pkg }} for {{ item.key }}"
  loop: "{{ users | dict2items }}"
```

</details>

---

### Task 5: `index_var`

Write a task that loops over `packages` and prints:

```
0: httpd
1: vim
2: git
```

<details>
<summary>Show solution</summary>

```yaml
- name: Show package index
  ansible.builtin.debug:
    msg: "{{ idx }}: {{ item }}"
  loop: "{{ packages }}"
  loop_control:
    index_var: idx
```

</details>

---

### Task 6: `until` retry loop

Write a task that checks whether `/tmp/ready.txt` exists, retrying up to 5
times with a 3 second delay between attempts, and registers the result.
Use `command: stat /tmp/ready.txt` (ignore errors) with `until` checking
`result.rc == 0`.

<details>
<summary>Show solution</summary>

```yaml
- name: Wait for ready file
  ansible.builtin.command: stat /tmp/ready.txt
  register: result
  until: result.rc == 0
  retries: 5
  delay: 3
  ignore_errors: true
```

</details>

---

### Task 7: Loop with a condition

Using `users`, write a task that only prints a debug message for users
whose role is `admin` or `developer` (skip `viewer`).

<details>
<summary>Show solution</summary>

```yaml
- name: Show privileged users
  ansible.builtin.debug:
    msg: "{{ item.key }} ({{ item.value.role }})"
  loop: "{{ users | dict2items }}"
  when: item.value.role in ['admin', 'developer']
```

</details>

---

### Task 8: `with_items` vs `loop`

Rewrite this legacy task to use `loop` instead of `with_items`, and explain
in one sentence why `loop` is generally preferred today.

```yaml
- name: Old style
  ansible.builtin.debug:
    msg: "{{ item }}"
  with_items:
    - a
    - b
    - c
```

<details>
<summary>Show solution</summary>

```yaml
- name: New style
  ansible.builtin.debug:
    msg: "{{ item }}"
  loop:
    - a
    - b
    - c
```

`loop` is the modern, explicit replacement for the `with_*` lookup-plugin
family — it's simpler to read, doesn't rely on implicit lookup plugin
behavior, and is the form Ansible's own docs now recommend for straightforward
list iteration.

</details>

---

### Task 9: Generate `/etc/hosts` with a Jinja2 `{% for %}` loop

This is a different kind of loop from Tasks 1–8: instead of a `loop:`
keyword on a module, the loop lives *inside a Jinja2 template* and is
rendered by the `template` module. This is the classic RHCE/EX407-style
"generate a hosts file" exercise.

Build a template `hosts.j2` that iterates over every host in inventory and
appends one line per host to `/etc/hosts`, in the form:

```
<ipv4 address> <fqdn> <short hostname>
```

Requirements:

- Iterate with `{% for host in groups['all'] %}` / `{% endfor %}`.
- Pull facts for each host via `hostvars[host]`.
- Do **not** touch the existing content of `/etc/hosts` — only append.

<details>
<summary>Show solution</summary>

**`templates/hosts.j2`**

```jinja2
{% for host in groups['all'] %}
{{ hostvars[host].ansible_default_ipv4.address }} {{ hostvars[host].ansible_fqdn }} {{ hostvars[host].ansible_hostname }}
{% endfor %}
```

**Playbook task**

```yaml
- name: Gather facts from all hosts so hostvars is populated
  hosts: all
  tasks: []

- name: Generate hosts file entries
  hosts: all
  become: true
  tasks:
    - name: Append generated host entries to /etc/hosts
      ansible.builtin.blockinfile:
        path: /etc/hosts
        marker: "# {mark} ANSIBLE MANAGED HOSTS BLOCK"
        block: "{{ lookup('template', 'hosts.j2') }}"
```

Notes:

- `hostvars[host]` only has facts for hosts Ansible has already gathered
  facts from — every host in the play (or a prior play/task) needs
  `gather_facts: true` (the default) so `ansible_default_ipv4`,
  `ansible_fqdn`, and `ansible_hostname` are populated for all of them.
- Using `blockinfile` with a marker (instead of overwriting the file with
  `template`) is what satisfies "do not modify the existing entries" —
  it inserts a clearly delimited block and leaves the rest of
  `/etc/hosts` untouched. Re-running the playbook updates only that block.
- If you truly want to *replace* the whole file (e.g. in a lab/EX407
  context where the file is meant to be fully regenerated), use the
  `template` module directly against `/etc/hosts` instead of
  `blockinfile` — but then the "don't modify existing entries" note in
  the original template becomes your reminder not to *delete* lines
  above your `{% for %}` loop when you edit `hosts.j2` itself.

</details>

---

## Part 2 — Q&A

<details>
<summary>Q1. What variable holds the current item in a standard <code>loop</code>?</summary>

`item`, unless overridden with `loop_control.loop_var`.

</details>

<details>
<summary>Q2. Why can't you nest two <code>loop:</code> keywords directly on the same task?</summary>

A single task only has one `item` context. Ansible has no native syntax for
a task to iterate two independent loops at once — nesting requires calling
out to a second task (typically via `include_tasks`) so each loop level runs
in its own task with its own `loop_var`.

</details>

<details>
<summary>Q3. What does <code>dict2items</code> do, and what's the reverse filter?</summary>

`dict2items` converts a dictionary into a list of `{key, value}` mappings so
it can be used with `loop`. The reverse filter is `items2dict`, which turns
a list of `{key, value}` mappings back into a dictionary.

</details>

<details>
<summary>Q4. In an <code>until</code> loop, what do <code>retries</code> and <code>delay</code> control?</summary>

`retries` sets the maximum number of attempts, and `delay` sets the number
of seconds to wait between each attempt. The loop stops as soon as the
`until` condition evaluates true, or after `retries` attempts are exhausted.

</details>

<details>
<summary>Q5. What does <code>loop_control.label</code> do, and when would you use it?</summary>

It controls what gets printed in the task output for each iteration instead
of dumping the entire `item` object. It's useful when looping over large
dictionaries or complex structures, where printing the full item would
clutter the output — e.g. `label: "{{ item.key }}"`.

</details>

<details>
<summary>Q6. What's the difference between <code>with_items</code> and <code>with_list</code>?</summary>

`with_items` automatically flattens one level of nested lists before
iterating, while `with_list` (and modern `loop`) does not flatten — it
iterates over the list exactly as given.

</details>

<details>
<summary>Q7. Can you use <code>register</code> inside a loop? What does the result look like?</summary>

Yes. The registered variable becomes a dictionary with a `results` key
containing a list — one entry per loop iteration, each holding that
iteration's module output plus the corresponding `item`.

</details>

<details>
<summary>Q8. How do you loop over a list of dictionaries and access nested keys?</summary>

Access fields with normal dot or bracket notation on `item`, e.g.
`item.name` or `item['name']`, since each `item` is itself a dictionary for
that iteration.

</details>

---

## Part 3 — Challenge

Write a single playbook (no `include_tasks`) that:

1. Loops over `users` (from Setup).
2. For `admin` and `developer` roles only, prints `"{{ key }} can deploy"`.
3. Uses `loop_control.label` so the task output shows only the username,
   not the full dictionary.
4. Uses `index_var` to also show the position of each user in the loop.

<details>
<summary>Show solution</summary>

```yaml
- name: Show deploy-eligible users
  ansible.builtin.debug:
    msg: "{{ item.key }} can deploy (position {{ idx }})"
  loop: "{{ users | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
    index_var: idx
  when: item.value.role in ['admin', 'developer']
```

</details>

---

*Part of `kubearc/linux-practice` — Ansible topic series.*
