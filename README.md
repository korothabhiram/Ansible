# 🅰️ Ansible

## 📘 What is Ansible?

Ansible is an open-source **IT automation tool** used to manage configurations, deploy applications, orchestrate workflows, and provision infrastructure across on-premises and cloud environments. Unlike agent-based tools, Ansible is **agentless**—it connects over SSH (or WinRM for Windows) and uses simple, human-readable **YAML** to describe automation as code.

---

## 🙋‍♂️ Who Should Use Ansible?

- **DevOps Engineers**: Automate configuration management and deployments.
- **Sysadmins**: Manage fleets of servers without writing custom scripts.
- **SREs**: Enforce consistent, repeatable system state at scale.
- **Cloud Engineers**: Provision and configure cloud resources alongside Terraform.
- **Tech Learners**: Pick up automation without needing to learn a new agent architecture.

---

## 🎯 Why Use Ansible?

- 🔌 Agentless—just needs SSH and Python on the target
- 📜 Human-readable YAML playbooks, easy to version control
- 🔁 Idempotent—running the same playbook twice is safe
- 🧩 Reusable roles and a huge community (Ansible Galaxy)
- 🎯 Fine-grained control via tags, handlers, and conditionals

---
# 🅰️ Ansible Reference Guide

Ansible leans more on **structure and concepts** than a huge list of one-off commands. Below is a mix of the ad-hoc commands you'll actually use, plus the core concepts (inventory, playbooks, facts, roles, Galaxy, tags, handlers) you'll reach for constantly.

---

## ⚡ Ad-Hoc Commands

Ad-hoc commands run a single module against hosts without writing a playbook—handy for quick checks and one-off tasks.

| Command | Description |
|--------|-------------|
| `ansible all -m ping` | 🏓 Check connectivity to all hosts |
| `ansible all -m command -a "uptime"` | ⏱️ Run a shell command on all hosts |
| `ansible webservers -m shell -a "df -h"` | 💾 Run a shell command on a specific group |
| `ansible all -m setup` | 🧠 Gather and display host facts |
| `ansible all -m copy -a "src=file.txt dest=/tmp/file.txt"` | 📤 Copy a file to hosts |
| `ansible all -m yum -a "name=nginx state=present" -b` | 📦 Install a package (with sudo/become) |
| `ansible all -m service -a "name=nginx state=restarted" -b` | 🔁 Restart a service |
| `ansible all -m user -a "name=deploy state=present" -b` | 👤 Create a user |
| `ansible-doc <module>` | 📘 Show documentation for a module |
| `ansible-inventory --list` | 📋 Show the full inventory as JSON |

---

## 📋 Inventory

The **inventory** is the list of hosts (and host groups) Ansible manages. It's defined in an INI or YAML file, default location `/etc/ansible/hosts`, though you'll almost always point to your own with `-i`.

**Static inventory (INI style):**

```ini
[webservers]
web1.example.com
web2.example.com ansible_host=192.168.1.20

[dbservers]
db1.example.com

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

| Command | Description |
|--------|-------------|
| `ansible-inventory -i inventory.ini --list` | 📋 Dump inventory as JSON |
| `ansible-inventory -i inventory.ini --graph` | 🌳 Show inventory as a group tree |
| `ansible <group> -i inventory.ini -m ping` | 🏓 Target a specific group |

Hosts can belong to multiple groups, and groups can be nested (`[webservers:children]`) to build hierarchies like `prod` → `webservers` + `dbservers`.

---

## 🔄 Dynamic Inventory

Instead of hand-maintaining a static file, a **dynamic inventory** script or plugin queries a live source (AWS, Azure, GCP, VMware, etc.) at run time and builds the host list automatically—essential once infrastructure is created/destroyed by tools like Terraform.

```bash
# Example: AWS EC2 dynamic inventory plugin (aws_ec2.yml)
ansible-inventory -i aws_ec2.yml --graph
ansible all -i aws_ec2.yml -m ping
```

```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
```

- Dynamic inventory plugins live in collections (e.g. `amazon.aws`, `azure.azcollection`, `google.cloud`).
- Great for autoscaling environments where hosts come and go—no manual file edits.
- You can combine multiple inventory sources by passing a directory to `-i`.

---

## 📜 Playbooks

A **playbook** is a YAML file describing one or more **plays**—each targeting a set of hosts and running an ordered list of tasks (directly, or via roles).

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true
  vars:
    http_port: 80
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Deploy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

| Command | Description |
|--------|-------------|
| `ansible-playbook site.yml` | ▶️ Run a playbook |
| `ansible-playbook site.yml -i inventory.ini` | 📋 Run against a specific inventory |
| `ansible-playbook site.yml --check` | 🧪 Dry run—show what would change |
| `ansible-playbook site.yml --diff` | 🔍 Show file/config diffs during a dry run |
| `ansible-playbook site.yml --limit webservers` | 🎯 Restrict run to a host/group |
| `ansible-playbook site.yml -e "env=prod"` | 🔑 Pass extra variables |
| `ansible-playbook site.yml --syntax-check` | ✔️ Validate YAML/playbook syntax only |
| `ansible-playbook site.yml -v` / `-vvv` | 🐛 Increase verbosity for debugging |

---

## 🧠 Ansible Facts

**Facts** are system/environment data (OS, IP addresses, memory, disks, hostname, etc.) that Ansible automatically discovers from each target host at the start of a play via the built-in `setup` module—no extra config needed. Facts let playbooks make decisions based on what a host actually looks like, instead of hardcoding assumptions.

```yaml
- name: Show some facts
  debug:
    msg: "{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }} on {{ ansible_facts['default_ipv4']['address'] }}"

