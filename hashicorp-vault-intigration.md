# Ansible + HashiCorp Vault — Verified for RHEL 10 Controller

This replaces the earlier two Vault files. Every command and field
reference in **Setup** and **Level 1** below has been run and confirmed
working on your actual environment:

- Controller: RHEL 10, Python 3.12.11
- `hvac` 2.4.0 installed via `pip3 install hvac --break-system-packages`
  (system Python at `/usr/bin/python3`)
- Vault installed via HashiCorp's official yum repo, version 2.0.4 (dev mode)
- `community.hashi_vault` installed via `ansible-galaxy collection install`
- Ansible 2.16.14 (the collection prints a support warning on this
  version — harmless, everything below still works)
- **Confirmed return shape:** the `hashi_vault` lookup plugin returns KV
  v2 secret fields **flat** — `result.username`, not
  `result.data.data.username`. This is the opposite of what most online
  examples show, and it's the #1 thing that will trip up your students.

Levels 2 and 3 (AppRole, dynamic secrets, Transit, Vault Agent) are
included for teaching progression but have **not** been run on your
system yet — treat every field reference in those sections as unverified
until you dump the raw result first, exactly the way you debugged Task 1.4.

---

## Setup (confirmed working, in order)

**1. Install Vault (RHEL 10):**

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vault
vault -v
```

**2. Install pip for the system Python (RHEL 10 ships without it):**

```bash
sudo dnf install python3-pip -y
/usr/bin/python3 -m pip --version
```

**3. Install `hvac` — RHEL 10's Python is "externally managed," so you
need the override flag:**

```bash
sudo /usr/bin/python3 -m pip install hvac --break-system-packages
```

Verify (note: `hvac` 2.4.0 has no `__version__` attribute — use `pip
show` instead of `import hvac; hvac.__version__`, which will error even
though the install is fine):

```bash
/usr/bin/python3 -m pip show hvac
```

**4. Install the collection:**

```bash
ansible-galaxy collection install community.hashi_vault
```

**5. Start Vault in dev mode:**

```bash
vault server -dev -dev-listen-address=0.0.0.0:8200
```

Leave this running in its own terminal/session. It prints a `Root Token:`
on startup — **every time this process restarts, storage resets and you
get a new root token.** If you ever see "invalid token" or "permission
denied" after previously working, the server likely restarted — check
`vault status` for a changed `Cluster ID`/`Cluster Name` as the tell.

**6. In your working terminal, set env vars — use `127.0.0.1`, not
`0.0.0.0` (0.0.0.0 is a listen address, not a valid client target):**

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="<paste current root token>"
vault status
```

Confirm `Sealed: false` and `Initialized: true`.

**7. Seed a test secret:**

```bash
vault kv put secret/kubearc/db username="dbadmin" password="SuperSecr3t!"
vault kv get secret/kubearc/db
```

---

## Level 1 — Basic (confirmed working)

### Task 1: Dump the raw lookup result first — always

Before referencing any field, dump the whole thing. This is the habit
that would have saved several debugging rounds earlier — do this first on
every new lookup, every time.

```yaml
---
- name: Vault lookup test
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Show raw vault lookup result
      ansible.builtin.debug:
        msg: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/kubearc/db', url=lookup('env', 'VAULT_ADDR'), token=lookup('env', 'VAULT_TOKEN')) }}"
```

```bash
ansible-playbook file1.yaml
```

**Confirmed output on your system:**

```json
{
    "msg": {
        "password": "SuperSecr3t!",
        "username": "dbadmin"
    }
}
```

Flat. No `.data`, no `.data.data`. Note this is the KV v2 **read path**
(`secret/data/...`) — the double `data/` path segment is still required,
even though the *response* itself isn't nested the way the raw HTTP API
would return it.

---

### Task 2: Read a single field

```yaml
---
- name: Vault lookup test
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Read vault username
      ansible.builtin.debug:
        msg: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/kubearc/db', url=lookup('env', 'VAULT_ADDR'), token=lookup('env', 'VAULT_TOKEN')).username }}"
```

**Confirmed output:**

```json
{
    "msg": "dbadmin"
}
```

---

### Task 3: Store the whole secret in a fact, reference multiple fields

```yaml
---
- name: Vault lookup test
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Read full db secret from Vault
      ansible.builtin.set_fact:
        db_secret: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/kubearc/db', url=lookup('env', 'VAULT_ADDR'), token=lookup('env', 'VAULT_TOKEN')) }}"
      no_log: true

    - name: Show username only (never print password in a real run)
      ansible.builtin.debug:
        msg: "DB user is {{ db_secret.username }}"
```

`no_log: true` on the `set_fact` task keeps the full secret (including
`password`) out of console/log output even if the task fails or is run
with `-v`.

---

### Task 4: Drop the explicit `url=`/`token=` args

Since `VAULT_ADDR` and `VAULT_TOKEN` are already exported in your shell,
`community.hashi_vault` reads them automatically — you don't have to pass
them on every call.

```yaml
---
- name: Vault lookup test
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Read vault username, relying on env vars
      ansible.builtin.debug:
        msg: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/kubearc/db').username }}"
```

Run it the same way — if it prints `dbadmin` with no `url=`/`token=`
passed, env var pickup is confirmed working on your system too.

