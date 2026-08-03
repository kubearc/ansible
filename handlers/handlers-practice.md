# Ansible Handlers, `when`, and `block`/`rescue`/`always` — Practice

Hands-on tasks and Q&A covering handlers and notifications, conditional
execution with `when`, and error handling with `block` / `rescue` /
`always`.

Work through the tasks first, then use the Q&A section to check your
understanding. Answers are hidden in collapsible blocks — try to answer
before expanding.

---

## Setup

Assume a playbook skeleton targeting a webserver:

```yaml
---
- name: Handlers and blocks practice
  hosts: web
  become: true
  vars:
    app_env: production
  tasks:
    # add each task below here, one at a time
  handlers:
    # add handlers here
```

Run with:

```bash
ansible-playbook site.yml
```

---

## Part 1 — Hands-On Tasks

### Task 1: Basic handler with `notify`

Write a task that copies `httpd.conf` to `/etc/httpd/conf/httpd.conf`, and
a handler named `restart httpd` that restarts the `httpd` service. The
handler should only fire if the file actually changes.

<details>
<summary>Show solution</summary>

```yaml
tasks:
  - name: Deploy httpd config
    ansible.builtin.copy:
      src: httpd.conf
      dest: /etc/httpd/conf/httpd.conf
    notify: restart httpd

handlers:
  - name: restart httpd
    ansible.builtin.service:
      name: httpd
      state: restarted
```

Handlers only run when a task that notifies them reports `changed`. If the
file already matches, `copy` reports `ok` and the handler is skipped.

</details>

---

### Task 2: Multiple handlers, notified in sequence

Write a task that deploys a config file and notifies two handlers:
`validate config` (runs `httpd -t`) and `restart httpd`. Handlers should
run in the order listed under `handlers:`, not the order they're notified
in.

<details>
<summary>Show solution</summary>

```yaml
tasks:
  - name: Deploy httpd config
    ansible.builtin.copy:
      src: httpd.conf
      dest: /etc/httpd/conf/httpd.conf
    notify:
      - validate config
      - restart httpd

handlers:
  - name: validate config
    ansible.builtin.command: httpd -t

  - name: restart httpd
    ansible.builtin.service:
      name: httpd
      state: restarted
```

Even though the task lists `validate config` before `restart httpd`,
what actually decides the run order is the order the handlers appear
under `handlers:` — not the order in `notify:`. Keep them in the order
you want them to execute.

</details>

---

### Task 3: `listen` — one notify, multiple handlers

Write a single task that notifies a topic called `webserver restarted`,
and two handlers — `restart httpd` and `clear cache` — that both `listen`
for that topic.

<details>
<summary>Show solution</summary>

```yaml
tasks:
  - name: Deploy httpd config
    ansible.builtin.copy:
      src: httpd.conf
      dest: /etc/httpd/conf/httpd.conf
    notify: "webserver restarted"

handlers:
  - name: restart httpd
    ansible.builtin.service:
      name: httpd
      state: restarted
    listen: "webserver restarted"

  - name: clear cache
    ansible.builtin.file:
      path: /var/cache/httpd
      state: absent
    listen: "webserver restarted"
```

`listen` lets several handlers subscribe to one generic event name, so a
task doesn't need to know the specific handler names — useful when a role
wants to trigger handlers defined in another role.

</details>

---

### Task 4: Forcing handlers to run despite a failure

Normally if a later task in the play fails, any handlers notified earlier
are skipped (they run at the end of the play, after all tasks). Modify
the play so notified handlers still run even if a later task fails.

<details>
<summary>Show solution</summary>

```yaml
- name: Handlers and blocks practice
  hosts: web
  become: true
  force_handlers: true
  tasks:
    - name: Deploy httpd config
      ansible.builtin.copy:
        src: httpd.conf
        dest: /etc/httpd/conf/httpd.conf
      notify: restart httpd

    - name: A later task that might fail
      ansible.builtin.command: /usr/local/bin/healthcheck.sh

  handlers:
    - name: restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted
```

