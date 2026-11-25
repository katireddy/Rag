# Ansible – DevOps Interview Quick Notes

## 1. What is Ansible?
Ansible is an agentless automation and configuration-management tool used to:
- Install packages and configure servers
- Deploy applications
- Manage users, files, and services
- Automate infrastructure tasks and orchestration

**Architecture**
- **Control node:** Machine where Ansible is installed and playbooks run.
- **Managed nodes:** Remote servers Ansible manages.
- **Linux connection:** Usually SSH; no An Ansible agent is required.
- **Windows connection:** Commonly WinRM.

## 2. Installation

```bash
# macOS
brew install ansible

# Ubuntu
sudo apt update
sudo apt install ansible -y

# Verify
ansible --version
```

Important commands: `ansible`, `ansible-playbook`, `ansible-galaxy`, `ansible-inventory`, `ansible-doc`.

## 3. Project structure

```text
ansible-project/
├── ansible.cfg
├── inventory.ini
├── site.yml
├── group_vars/
├── host_vars/
├── roles/
├── files/
└── templates/
```

- `ansible.cfg`: Ansible configuration.
- `inventory.ini`: Managed hosts and groups.
- `site.yml`: Main playbook.
- `group_vars/`, `host_vars/`: Group- and host-specific variables.
- `roles/`: Reusable automation.
- `files/`: Static files.
- `templates/`: Jinja2 templates.

## 4. Inventory

INI example (`inventory.ini`):

```ini
[webservers]
web01 ansible_host=192.168.1.10
web02 ansible_host=192.168.1.11

[dbservers]
db01 ansible_host=192.168.1.20
```

Test connectivity:

```bash
ansible all -i inventory.ini -m ansible.builtin.ping
ansible webservers -i inventory.ini -m ping
```

Ansible's `ping` module tests Ansible connectivity; it is not the Linux `ping` command.

## 5. Ad-hoc commands

Ad-hoc commands run one-off tasks without a playbook.

```bash
# Check connectivity
ansible all -i inventory.ini -m ping

# Check uptime
ansible all -i inventory.ini -m command -a "uptime"

# Check disk
ansible all -i inventory.ini -m command -a "df -h"

# Install Nginx on Debian/Ubuntu hosts
ansible webservers -i inventory.ini -m apt -a "name=nginx state=present" -b

# Start service
ansible webservers -i inventory.ini -m service -a "name=nginx state=started enabled=yes" -b

# Create user
ansible all -i inventory.ini -m user -a "name=devops state=present" -b
```

Useful flags:
- `-i`: inventory
- `-m`: module
- `-a`: module arguments
- `-b`: privilege escalation (become)
- `-K`: prompt for become password
- `-u`: remote user
- `-v`, `-vvv`: verbosity

## 6. Playbooks

A playbook is a YAML file containing plays and tasks.

`install-nginx.yml`:

```yaml
---
- name: Install and start Nginx
  hosts: webservers
  become: true

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Start and enable Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Run and validate:

```bash
ansible-playbook -i inventory.ini install-nginx.yml
ansible-playbook --syntax-check install-nginx.yml
ansible-playbook --check -i inventory.ini install-nginx.yml
ansible-playbook --diff -i inventory.ini install-nginx.yml
ansible-playbook -i inventory.ini install-nginx.yml --limit web01
ansible-playbook -i inventory.ini install-nginx.yml -vvv
```

`--check` is a simulation; not every module supports full check-mode prediction.

## 7. Variables

```yaml
- name: Deploy application
  hosts: webservers
  vars:
    app_name: payment-service
    app_port: 8080

  tasks:
    - name: Print application details
      ansible.builtin.debug:
        msg: "Deploying {{ app_name }} on port {{ app_port }}"
```

Variable sources include play vars, `vars_files`, inventory, `group_vars`, `host_vars`, role defaults/vars, and command-line extra vars.

```bash
ansible-playbook -i inventory.ini site.yml -e "app_port=9090"
```

Extra vars (`-e`) have very high precedence.

## 8. Facts

Facts are system details gathered from managed nodes, such as OS, hostname, IP, and memory.

```yaml
- name: Display host facts
  hosts: all
  tasks:
    - ansible.builtin.debug:
        msg: "{{ ansible_facts['distribution'] }}"
