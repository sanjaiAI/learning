# Module 01: Ansible Fundamentals
# மாடுல் 01: Ansible அடிப்படைகள்

---

## 🎯 What? | என்ன?

**English:** Ansible = agentless automation. SSH into servers, configure them declaratively. No agent install needed — just SSH access.

**தமிழ்:** Ansible = agentless automation. Servers-க்கு SSH செய்து, declaratively configure செய்வது. Agent install தேவையில்லை — SSH access மட்டும் போதும்.

### Analogy | உதாரணம்
> Restaurant kitchen: Chef (Ansible) gives instructions to sous-chefs (servers) via walkie-talkie (SSH). No special device needed on their side — just listen and execute.

> Kitchen: Chef (Ansible) walkie-talkie (SSH) மூலம் instructions கொடுக்கிறார். Sous-chefs (servers) execute செய்கிறார்கள்.

---

## 📊 Terraform vs Ansible

| Aspect | Terraform | Ansible |
|--------|-----------|---------|
| Purpose | Provision infrastructure | Configure servers |
| Style | Declarative only | Declarative + imperative |
| State | State file | Stateless (checks each run) |
| Agent | No (API calls) | No (SSH) |
| Idempotent | Yes | Yes (if written correctly) |
| Best for | Create VMs, networks | Install software, configure |

> **Together:** Terraform creates the VM → Ansible configures it.

---

## 🛠️ Inventory | Inventory (Server List)

```ini
# inventory/hosts.ini
[ci_agents]
ci-agent-01 ansible_host=10.0.1.10
ci-agent-02 ansible_host=10.0.1.11
ci-agent-03 ansible_host=10.0.1.12

[ct_lab]
ct-rack-01 ansible_host=10.0.2.10
ct-rack-02 ansible_host=10.0.2.11

[k8s_nodes]
k8s-master-01 ansible_host=10.0.3.10
k8s-worker-01 ansible_host=10.0.3.11
k8s-worker-02 ansible_host=10.0.3.12

[ci_agents:vars]
ansible_user=azureuser
ansible_ssh_private_key_file=~/.ssh/id_rsa

[ct_lab:vars]
ansible_user=root
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# inventory/hosts.yml (YAML format — preferred!)
all:
  children:
    ci_agents:
      hosts:
        ci-agent-01: {ansible_host: 10.0.1.10}
        ci-agent-02: {ansible_host: 10.0.1.11}
      vars:
        ansible_user: azureuser
        
    ct_lab:
      hosts:
        ct-rack-01: {ansible_host: 10.0.2.10}
        ct-rack-02: {ansible_host: 10.0.2.11}
      vars:
        ansible_user: root
        
    k8s_nodes:
      children:
        masters:
          hosts:
            k8s-master-01: {ansible_host: 10.0.3.10}
        workers:
          hosts:
            k8s-worker-01: {ansible_host: 10.0.3.11}
```

---

## 🛠️ Ad-Hoc Commands | Quick Commands

```bash
# Ping all hosts (connection test)
ansible all -i inventory/hosts.yml -m ping

# Run command on ci_agents group
ansible ci_agents -m shell -a "df -h"

# Install package
ansible ci_agents -m apt -a "name=docker.io state=present" --become

# Copy file
ansible ct_lab -m copy -a "src=./config.yaml dest=/etc/app/config.yaml"

# Gather facts (system info)
ansible ci-agent-01 -m setup | grep -A 5 "ansible_distribution"

# Reboot
ansible ct_lab -m reboot --become
```

---

## 🛠️ First Playbook | முதல் Playbook

