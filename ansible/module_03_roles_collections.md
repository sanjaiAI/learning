# Module 03: Roles & Collections
# மாடுல் 03: Roles & Collections

---

## 🎯 What? | என்ன?

**English:** Roles = reusable, shareable automation packages with defined structure. Collections = namespace for distributing roles, modules, and plugins via Ansible Galaxy.

**தமிழ்:** Roles = reusable automation packages. Collections = roles + modules distribute செய்ய Galaxy namespace.

---

## 🛠️ Role Structure

```
roles/
└── docker/
    ├── defaults/
    │   └── main.yml       # Default variables (lowest precedence)
    ├── vars/
    │   └── main.yml       # Role variables (high precedence)
    ├── tasks/
    │   └── main.yml       # Tasks to execute
    ├── handlers/
    │   └── main.yml       # Handlers
    ├── templates/
    │   └── daemon.json.j2 # Jinja2 templates
    ├── files/
    │   └── docker-ce.gpg  # Static files to copy
    ├── meta/
    │   └── main.yml       # Dependencies, Galaxy metadata
    └── README.md          # Usage documentation
```

```yaml
# roles/docker/defaults/main.yml
docker_version: "24.0"
docker_users: []
docker_storage_driver: "overlay2"
docker_log_driver: "json-file"
docker_log_max_size: "100m"

# roles/docker/tasks/main.yml
---
- name: Install prerequisites
  apt:
    name: [apt-transport-https, ca-certificates, curl, gnupg]
    state: present

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker repository
  apt_repository:
    repo: "deb https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present

- name: Install Docker
  apt:
    name: "docker-ce={{ docker_version }}*"
    state: present
  notify: Restart Docker

- name: Configure Docker daemon
  template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json
  notify: Restart Docker

- name: Add users to docker group
  user:
    name: "{{ item }}"
    groups: docker
    append: true
  loop: "{{ docker_users }}"

# roles/docker/handlers/main.yml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
    enabled: true

# roles/docker/meta/main.yml
---
dependencies:
  - role: common
```

---

## 🛠️ Using Roles in Playbooks

```yaml
# playbooks/setup_ci_agent.yml
---
- name: Configure CI Agent
  hosts: ci_agents
  become: true
  
  roles:
    - common              # Basic hardening
    - docker              # Install Docker
    - jenkins_agent       # Configure Jenkins agent
    
  # Override role defaults:
  vars:
    docker_version: "25.0"
    docker_users: ["jenkins", "azureuser"]

# Alternative: include role with conditions
- name: Setup with conditions
  hosts: all
  tasks:
    - name: Include Docker role
      include_role:
        name: docker
      when: install_docker | default(true)
      
    - name: Include monitoring
      include_role:
        name: monitoring
      vars:
        prometheus_port: 9090
```

---

## 🛠️ Collections

```yaml
# requirements.yml — install from Galaxy
---
collections:
  - name: community.general
    version: ">=7.0.0"
  - name: community.docker
    version: ">=3.0.0"
  - name: azure.azcollection
    version: ">=2.0.0"
  - name: google.cloud
    version: ">=1.0.0"

roles:
  - name: geerlingguy.docker
    version: "6.0.0"
  - name: geerlingguy.java
```

```bash
# Install collections and roles
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml

# Create your own role
ansible-galaxy role init roles/ct_hardware
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│       ROLES & COLLECTIONS CHEAT SHEET            │
├──────────────────────────────────────────────────┤
│ ROLE STRUCTURE:                                  │
│   defaults/  = variables (overridable)           │
│   tasks/     = what to do                        │
│   handlers/  = triggered actions                 │
│   templates/ = Jinja2 files                      │
│   files/     = static files                      │
│   meta/      = dependencies, metadata            │
│                                                  │
│ ROLE BEST PRACTICES:                             │
│   ✓ One role = one concern (docker, java, etc.)  │
│   ✓ README with usage example                    │
│   ✓ Defaults for ALL variables                   │
│   ✓ Molecule tests                               │
│   ✓ Version tag releases                         │
│                                                  │
│ COLLECTIONS:                                     │
│   community.general  = swiss army knife          │
│   community.docker   = Docker management         │
│   azure.azcollection = Azure modules             │
│   google.cloud       = GCP modules               │
│                                                  │
│ COMMANDS:                                        │
│   galaxy role init roles/name                    │
│   galaxy collection install -r requirements.yml  │
│   galaxy role install geerlingguy.docker         │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Role from scratch create முடியும்
- [ ] Role in playbook use முடியும்
- [ ] requirements.yml write and install முடியும்
- [ ] Role defaults vs vars precedence explain முடியும்
