# Lesson 45. Ansible

## Goal
Configure many servers the same way, repeatably, from code in Git.

## Theory

- **Agentless**: Ansible connects over SSH and runs small modules with Python on the target.
- **Declarative and idempotent**: you describe the desired state ("nginx installed, config is X, service running"). Running twice changes nothing the second time.
- **Inventory**: which hosts, in which groups.
- **Playbook**: YAML list of **plays**; each play maps hosts to **tasks**.
- **Module**: the unit of work: `apt`, `copy`, `template`, `service`, `user`, `lineinfile`...
- **Role**: a reusable package of tasks, templates, handlers and defaults.
- **Handler**: a task that runs only when notified (e.g. reload nginx after its config changed).

## Setup

```bash
# or sudo apt install ansible
pipx install ansible-core

ansible --version
```

Project layout:

```
infra/
├── ansible.cfg
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── all.yml
│       └── web.yml
├── playbooks/
│   └── site.yml
└── roles/
    └── nginx/
        ├── defaults/main.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/site.conf.j2
```

`ansible.cfg`:

```ini
[defaults]
inventory = inventory/hosts.ini
remote_user = admin
host_key_checking = True
forks = 20

[ssh_connection]
pipelining = True
```

`inventory/hosts.ini`:

```ini
[web]
web1 ansible_host=10.0.1.11
web2 ansible_host=10.0.1.12

[db]
db1 ansible_host=10.0.2.11

[prod:children]
web
db
```

## Ad-hoc commands

```bash
ansible all -m ping
ansible web -a "uptime"

# -b = become (sudo)
ansible web -b -m apt -a "name=htop state=present update_cache=yes"

# facts
ansible all -m setup -a "filter=ansible_distribution*"
```

## Playbook

`playbooks/site.yml`:

```yaml
- name: Baseline for all servers
  hosts: all
  become: true
  tasks:
    - name: Install base packages
      ansible.builtin.apt:
        name: [curl, vim, htop, unattended-upgrades]
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Admin users
      ansible.builtin.user:
        name: "{{ item }}"
        groups: sudo
        append: true
        shell: /bin/bash
      loop: "{{ admin_users }}"

    - name: Harden SSH
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/10-hardening.conf
        content: |
          PermitRootLogin no
          PasswordAuthentication no
        mode: "0644"
        validate: /usr/sbin/sshd -t -f %s
      notify: Reload ssh

  handlers:
    - name: Reload ssh
      ansible.builtin.service:
        name: ssh
        state: reloaded

- name: Web servers
  hosts: web
  become: true
  roles:
    - nginx
```

`inventory/group_vars/all.yml`:

```yaml
admin_users: [alice, bob]
```

## Role with a template

`roles/nginx/defaults/main.yml`:

```yaml
nginx_server_name: example.com
nginx_upstream_port: 8000
```

`roles/nginx/templates/site.conf.j2`:

```jinja
server {
    listen 80;
    server_name {{ nginx_server_name }};
    location / {
        proxy_pass http://127.0.0.1:{{ nginx_upstream_port }};
        proxy_set_header Host $host;
    }
}
```

`roles/nginx/tasks/main.yml`:

```yaml
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present

- name: Site config
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/app.conf
    mode: "0644"
  notify: Reload nginx

- name: Enable site
  ansible.builtin.file:
    src: /etc/nginx/sites-available/app.conf
    dest: /etc/nginx/sites-enabled/app.conf
    state: link
  notify: Reload nginx

- name: nginx running
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

`roles/nginx/handlers/main.yml`:

```yaml
- name: Reload nginx
  ansible.builtin.shell: nginx -t && systemctl reload nginx
```

## Running safely

```bash
ansible-playbook playbooks/site.yml --syntax-check

# dry run with diffs
ansible-playbook playbooks/site.yml --check --diff

# one host first
ansible-playbook playbooks/site.yml --limit web1

ansible-playbook playbooks/site.yml --tags nginx

# everyone
ansible-playbook playbooks/site.yml

ansible-lint
```

For rolling changes across a fleet: `serial: 2` in the play (2 hosts at a time) and `max_fail_percentage`.

## Secrets: Ansible Vault

```bash
ansible-vault create inventory/group_vars/db/vault.yml
ansible-vault edit inventory/group_vars/db/vault.yml

# or --vault-password-file
ansible-playbook site.yml --ask-vault-pass
```

Convention: `vault_db_password` in the vault file, `db_password: "{{ vault_db_password }}"` in plain vars, so you can grep for variable names.

## Good habits

- Prefer modules over `shell`/`command`. If you must use them, add `creates:` or `changed_when:` to stay idempotent.
- Use fully qualified module names (`ansible.builtin.copy`).
- Run the same playbook twice in CI: second run must report `changed=0`.
- Test roles with Molecule in containers or VMs.
- Ansible configures servers; Terraform (next lesson) creates them. They work together.

## Practice

1. Make an inventory of two or three VMs. Run `ansible all -m ping`.
2. Turn the hardening checklist from lesson 33 into a playbook. Run it twice: the second run must show no changes.
3. Write the nginx role above, deploy it, change the template variable, watch the handler reload nginx.
4. Store a password in Ansible Vault and use it in a template.
5. Use `serial: 1` to roll a config change through the web group one host at a time.

## Check yourself

- What does idempotent mean, and why does it matter?
- When does a handler run?
- Why is `--check --diff` the first command you run against production?

---

<p align="left">
  <a href="media/45-ansible.mp4"><img width="640" src="media/45-ansible.gif" alt="Lesson 45 animation: Ansible"></a>
</p>

## Next
[Lesson 46. Terraform and infrastructure as code](46-terraform.md)