```

```bash
ansible all -i inventory.ini -m setup
```

## 9. Handlers

Handlers run when notified, commonly to restart a service only when a configuration changes.

```yaml
tasks:
  - name: Deploy Nginx config
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart Nginx

handlers:
  - name: Restart Nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

Handlers normally run at the end of the play. Multiple notifications to the same handler normally trigger it once.

## 10. Loops and conditions

```yaml
- name: Install packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - nginx
```

```yaml
- name: Install package on Ubuntu
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts['distribution'] == "Ubuntu"
```

## 11. Templates (Jinja2)

`app.conf.j2`:

```jinja2
APP_NAME={{ app_name }}
APP_PORT={{ app_port }}
```

Task:

```yaml
- name: Generate config
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
```

- `copy`: Copies a static file.
- `template`: Renders a Jinja2 template with variables.

## 12. Ansible Vault

Vault encrypts sensitive data such as passwords and tokens.

```bash
ansible-vault create secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-playbook site.yml --ask-vault-pass
```

Keep Vault passwords in a secure secrets manager or protected CI/CD secret—not in Git.

## 13. Roles

Roles organize reusable automation into a standard directory structure.

```bash
ansible-galaxy role init nginx
```

Typical structure:

```text
roles/nginx/
├── tasks/main.yml
├── handlers/main.yml
├── templates/
├── files/
├── defaults/main.yml
├── vars/main.yml
├── meta/main.yml
└── README.md
```

Use a role:

```yaml
- name: Configure web servers
  hosts: webservers
  roles:
    - nginx
```

- `tasks/`: Main tasks.
- `handlers/`: Restart/reload tasks.
- `defaults/`: Default variables (lower precedence).
- `vars/`: Role variables (higher precedence).
- `templates/`, `files/`: Dynamic and static files.
- `meta/`: Metadata and dependencies.

## 14. Ansible Galaxy

Ansible Galaxy is a hub and CLI ecosystem for discovering, sharing, and installing reusable Ansible content.

```bash
# Search for roles
ansible-galaxy role search nginx

# Install a role
ansible-galaxy role install geerlingguy.nginx

# List roles
ansible-galaxy role list

# Install a collection
ansible-galaxy collection install community.general

# List collections
ansible-galaxy collection list
```

**Role vs collection**
- **Role:** Reusable structure for tasks, handlers, templates, and variables.
- **Collection:** Package of Ansible content; can include roles, modules, plugins, and playbooks.

Pin and review external dependencies before using them in production.

## 15. Useful modules and concepts

| Module/concept | Purpose |
|---|---|
| `apt`, `dnf` | Package management |
| `copy`, `template` | File deployment |
| `file` | Files, directories, permissions |
| `service`, `systemd_service` | Service management |
| `user`, `group` | User and group management |
| `command`, `shell` | Run commands |
| `git` | Clone/update repositories |
| `uri` | HTTP/API requests |
| `debug` | Display values |
| `register` | Save task result |
| `tags` | Select tasks to run |
| `delegate_to` | Run a task on another host |
| `serial` | Run hosts in batches |
| `block`, `rescue`, `always` | Error handling |

### `command` vs `shell`

- `command`: Runs a command directly; no shell pipes, redirects, or expansion.
- `shell`: Runs through a shell; supports pipes and redirection. Prefer `command` when shell features are unnecessary.

### `register`

```yaml
- name: Check Nginx
  ansible.builtin.command: systemctl is-active nginx
  register: nginx_status
  changed_when: false
  failed_when: false

- name: Print status
  ansible.builtin.debug:
    var: nginx_status.stdout
```

### Tags

```yaml
tasks:
  - name: Install Nginx
    ansible.builtin.apt:
      name: nginx
      state: present
    tags: [install]
```

```bash
ansible-playbook site.yml --tags install
```

### Rolling deployment

```yaml
- name: Deploy in batches
  hosts: webservers
  serial: 2
  tasks:
    - name: Deployment step
      ansible.builtin.debug:
        msg: "Deploying to this batch"
```

Use health checks and load-balancer draining/registration for safe production rolling deployments.

## 16. CI/CD integration

Typical workflow:

1. Developer pushes code to Git.
2. Jenkins or GitHub Actions starts a pipeline.
3. Pipeline checks out code and runs tests/lint/syntax checks.
4. Pipeline obtains credentials securely.
5. Pipeline runs an Ansible playbook.
6. Ansible configures servers or deploys the application.
7. Pipeline reports the result.

Example GitHub Actions workflow:

```yaml
name: Ansible validation

on:
  pull_request:
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Ansible
        run: |
          python -m pip install --user ansible-core ansible-lint

      - name: Check syntax
        run: ansible-playbook --syntax-check site.yml

      - name: Lint playbook
        run: ansible-lint site.yml
```

For deployment, configure SSH access and protected secrets; do not commit private keys or plaintext passwords. Pin third-party actions to reviewed commit SHAs in production.

## 17. Ansible vs Terraform vs Puppet vs Chef

| Tool | Main purpose |
|---|---|
| Ansible | Configuration management, deployment, automation |
| Terraform | Provision infrastructure through provider APIs |
| Puppet | Configuration management |
| Chef | Configuration management using Ruby-based recipes |

Common pattern: Terraform creates AWS VPCs, subnets, security groups, and EC2 instances; Ansible configures the operating system and deploys applications.

## 18. DevSecOps best practices

- Use Ansible Vault or a secrets manager for secrets.
- Use SSH keys and least-privilege sudo access.
- Review playbook changes through pull requests.
- Run `ansible-lint` and syntax checks in CI.
- Pin and review Galaxy roles and collections.
- Protect production inventories and deployment permissions.
- Avoid printing secrets in logs.
- Use check mode where supported and test changes before production.

## 19. Interview questions

1. **What is Ansible?** Agentless automation and configuration-management tool.
2. **What is a control node?** Machine where Ansible runs.
3. **What is a managed node?** Server Ansible manages.
4. **What is inventory?** List/grouping of managed hosts.
5. **What is a playbook?** YAML automation definition containing plays and tasks.
6. **What is a module?** Unit of work, such as `apt`, `copy`, or `service`.
7. **What is idempotency?** Repeated runs maintain desired state without unnecessary changes.
8. **What is Ansible Galaxy?** Hub/CLI ecosystem for reusable roles and collections.
9. **What is a role?** Reusable, organized Ansible content.
10. **What is a handler?** Task triggered by `notify`, often for service restart.
11. **What are facts?** Information gathered from managed hosts.
12. **What is Vault?** Encryption for sensitive Ansible data.
13. **What is `become`?** Privilege escalation, commonly sudo.
14. **What is `register`?** Stores task output in a variable.
15. **What is `when`?** Conditional task execution.
16. **What is `serial`?** Runs a play across hosts in batches.
17. **`copy` vs `template`?** Static copy vs Jinja2-rendered file.
18. **`command` vs `shell`?** Direct command execution vs shell interpretation.
19. **`import_tasks` vs `include_tasks`?** Static import at parse time vs dynamic inclusion at runtime.
20. **What is dynamic inventory?** Inventory discovered from sources such as AWS EC2.
21. **How do you manage dev/QA/prod?** Separate inventories and group variables.
22. **How do you troubleshoot?** Use `-vvv`, `--syntax-check`, `debug`, `ansible-doc`, and verify SSH/permissions.
23. **How do you avoid restarting a service every run?** Use a handler notified by a task that changes.
24. **How do you deploy without downtime?** Rolling batches, health checks, and load-balancer integration.
25. **How do you protect secrets in CI/CD?** Protected pipeline secrets or a secrets manager, optionally with Vault-encrypted files.

## 20. Hands-on practice order

1. Install Ansible on your Mac.
2. Create an inventory and test SSH connectivity.
3. Run ad-hoc commands.
4. Write a playbook to install and start Nginx.
5. Add variables, loops, and conditions.
6. Add a template and handler.
7. Create an Nginx role with `ansible-galaxy role init`.
8. Encrypt secrets using Ansible Vault.
9. Add `ansible-lint` and syntax checks to GitHub Actions.
10. Practice a deployment to Linux VMs or EC2 instances.

## Official documentation

- https://docs.ansible.com/
- https://docs.ansible.com/projects/ansible/latest/playbook_guide/
- https://docs.ansible.com/projects/ansible/latest/galaxy/user_guide.html
- https://docs.ansible.com/projects/ansible/latest/vault_guide/