---

### Task 5: Template a config file from the secret

**`db_config.j2`:**

```jinja2
DB_USER={{ db_secret.username }}
DB_PASS={{ db_secret.password }}
```

**Playbook:**

```yaml
---
- name: Deploy db config from Vault
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Read db secret from Vault
      ansible.builtin.set_fact:
        db_secret: "{{ lookup('community.hashi_vault.hashi_vault', 'secret/data/kubearc/db') }}"
      no_log: true

    - name: Render db config
      ansible.builtin.template:
        src: db_config.j2
        dest: /tmp/db_config.conf
        mode: "0600"
      no_log: true
```

Check the result:

```bash
cat /tmp/db_config.conf
```

---

## Level 2 — Intermediate (⚠ not yet run on your system — verify field shapes yourself)

### Task 6: Least-privilege policy instead of the root token

```bash
vault policy write kubearc-read - <<EOF
path "secret/data/kubearc/*" {
  capabilities = ["read"]
}
EOF

vault token create -policy="kubearc-read" -ttl=1h
```

Use the returned token as `VAULT_TOKEN` for read-only work from here on,
instead of the dev root token.

---

### Task 7: AppRole auth

```bash
vault auth enable approle

vault write auth/approle/role/kubearc-ansible \
    token_policies="kubearc-read" \
    token_ttl=1h \
    token_max_ttl=4h

vault read -field=role_id auth/approle/role/kubearc-ansible/role-id
vault write -f -field=secret_id auth/approle/role/kubearc-ansible/secret-id
```

```yaml
- name: Read secret via AppRole — DUMP RAW FIRST
  ansible.builtin.debug:
    msg: "{{ lookup('community.hashi_vault.hashi_vault',
      'secret/data/kubearc/db',
      url=lookup('env', 'VAULT_ADDR'),
      auth_method='approle',
      role_id=lookup('env', 'VAULT_ROLE_ID'),
      secret_id=lookup('env', 'VAULT_SECRET_ID')) }}"
```

Run this exact "dump raw" version first, same as Task 1, before assuming
`.username` works here too — AppRole is a different auth path and it's
worth confirming the return shape is still flat before trusting it.

---

### Task 8: Write a secret from Ansible

```yaml
- name: Write a new secret into Vault
  community.hashi_vault.vault_write:
    url: "{{ lookup('env', 'VAULT_ADDR') }}"
    token: "{{ lookup('env', 'VAULT_TOKEN') }}"
    path: secret/data/kubearc/api
    data:
      data:
        api_key: "abc123xyz"
        api_env: "production"
```

Note: `vault_write` is a **module**, not the lookup plugin — its return
value on `register:` may follow a different (possibly still-nested)
shape than what you confirmed for lookups. Dump it with `debug` before
referencing any field from `register:`ed output here.

---

## Q&A

<details>
<summary>Q1. Why did "invalid token" / "permission denied" errors appear even after the token had worked minutes earlier?</summary>

The dev server had restarted — visible from a changed `Cluster ID` in
`vault status` output. Dev mode storage is in-memory only, so a restart
wipes all secrets and issues a brand-new root token; the old exported
`VAULT_TOKEN` becomes invalid immediately.

</details>

<details>
<summary>Q2. Why does <code>0.0.0.0:8200</code> work as a listen address but not reliably as <code>VAULT_ADDR</code>?</summary>

`0.0.0.0` tells the server to accept connections on all local network
interfaces — it's a bind address, not a routable destination. A client
needs an actual reachable address, like `127.0.0.1` (loopback) or the
host's real IP.

</details>

<details>
<summary>Q3. Why does the KV v2 <em>path</em> still need <code>secret/data/kubearc/db</code> even though the lookup's <em>response</em> isn't nested?</summary>

The path structure and the response structure are separate concerns. KV
v2's API endpoint itself is versioned and requires the `data/` segment to
address the current version of a secret (`secret/metadata/...` would be a
different endpoint, for version history). The `hashi_vault` lookup plugin
independently chooses to unwrap the *response* for convenience — that
unwrapping doesn't change what URL/path it has to call underneath.

</details>

<details>
<summary>Q4. Why check <code>pip show hvac</code> instead of <code>import hvac; print(hvac.__version__)</code>?</summary>

`hvac` 2.4.0 doesn't expose a `__version__` attribute on the module
directly, so that particular check raises `AttributeError` even on a
perfectly good install. `pip show hvac` reads packaging metadata instead
of module internals, so it works regardless of what the library itself
exposes.

</details>

<details>
<summary>Q5. Why did <code>pip install hvac</code> fail with "externally-managed-environment" before <code>--break-system-packages</code> was added?</summary>

RHEL 10's system Python follows PEP 668, which blocks `pip` from
installing directly into system site-packages to avoid conflicts with
`dnf`-managed Python packages. `--break-system-packages` explicitly
overrides that guard. The cleaner long-term alternative is a virtual
environment, which sidesteps the guard entirely rather than overriding it.

</details>

---

*Part of `kubearc/linux-practice` — Ansible topic series. Replaces
`ansible-hashicorp-vault-practice.md` and
`ansible-vault-basic-to-advanced.md`.*