```yaml
# playbooks/setup_ci_agent.yml
---
- name: Setup CI Agent
  hosts: ci_agents
  become: true    # Run as root (sudo)
  
  vars:
    java_version: "17"
    docker_users: ["azureuser", "jenkins"]
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install required packages
      apt:
        name:
          - docker.io
          - openjdk-{{ java_version }}-jdk
          - git
          - curl
          - unzip
        state: present

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: true

    - name: Add users to docker group
      user:
        name: "{{ item }}"
        groups: docker
        append: true
      loop: "{{ docker_users }}"

    - name: Download Jenkins agent jar
      get_url:
        url: "{{ jenkins_url }}/jnlpJars/agent.jar"
        dest: /opt/jenkins/agent.jar
        mode: '0644'
      notify: Restart Jenkins Agent

  handlers:
    - name: Restart Jenkins Agent
      service:
        name: jenkins-agent
        state: restarted
```

```bash
# Run playbook
ansible-playbook -i inventory/hosts.yml playbooks/setup_ci_agent.yml

# Dry-run (check mode — don't change anything!)
ansible-playbook -i inventory/hosts.yml playbooks/setup_ci_agent.yml --check

# Limit to specific host
ansible-playbook -i inventory/hosts.yml playbooks/setup_ci_agent.yml --limit ci-agent-01

# Extra variables
ansible-playbook playbooks/setup_ci_agent.yml -e "java_version=21"

# Verbose output
ansible-playbook playbooks/setup_ci_agent.yml -vvv
```

---

## 📁 Project Structure

```
ansible/
├── ansible.cfg              # Configuration
├── inventory/
│   ├── hosts.yml            # Static inventory
│   └── group_vars/
│       ├── all.yml          # Variables for all hosts
│       ├── ci_agents.yml    # Variables for ci_agents group
│       └── ct_lab.yml       # Variables for ct_lab group
├── playbooks/
│   ├── setup_ci_agent.yml
│   ├── setup_ct_lab.yml
│   └── deploy_app.yml
└── roles/
    ├── common/
    ├── docker/
    └── jenkins_agent/
```

```ini
# ansible.cfg
[defaults]
inventory = inventory/hosts.yml
roles_path = roles
remote_user = azureuser
host_key_checking = false
retry_files_enabled = false
stdout_callback = yaml

[privilege_escalation]
become = true
become_method = sudo
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         ANSIBLE FUNDAMENTALS CHEAT SHEET         │
├──────────────────────────────────────────────────┤
│ CORE CONCEPTS:                                   │
│   Inventory   = which servers                    │
│   Playbook    = what to do (YAML)                │
│   Task        = single action                    │
│   Module      = how to do it (apt, copy, etc.)   │
│   Handler     = triggered on change only         │
│   Role        = reusable package of tasks        │
│                                                  │
│ KEY COMMANDS:                                    │
│   ansible-playbook -i inv play.yml               │
│   --check      = dry-run                         │
│   --diff       = show file changes               │
│   --limit host = run on specific host            │
│   --tags tag   = run specific tasks              │
│   -e "var=val" = override variables              │
│                                                  │
│ IDEMPOTENCY:                                     │
│   Run 10 times = same result                     │
│   state: present (ensure exists)                 │
│   state: absent  (ensure removed)                │
│   Only changes what's needed!                    │
│                                                  │
│ WHEN TO USE:                                     │
│   Terraform creates VM → Ansible configures it   │
│   Ansible ≠ provisioning tool (use Terraform!)   │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: Why Ansible over shell scripts?**
- Idempotent: run multiple times safely (scripts may break on re-run)
- Declarative: describe desired state, not steps
- Error handling: built-in retry, failure handling
- Reusable: roles, variables, templates
- Auditable: YAML is readable, version-controlled

**Q: Ansible vs Chef/Puppet?**
- Ansible: agentless (SSH), push-based, simple YAML
- Chef/Puppet: agent required, pull-based, Ruby/DSL
- Ansible wins for: simplicity, no infra overhead, quick adoption

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Inventory file (INI + YAML) write முடியும்
- [ ] Ad-hoc commands run முடியும்
- [ ] Basic playbook write and execute முடியும்
- [ ] Check mode (dry-run) explain முடியும்
- [ ] Idempotency concept explain முடியும்