`force_handlers: true` (settable at play, block, or task level, or globally
in `ansible.cfg`) makes pending handlers run even after a task failure
aborts the rest of the play for that host.

</details>

---

### Task 5: Running handlers mid-play with `meta: flush_handlers`

Write a play where a config file is deployed and its handler restarts the
service — but you need the restart to happen *immediately*, before the
next task runs a test against the now-running service (rather than waiting
until the end of the play, which is the default).

<details>
<summary>Show solution</summary>

```yaml
tasks:
  - name: Deploy httpd config
    ansible.builtin.copy:
      src: httpd.conf
      dest: /etc/httpd/conf/httpd.conf
    notify: restart httpd

  - name: Flush handlers now
    ansible.builtin.meta: flush_handlers

  - name: Test that httpd responds
    ansible.builtin.uri:
      url: http://localhost/
      status_code: 200

handlers:
  - name: restart httpd
    ansible.builtin.service:
      name: httpd
      state: restarted
```

</details>

---

### Task 6: `when` on a single condition

Write a task that installs `httpd` only when the target's OS family is
`RedHat`.

<details>
<summary>Show solution</summary>

```yaml
- name: Install httpd on RedHat family
  ansible.builtin.dnf:
    name: httpd
    state: present
  when: ansible_facts['os_family'] == "RedHat"
```

</details>

---

### Task 7: `when` with multiple conditions (AND / OR)

Using the `app_env` variable from Setup, write a task that runs only when
`app_env` is `production` **and** the OS family is `RedHat`. Then write a
second task that runs when `app_env` is `production` **or** `staging`.

<details>
<summary>Show solution</summary>

```yaml
- name: Apply production RedHat tuning
  ansible.builtin.debug:
    msg: "Tuning for production RedHat host"
  when:
    - app_env == "production"
    - ansible_facts['os_family'] == "RedHat"

- name: Deploy non-dev config
  ansible.builtin.debug:
    msg: "Deploying config for {{ app_env }}"
  when: app_env == "production" or app_env == "staging"
```

A list under `when:` is implicit AND — every item must be true. For OR,
use `or` inside a single expression (or `in [...]`, shown in Task 8).

</details>

---

### Task 8: `when` with a registered variable

Write a task that runs `systemctl is-active httpd`, registers the result
(ignoring errors), and a second task that only prints "httpd is down" when
the registered command's return code is not `0`.

<details>
<summary>Show solution</summary>

```yaml
- name: Check httpd status
  ansible.builtin.command: systemctl is-active httpd
  register: httpd_status
  ignore_errors: true
  changed_when: false

- name: Warn if httpd is down
  ansible.builtin.debug:
    msg: "httpd is down"
  when: httpd_status.rc != 0
```

</details>

---

### Task 9: `block` to group related tasks

Group the following three tasks under a single `block` so they share one
`when` condition (`app_env == "production"`) and one `become: true`,
instead of repeating both on every task: install `httpd`, start `httpd`,
deploy `httpd.conf`.

<details>
<summary>Show solution</summary>

```yaml
- name: Production web server setup
  block:
    - name: Install httpd
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: Start httpd
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

    - name: Deploy httpd config
      ansible.builtin.copy:
        src: httpd.conf
        dest: /etc/httpd/conf/httpd.conf
  become: true
  when: app_env == "production"
```

`when` on a `block` is evaluated per-task inside the block (each task gets
the condition applied to it individually), but written once — it's not
evaluated only once for the whole block.

</details>

---

### Task 10: `block` / `rescue` / `always` — basic error handling

Write a block that attempts to deploy a config file and restart `httpd`.
If either step fails, `rescue` should roll back by restoring
`httpd.conf.bak` over `httpd.conf`. Regardless of success or failure, an
`always` section should print a debug message `"deployment attempt
finished"`.

<details>
<summary>Show solution</summary>