- name: Only run on Debian-family hosts
  apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == "Debian"
```

| Command | Description |
|--------|-------------|
| `ansible all -m setup` | 🧠 Gather and print all facts for hosts |
| `ansible all -m setup -a "filter=ansible_distribution*"` | 🔍 Gather only facts matching a filter |
| `ansible-playbook site.yml --start-at-task="Task name"` | ⏭️ Skip ahead, still gathers facts first by default |

- Disable fact gathering with `gather_facts: false` in a play if you don't need it—speeds up runs.
- **Custom facts** can be added via `fact_caching`, a `setup` module fact file, or the `set_fact` module mid-play.
- Facts are namespaced under `ansible_facts` (e.g. `ansible_facts['hostname']`); the older bare `ansible_hostname` style still works but the namespaced form is preferred.

---

## 🎭 Roles

A **role** is a reusable, self-contained bundle of tasks, handlers, variables, files, templates, and defaults, organized in a standard directory structure—the main way Ansible content is shared and reused.

```
roles/
└── webserver/
    ├── tasks/main.yml       # the actual steps
    ├── handlers/main.yml    # notify targets (e.g. restart service)
    ├── templates/           # Jinja2 templates (.j2)
    ├── files/                # static files to copy
    ├── vars/main.yml         # role-specific variables (high precedence)
    ├── defaults/main.yml     # default variables (low precedence, easily overridden)
    └── meta/main.yml         # role metadata & dependencies
```

| Command | Description |
|--------|-------------|
| `ansible-galaxy init roles/<name>` | 🆕 Scaffold a new role's directory structure |
| `ansible-galaxy role list` | 📋 List installed roles |

Using a role in a playbook:

```yaml
- hosts: webservers
  roles:
    - webserver
    - { role: firewall, tags: ['security'] }
```

---

## 🏷️ Skip Tags & Tags

**Tags** let you label tasks/plays so you can run or skip just a subset—handy for large playbooks where you don't want to re-run everything.

```yaml
tasks:
  - name: Install packages
    apt:
      name: nginx
      state: present
    tags: [packages, install]

  - name: Configure firewall
    ufw:
      rule: allow
      port: '80'
    tags: [security]
```

| Command | Description |
|--------|-------------|
| `ansible-playbook site.yml --tags "packages"` | 🎯 Run only tasks tagged `packages` |
| `ansible-playbook site.yml --tags "packages,security"` | 🎯 Run tasks matching any of several tags |
| `ansible-playbook site.yml --skip-tags "security"` | ⏭️ Run everything except tasks tagged `security` |
| `ansible-playbook site.yml --list-tags` | 📋 List all tags defined in a playbook |

---

## 🔔 Notify & Handlers

**Handlers** are tasks that only run when explicitly **notified**, and only run **once**, at the end of the play, no matter how many tasks notify them—ideal for "restart the service, but only if config actually changed."

```yaml
tasks:
  - name: Update config file
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart app service

  - name: Update another config
    template:
      src: extra.conf.j2
      dest: /etc/app/extra.conf
    notify: Restart app service   # same handler, still only fires once

handlers:
  - name: Restart app service
    service:
      name: app
      state: restarted
```

- A handler fires only if the notifying task reports `changed`.
- Multiple tasks can notify the same handler—it still runs just once, after all tasks in the play finish.
- Use `meta: flush_handlers` if you need handlers to run immediately, mid-play, instead of waiting for the end.

---

## 🌌 Ansible Galaxy

**Ansible Galaxy** is the hub (and CLI) for finding, sharing, and installing community roles and collections instead of writing everything from scratch.

| Command | Description |
|--------|-------------|
| `ansible-galaxy install <namespace.role>` | 📥 Install a role from Galaxy |
| `ansible-galaxy collection install <namespace.collection>` | 📥 Install a collection (e.g. `amazon.aws`) |
| `ansible-galaxy role list` | 📋 List installed roles |
| `ansible-galaxy collection list` | 📋 List installed collections |
| `ansible-galaxy init <role_name>` | 🆕 Scaffold a new role |
| `ansible-galaxy install -r requirements.yml` | 📄 Install everything listed in a requirements file |

```yaml
# requirements.yml
roles:
  - name: geerlingguy.nginx
collections:
  - name: amazon.aws
```

---

## 🧠 Tip

💡 Use `ansible-doc <module>` to see any module's full documentation and examples, e.g.:

```bash
ansible-doc template
```

---

> ✅ Keep this README as a reference for your Ansible journey. Contributions welcome!
> ⭐ Star this repo if you found it helpful!
