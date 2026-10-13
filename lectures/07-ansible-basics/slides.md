---
theme: seriph
title: "INF345 — Lecture 7: Configuration Management & Ansible Basics"
info: |
  INF345 — Fundamentals of DevOps
  Lecture 7 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# INF 345 — Fundamentals of DevOps

## Lecture 7: Configuration Management & Ansible Basics

<div class="pt-8 opacity-70">
Adil Akhmetov · Lesson 7
</div>

---
layout: default
---

# Recap — Lesson 6 (Container Security)

<v-clicks>

- Why is `ENV API_TOKEN=...` in a Containerfile a leak? <span v-click class="opacity-60">(anyone who can pull the image can read it with `docker inspect` / `docker history`)</span>
- What does `--cap-drop ALL --read-only` buy you? <span v-click class="opacity-60">(a compromised process can't use root's powers or change the image's files)</span>
- Why pin `golang:1.23-alpine` instead of `golang:latest`? <span v-click class="opacity-60">(`latest` moves — the same Containerfile builds a different image next week)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Why configuration management exists: drift and snowflake servers
- [ ] Declarative vs imperative — and what idempotent really means
- [ ] Ansible's model: control node, inventory, modules, SSH
- [ ] Ad-hoc commands
- [ ] Playbooks: plays, tasks, variables, templates, handlers
- [ ] Dry runs and debugging
- [ ] Straight into today's practice: your first playbook

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

You run 30 servers. Each was set up by hand over SSH, by different people,
over two years. A security fix means changing one line in a config file
on all of them. Server 12 has a different version of the file. Server 19
has a manual hotfix nobody wrote down. Server 27 was rebuilt last month
and is missing three packages.

</div>

<div v-click class="mt-8 text-xl font-bold">
Containers solved this for the app. Today: the machines underneath — and
everything that isn't a container.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## Why configuration management

---
---

# Drift and snowflakes

<v-clicks>

- **Configuration drift**: servers that started identical slowly diverge —
  one-off fixes, manual installs, forgotten experiments.
- **Snowflake server**: unique, undocumented, impossible to rebuild. Nobody
  dares to touch it.
- The fix is the same idea as a Containerfile: the server's desired state
  is **written down, in git**, and a tool makes reality match it.
- That's **configuration management** — and with everything in git, it's
  **infrastructure as code**.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
Rule of thumb: if you SSH in and change something by hand, it isn't
done until it's in the playbook.
</div>

---
---

# Imperative vs declarative

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

**Imperative** — *how*: a list of commands

```bash
apt-get install -y nginx
echo "server_tokens off;" >> /etc/nginx/nginx.conf
systemctl restart nginx
```

Run it twice → the line is appended twice, nginx restarts for nothing.

</div>
<div>

**Declarative** — *what*: the end state

```yaml
- ansible.builtin.apt:
    name: nginx
    state: present
- ansible.builtin.lineinfile:
    path: /etc/nginx/nginx.conf
    line: "server_tokens off;"
```

Run it twice → the second run sees it's already true and does nothing.

</div>
</div>

<div v-click class="mt-6 text-base">

**Idempotent**: running it once or ten times gives the same result. That's
what makes it safe to run against 30 servers you're not sure about.

</div>

---
---

# The tools

| Tool | Model | Agent on servers? | Language |
|---|---|---|---|
| **Ansible** | push, over SSH | **no** | YAML |
| Puppet | pull, from a server | yes | Puppet DSL |
| Chef | pull | yes | Ruby |
| Salt | push or pull | usually | YAML |

<v-clicks>

- Ansible needs only **SSH and Python** on the managed machine — nothing to
  install, nothing running between runs.
- It's what Red Hat ships for RHEL automation — **RH294, your Lab 02**.

</v-clicks>

---
layout: section
transition: slide-left
---

# Block 2
## How Ansible works

---
---

# Control node, managed nodes

<v-clicks>

- **Control node**: where you run `ansible` — your laptop, a CI runner.
  Linux or macOS (Windows → WSL).
- **Managed nodes**: the servers. Need SSH + Python, nothing else.
- **Inventory**: the list of managed nodes, in groups.
- **Modules**: units of work — `apt`, `copy`, `template`, `service`,
  `user`… Ansible copies a module to the node over SSH, runs it, reads the
  JSON result, deletes it.
- **Playbook**: YAML that says which modules to run on which hosts.

</v-clicks>

```bash {none|1|2}
pipx install ansible-core     # or: pip install --user ansible-core
ansible --version
```

---
---

# Inventory

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

`inventory.ini`

```ini
[web]
web1.example.com
web2.example.com ansible_port=2222

[db]
db1.example.com

[prod:children]
web
db

[web:vars]
http_port=8080
```

</div>
<div>

<v-clicks>

- **Groups** in `[brackets]`; a host can be in many.
- `:children` — groups of groups.
- Variables per host (same line) or per group (`[group:vars]`).
- Special variables change *how* Ansible connects:
  `ansible_host`, `ansible_user`, `ansible_port`,
  `ansible_connection=local` (no SSH at all).
- `all` always exists.

</v-clicks>

```bash
ansible-inventory --graph
```

</div>
</div>

---
---

# Ad-hoc commands

```bash {none|1|2|3|4|all}
ansible all -m ansible.builtin.ping                        # can I reach them all?
ansible web -m ansible.builtin.setup -a "filter=ansible_distribution*"   # facts
ansible web -m ansible.builtin.apt -a "name=nginx state=present" --become
ansible db  -m ansible.builtin.command -a "uptime"         # read-only one-off
```

<v-clicks>

- `-m` module, `-a` its arguments, `--become` = sudo.
- `ping` isn't ICMP: it logs in over SSH and checks Python works.
- Good for questions ("which kernel is everyone on?") and emergencies.
- Anything you'll need twice belongs in a **playbook**.

</v-clicks>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
<code>ansible all -m ping</code> returns <i>"Failed to connect to the host
via ssh"</i> for <code>localhost</code>. You're running Ansible on that
same machine. What's missing?
</div>

<div v-click class="mt-8 text-lg opacity-70">
<code>ansible_connection=local</code> — without it, Ansible uses SSH for
every host, even localhost.
</div>

---
layout: section
transition: slide-left
---

# Block 3
## Playbooks

---
---

# Anatomy of a playbook

```yaml {all|1-3|5-6|8-12|14-18|all}
- name: Web servers           # a PLAY: which hosts, what to do
  hosts: web
  become: true                # run tasks with sudo

  vars:
    http_port: 8080

  tasks:                      # TASKS run top to bottom, on every host in parallel
    - name: Install nginx
      ansible.builtin.apt:    # the module (fully qualified name)
        name: nginx
        state: present        # declarative: "present", not "install"

    - name: Start and enable nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

---
---

# Running it

```text
$ ansible-playbook site.yml

TASK [Install nginx] *******************************
changed: [web1]
ok: [web2]                    ← already installed — nothing to do

PLAY RECAP *****************************************
web1 : ok=3  changed=2  unreachable=0  failed=0
web2 : ok=3  changed=0  unreachable=0  failed=0
```

<v-clicks>

- **ok** = already in the desired state. **changed** = Ansible changed it.
- Run it again: everything is `ok`, `changed=0`. That's idempotence you can
  *see*.
- A host that fails stops; the others carry on.

</v-clicks>

---
---

# Modules, not shell

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

```yaml
# ✗ always "changed", no idea what happened
- ansible.builtin.shell: mkdir -p /opt/app
```

```yaml
# ✓ checks first, changes only if needed
- ansible.builtin.file:
    path: /opt/app
    state: directory
    mode: "0755"
```

</div>
<div>

<v-clicks>

- `shell`/`command` run anything — so Ansible can't know if it changed
  something. It reports `changed` every run.
- A module knows its own state: directory exists? right mode? → `ok`.
- The everyday set: `file`, `copy`, `template`, `lineinfile`, `apt`/`dnf`,
  `service`, `user`.
- `command` is fine for read-only checks — never for changing state.

</v-clicks>

</div>
</div>

---
---

# Variables and templates

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

`templates/nginx.conf.j2` (Jinja2)

```jinja
server {
    listen {{ http_port }};
    server_name {{ inventory_hostname }};
}
```

```yaml
- ansible.builtin.template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/conf.d/app.conf
```

</div>
<div>

<v-clicks>

- `{{ var }}` is filled in from play `vars:`, inventory, facts…
- One template, every server gets its own rendered file.
- **Precedence**: extra vars on the command line win over everything:
  `ansible-playbook site.yml -e http_port=9090`.
- Never hardcode a value you might change — make it a variable.

</v-clicks>

</div>
</div>

---
---

# Handlers: restart only when needed

```yaml {all|5|8-12|all}
  tasks:
    - name: Nginx config
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/conf.d/app.conf
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

<div v-click class="mt-4 text-sm">

The handler runs **once, at the end of the play**, and **only if** a task
that notifies it reported `changed`. Config unchanged → no restart → no
dropped connections.

</div>

---
---

# Before you run it for real

```bash {none|1|2|3|4|all}
ansible-playbook site.yml --syntax-check   # YAML and structure
ansible-playbook site.yml --check --diff   # dry run: what WOULD change, as a diff
ansible-playbook site.yml --limit web1     # one host first
ansible-playbook site.yml -v               # more output (-vvv: SSH-level detail)
```

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
<code>--check --diff</code> against production is how you find drift: every
"changed" it predicts is a server that no longer matches the code.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You run your playbook twice. The second run says <code>changed=1</code>,
always on the same task. What's the most likely cause?
</div>

<div v-click class="mt-8 text-lg opacity-70">
That task isn't declarative — usually a <code>shell</code>/<code>command</code>
task, which reports "changed" every time because Ansible can't check its
state. Replace it with the module that does the job.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Debugging checklist

---
---

# When it doesn't work

| Symptom | Likely cause |
|---|---|
| `skipping: no hosts matched` | `hosts:` doesn't match a group in the inventory, or the wrong inventory is used |
| `Failed to connect ... via ssh` | Wrong `ansible_host`/user/key — or localhost without `ansible_connection=local` |
| `Missing sudo password` / permission denied | Task needs `become: true` |
| `Destination does not exist` (lineinfile) | Add `create: true` |
| Second run still `changed` | A `shell`/`command` task, or content that differs each run |
| YAML error at line N | Indentation: tasks are a list, module args go under the module |

---
layout: section
transition: slide-left
---

# Block 5
## Today's practice

---
---

# Today's practice — your first playbook

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

Practice 07 gives you `ansible.cfg`, a page and a template. You write:

1. `inventory.ini` — group `web` → `localhost`, local connection
2. `site.yml` — one play that, under `/tmp/inf345-p07/app`:
   - creates the directory (`file`)
   - copies `index.html` (`copy`)
   - renders `app.conf` (`template`)
   - ensures `APP_ENV=production` in `.env` (`lineinfile`)

All values as **variables**. No `shell`.

</div>
<div>

```bash
ansible-inventory --graph
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml      # changed=4
ansible-playbook site.yml      # changed=0
ansible-playbook site.yml -e app_port=9090
```

<div v-click class="mt-4 opacity-70">
Graded automatically: inventory, syntax, the files it creates, the
template and the variable override, idempotence, and real modules only.
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain configuration drift, and why declarative + idempotent fixes it
- [ ] Write an inventory with groups, host vars and a local connection
- [ ] Use ad-hoc commands to check and change hosts
- [ ] Write a playbook with variables, a template and a handler
- [ ] Prove it's idempotent — and dry-run it with `--check --diff`

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice if you didn't wrap it up in session — same
      Maru flow as always
- [ ] **Lab 02 (RHA RH294)** is due **Wed, Nov 4, 23:59** — its first
      chapters are today's material; see "Red Hat labs" in Maru
- [ ] Lab 01 (DO188) closed on Sunday — your final Lab 01 grade is in
      "My grades"

<div class="mt-8 text-sm opacity-60">
Bring a laptop with Ansible working (or Docker) — next week builds on
today's playbook.
</div>

---
layout: end
---

# Next lecture

Playbooks, roles & variables — organising real playbooks: roles, group and
host vars, and where each variable should live.
