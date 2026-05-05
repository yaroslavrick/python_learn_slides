---
title: Ansible
---

![Ansible YAML playbook with tasks](/assets/images/topics/ansible.svg)
<!-- .element: class="title-illustration" -->

# Ansible

Agentless infrastructure-as-code in YAML.

---

## Why Ansible?

- **Agentless** — only needs SSH + Python on the target
- **YAML playbooks** — declarative, readable
- **Idempotent** — run the same playbook twice, second run is a no-op
- **Wide module catalog** — packages, services, files, cloud APIs, containers
- **Written in Python** — natural fit for Python shops

For "configure these servers, install these packages, copy this file" — Ansible is hard to beat.

---

## Install

```bash
uv tool install ansible
ansible --version
# ansible [core 2.18.x]
```

`ansible` is the ad-hoc CLI; `ansible-playbook` runs full playbooks.

---

## Inventory — what to manage

```ini
# inventory.ini
[web]
web1.example.com
web2.example.com

[db]
db1.example.com

[all:vars]
ansible_user=deploy
ansible_python_interpreter=/usr/bin/python3
```

YAML inventory is also supported. Group hosts; reference groups in playbooks.

---

## Ad-hoc commands

```bash
ansible all -i inventory.ini -m ping
# web1.example.com | SUCCESS => {"changed": false, "ping": "pong"}

ansible web -m shell -a "uptime"
# web1.example.com | CHANGED | rc=0 >>
# 14:32:01 up 12 days, ...

ansible web -m apt -a "name=nginx state=present" --become
# (installs nginx via sudo)
```

`--become` runs as root via sudo. Useful for one-off checks; for anything repeatable, write a playbook.

---

## A first playbook

```yaml
# webserver.yml
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: true
```

```bash
ansible-playbook -i inventory.ini webserver.yml
```

--

## Adding a config + handler

```yaml
  tasks:
    # ...as before...
    - name: Copy our config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: reload nginx

  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

`notify` queues the handler; it runs once at the end if anything fired it.

---

## Handlers — react to changes

A **handler** runs only if a task `notify`s it. Multiple notifications collapse to one run at the end of the play.

```yaml
tasks:
  - name: Update config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: reload nginx

  - name: Update other config
    template:
      src: site.conf.j2
      dest: /etc/nginx/sites-enabled/site.conf
    notify: reload nginx        # → still reloads only once

handlers:
  - name: reload nginx
    service: { name: nginx, state: reloaded }
```

Idiomatic for "if anything changed, restart/reload the service".

---

## Variables and templates

```yaml
# group_vars/web.yml
nginx_worker_processes: 4
app_port: 8000
domains:
  - example.com
  - www.example.com
```

Variables in `group_vars/<group>.yml` apply to every host in that group.

--

## Jinja2 template

```jinja
# templates/nginx.conf.j2
worker_processes {{ nginx_worker_processes }};

server {
    listen 80;
    server_name {% raw %}{{ domains | join(' ') }}{% endraw %};
    location / { proxy_pass http://localhost:{{ app_port }}; }
}
```

```yaml
- name: Generate nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

The `template` module renders Jinja2 with the playbook's variables in scope.

---

## Idempotence

Ansible modules check current state and only act if it differs:

```yaml
- name: Ensure nginx is installed
  apt: { name: nginx, state: present }
# First run:  CHANGED  (installs)
# Second run: ok       (already installed; no action)
```

Avoid `command` and `shell` modules where a real module exists — `command` always reports `changed`. When you must use `command`, set `creates:` or `removes:` so Ansible can skip it.

---

## Roles — reusable units

```
roles/
└── webserver/
    ├── tasks/main.yml
    ├── handlers/main.yml
    ├── templates/nginx.conf.j2
    ├── files/something.txt
    ├── defaults/main.yml         # default variable values
    ├── vars/main.yml             # higher-priority variables
    └── meta/main.yml             # role dependencies, supported platforms
```

Use them:

```yaml
- hosts: web
  become: true
  roles:
    - webserver
    - app
```

Roles encapsulate one concern. Browse [Ansible Galaxy](https://galaxy.ansible.com/) for community roles before writing your own.

---

## ansible-vault — secrets

Encrypted YAML for things like API keys and passwords:

```bash
ansible-vault create secrets.yml
# (opens editor; you set a password)

ansible-vault edit secrets.yml
ansible-vault rekey secrets.yml          # change password

ansible-playbook deploy.yml --ask-vault-pass
```

In CI, use `--vault-password-file` pointing at a script that fetches from your secret manager (Vault, AWS Secrets Manager, ...).

---

## Common modules

| Module | What it does |
| --- | --- |
| `apt` / `yum` / `dnf` | Install OS packages |
| `pip` | Install Python packages |
| `service` / `systemd` | Manage services |
| `copy` / `template` | Push files, render Jinja templates |
| `file` | Permissions, symlinks, dirs |
| `lineinfile` / `blockinfile` | Edit files in place (use sparingly) |
| `user` / `group` | Manage accounts |
| `cron` | Schedule jobs |
| `git` | Clone / pull a repo |
| `docker_container` / `docker_compose_v2` | Container ops |
| `community.aws.*`, `google.cloud.*` | Cloud APIs |

`ansible-doc <module>` opens the docs for any module.

---

## When to use Ansible — and when not

| Use Ansible for... | Don't use Ansible for... |
| --- | --- |
| Configure existing servers | Provisioning the cloud itself (use Terraform / OpenTofu) |
| Run one-off ops on many hosts | Real-time orchestration (use a queue / scheduler) |
| Bootstrap a new server | Building artifacts (use CI + Docker) |
| Roll out a deploy | Stateful data migration (use proper migrations) |

Ansible is glue. Pair it with **Terraform** for cloud provisioning and **Docker** for app delivery — those are different jobs.

---

## What's next

- **Deployment** — Ansible + Docker + GitHub Actions, end-to-end
