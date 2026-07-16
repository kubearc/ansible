# Ansible Practice: Working with Arrays (Lists)

In Ansible, what most languages call an "array" is called a **list**. This practice
file covers defining lists, looping over them, indexing, filtering, and combining
them — the operations you'll use constantly in real playbooks.

Each task gives you a scenario. Try to write the YAML yourself before expanding
the solution.

---

## Task 1: Define a simple list variable

Define a variable called `fruits` containing three items: `apple`, `banana`,
`mango`. Then use `debug` to print the whole list.

<details>
<summary>Solution</summary>

```yaml
---
- hosts: localhost
  vars:
    fruits:
      - apple
      - banana
      - mango
  tasks:
    - name: Print the fruits list
      debug:
        var: fruits
```

Lists can also be written in flow style: `fruits: [apple, banana, mango]`.
Both are valid YAML.
</details>

---

## Task 2: Loop over a list with `loop`

Using the `fruits` list from Task 1, print each fruit on its own line with a
message like `Fruit: apple`.

<details>
<summary>Solution</summary>

```yaml
- name: Print each fruit
  debug:
    msg: "Fruit: {{ item }}"
  loop: "{{ fruits }}"
```

`loop` is the modern, preferred way to iterate. It replaces the older
`with_items`.
</details>

---

## Task 3: Loop with `with_items` (legacy syntax)

Rewrite Task 2 using `with_items` instead of `loop`, and explain in a comment
one key difference between the two.

<details>
<summary>Solution</summary>

```yaml
- name: Print each fruit (legacy style)
  debug:
    msg: "Fruit: {{ item }}"
  with_items: "{{ fruits }}"
```

Key difference: `with_items` automatically flattens one level of nested lists,
while `loop` does not (you'd need the `flatten` filter with `loop` to get the
same behavior). `loop` is recommended for new playbooks.
</details>

---

## Task 4: Index into a list

Print only the second item (`banana`) from the `fruits` list using index
notation.

<details>
<summary>Solution</summary>

```yaml
- name: Print the second fruit
  debug:
    msg: "{{ fruits[1] }}"
```

Ansible/Jinja2 lists are zero-indexed, so `fruits[1]` is `banana`.
</details>

---

## Task 5: Get the length of a list

Print how many items are in the `fruits` list using a filter.

<details>
<summary>Solution</summary>

```yaml
- name: Print number of fruits
  debug:
    msg: "There are {{ fruits | length }} fruits"
```
</details>

---

## Task 6: Append an item to a list at runtime

Start with `fruits` as defined above. Add `grape` to the list during play
execution and print the updated list.

<details>
<summary>Solution</summary>

```yaml
- name: Add grape to the list
  set_fact:
    fruits: "{{ fruits + ['grape'] }}"

- name: Print updated list
  debug:
    var: fruits
```

Lists in Jinja2 are immutable in place — you rebuild the variable using `+`
concatenation and reassign it with `set_fact`.
</details>

---

## Task 7: Loop over a list of dictionaries

Define a list called `users` where each item is a dictionary with `name` and
`shell` keys. Loop over it and create each user with the specified shell.

<details>
<summary>Solution</summary>

```yaml
---
- hosts: webservers
  become: true
  vars:
    users:
      - name: alice
        shell: /bin/bash
      - name: bob
        shell: /bin/zsh
  tasks:
    - name: Create users with specified shells
      user:
        name: "{{ item.name }}"
        shell: "{{ item.shell }}"
        state: present
      loop: "{{ users }}"
```

Accessing dict keys inside a loop uses dot notation: `item.name`, `item.shell`.
</details>

---

## Task 8: Combine two lists

You have `web_packages: [httpd, php]` and `db_packages: [mariadb-server]`.
Combine them into a single list called `all_packages` and install everything
in one task.

<details>
<summary>Solution</summary>

```yaml
---
- hosts: all
  become: true
  vars:
    web_packages:
      - httpd
      - php
    db_packages:
      - mariadb-server
  tasks:
    - name: Install all packages
      dnf:
        name: "{{ web_packages + db_packages }}"
        state: present
```

Note: `dnf`/`yum` modules accept a list directly for `name:` — you don't even
need a loop for package installation.
</details>

---

## Task 9: Filter a list with `select`/`reject`

Given a list of integers `numbers: [1,2,3,4,5,6,7,8,9,10]`, print only the
even numbers using a Jinja2 filter (no manual loop/if needed).

<details>
<summary>Solution</summary>

```yaml
- name: Print only even numbers
  debug:
    msg: "{{ numbers | select('even') | list }}"
```

`select()` keeps items that match a test; `reject()` does the opposite (keeps
items that do NOT match). Common tests: `even`, `odd`, `defined`, `match`.
</details>

---

## Task 10: Check list membership with `when`

Using the `fruits` list, run a task only if `mango` is in the list.

<details>
<summary>Solution</summary>

```yaml
- name: Run only if mango is in the fruits list
  debug:
    msg: "We have mango!"
  when: "'mango' in fruits"
```

The `in` operator works the same way it does in Python — it checks list
membership.
</details>

---

## Quick Reference Table

| Operation                     | Syntax                                  |
|--------------------------------|------------------------------------------|
| Define a list                  | `mylist: [a, b, c]`                     |
| Loop over a list               | `loop: "{{ mylist }}"`                  |
| Legacy loop                    | `with_items: "{{ mylist }}"`            |
| Index an item                  | `mylist[0]`                             |
| List length                    | `mylist \| length`                      |
| Concatenate lists              | `list1 + list2`                         |
| Filter items                   | `mylist \| select('test') \| list`      |
| Membership check                | `'x' in mylist`                         |
| Loop over list of dicts        | `item.key` inside the loop              |

---

**Notes for self-checking:** Run each play with `ansible-playbook file.yml -v`
to see the full `debug` output including data types — this helps confirm
whether you're working with a list vs. a string that looks like one.