```yaml
- name: Deploy with rollback on failure
  block:
    - name: Deploy new httpd config
      ansible.builtin.copy:
        src: httpd.conf
        dest: /etc/httpd/conf/httpd.conf
        backup: true

    - name: Restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted

  rescue:
    - name: Restore previous config on failure
      ansible.builtin.copy:
        src: /etc/httpd/conf/httpd.conf.bak
        dest: /etc/httpd/conf/httpd.conf
        remote_src: true

    - name: Restart httpd with restored config
      ansible.builtin.service:
        name: httpd
        state: restarted

  always:
    - name: Report deployment attempt
      ansible.builtin.debug:
        msg: "deployment attempt finished"
```

If any task in `block` fails, Ansible immediately jumps to `rescue` (the
rest of `block` is skipped) — and after `rescue` completes, the host is
treated as "recovered" and the play continues normally. `always` runs no
matter what happened in `block` or `rescue`, including when nothing
failed at all.

</details>

---

### Task 11: Inspecting the failure inside `rescue`

Modify Task 10's `rescue` section so it first prints the actual error
message from the failed task before restoring the backup.

<details>
<summary>Show solution</summary>

```yaml
- name: Deploy with rollback on failure
  block:
    - name: Deploy new httpd config
      ansible.builtin.copy:
        src: httpd.conf
        dest: /etc/httpd/conf/httpd.conf
        backup: true

    - name: Restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted

  rescue:
    - name: Show what failed
      ansible.builtin.debug:
        msg: "Rescuing from failure: {{ ansible_failed_result.msg | default('unknown error') }}"

    - name: Restore previous config on failure
      ansible.builtin.copy:
        src: /etc/httpd/conf/httpd.conf.bak
        dest: /etc/httpd/conf/httpd.conf
        remote_src: true

    - name: Restart httpd with restored config
      ansible.builtin.service:
        name: httpd
        state: restarted
```

Inside `rescue`, Ansible exposes `ansible_failed_task` (the failed task's
definition) and `ansible_failed_result` (its result, including `.msg`) as
special variables you can reference.

</details>

---

### Task 12: Combining `block`/`rescue` with a handler

Write a block that deploys a config and notifies a `restart httpd`
handler on success. If deployment fails, `rescue` should notify a
different handler, `alert oncall`, instead. Both handlers should be
defined under `handlers:`.

<details>
<summary>Show solution</summary>

```yaml
tasks:
  - name: Deploy with alerting on failure
    block:
      - name: Deploy httpd config
        ansible.builtin.copy:
          src: httpd.conf
          dest: /etc/httpd/conf/httpd.conf
        notify: restart httpd

    rescue:
      - name: Notify on-call of failed deployment
        ansible.builtin.debug:
          msg: "Deployment failed, alerting on-call"
        notify: alert oncall

handlers:
  - name: restart httpd
    ansible.builtin.service:
      name: httpd
      state: restarted

  - name: alert oncall
    ansible.builtin.debug:
      msg: "PagerDuty alert would fire here"
```

Handlers notified inside `block` or `rescue` still queue normally and fire
at the end of the play (or wherever `flush_handlers` is placed) — only one
of the two paths runs per host, so only that path's handler gets notified.

</details>

---

## Part 2 — Q&A

<details>
<summary>Q1. When exactly do handlers run by default?</summary>

At the end of the play, after all tasks for that host have run — not
immediately when `notify` fires. All notified handlers run once each, in
the order they're defined under `handlers:`, regardless of how many times
they were notified.

</details>

<details>
<summary>Q2. If a task notifies a handler but the task result is <code>ok</code> (not <code>changed</code>), does the handler run?</summary>

No. `notify` only queues the handler when the task reports `changed`. A
task that makes no change (`ok`) does not trigger its notified handlers.

</details>

<details>
<summary>Q3. What's the difference between a handler's <code>name</code> and <code>listen</code> topic?</summary>

`name` is the handler's own unique identifier, used directly in a task's
`notify:`. `listen` is a topic string a handler subscribes to — any task
that notifies that topic string triggers every handler listening on it,
even across roles, without the task needing to know individual handler
names.

</details>

<details>
<summary>Q4. Does <code>when</code> on a <code>block</code> get evaluated once for the block, or once per task inside it?</summary>

Once per task inside the block — the condition is applied individually to
every task in the block (and in `rescue`/`always` if placed there too),
it's just written once for convenience. It is not a single gate that
skips the whole block as one unit.

</details>

<details>
<summary>Q5. If a task inside <code>block</code> fails, what happens to the remaining tasks in that <code>block</code>?</summary>

They are skipped. Execution jumps straight to `rescue` (if present). Tasks
after the failed one in `block` never run.

</details>

<details>
<summary>Q6. After <code>rescue</code> completes successfully, is the host considered failed or recovered?</summary>

Recovered. Ansible clears the failure for that host and continues the play
normally with subsequent tasks, as if the block's overall result were
`ok`.

</details>

<details>
<summary>Q7. Does <code>always</code> run if there's no <code>rescue</code> section and a task in <code>block</code> fails?</summary>

Yes. `always` runs unconditionally — whether `block` succeeded, whether
`rescue` existed or ran, or whether the host ultimately still ends up
failed (if there's no `rescue`, `always` still runs, and the host is then
marked failed and stops afterward).

</details>

<details>
<summary>Q8. How do you make handlers run immediately instead of waiting for end-of-play?</summary>

Insert `meta: flush_handlers` as a task at the point where you want any
currently-queued handlers to run right away.

</details>

<details>
<summary>Q9. What does <code>force_handlers: true</code> change?</summary>

Normally, if a task fails and aborts the play for a host, any handlers
that were already notified but not yet run are skipped. `force_handlers:
true` makes those pending handlers run anyway, even though the play
failed for that host.

</details>

<details>
<summary>Q10. Can you use <code>rescue</code> without <code>block</code>, or <code>always</code> without <code>rescue</code>?</summary>

`rescue` and `always` must be attached to a `block` — you can't use them
standalone. `always` doesn't require `rescue` to be present (block +
always alone is valid); `rescue` doesn't require `always` either.

</details>

---

## Part 3 — Challenge

Write a single play that:

1. Wraps package install, config deploy, and service start in a `block`,
   guarded by `when: app_env == "production"`.
2. On failure, `rescue` restores a backup config and notifies a
   `restart httpd` handler.
3. `always` logs a debug message showing `app_env` and whether the block
   succeeded or was rescued, using `ansible_failed_task is not defined` as
   the check.
4. The `restart httpd` handler is flushed immediately after `rescue`
   completes, rather than waiting for end of play.

<details>
<summary>Show solution</summary>

```yaml
tasks:
  - name: Production deployment with rollback
    block:
      - name: Install httpd
        ansible.builtin.dnf:
          name: httpd
          state: present

      - name: Deploy httpd config
        ansible.builtin.copy:
          src: httpd.conf
          dest: /etc/httpd/conf/httpd.conf
          backup: true

      - name: Start httpd
        ansible.builtin.service:
          name: httpd
          state: started
          enabled: true

    rescue:
      - name: Restore previous config
        ansible.builtin.copy:
          src: /etc/httpd/conf/httpd.conf.bak
          dest: /etc/httpd/conf/httpd.conf
          remote_src: true
        notify: restart httpd

      - name: Flush handlers to restart httpd now
        ansible.builtin.meta: flush_handlers

    always:
      - name: Log deployment outcome
        ansible.builtin.debug:
          msg: >-
            app_env={{ app_env }},
            result={{ 'success' if ansible_failed_task is not defined else 'rescued' }}

    when: app_env == "production"

handlers:
  - name: restart httpd
    ansible.builtin.service:
      name: httpd
      state: restarted
```

</details>

---

*Part of `kubearc/linux-practice` — Ansible topic series.*
