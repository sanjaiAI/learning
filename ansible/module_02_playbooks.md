# Module 02: Playbook Deep-Dive
# மாடுல் 02: Playbook Deep-Dive

---

## 🎯 What? | என்ன?

**English:** Master playbook features — variables, templates (Jinja2), loops, conditionals, handlers, blocks, and error handling.

**தமிழ்:** Playbook features master — variables, templates, loops, conditionals, handlers, error handling.

---

## 🛠️ Variables | Variables (மாறிகள்)

```yaml
# Variable precedence (highest → lowest):
# 1. Extra vars (-e "var=value")
# 2. Task vars
# 3. Play vars
# 4. Host vars (host_vars/)
# 5. Group vars (group_vars/)
# 6. Role defaults

# --- In playbook ---
- hosts: ci_agents
  vars:
    java_version: "17"
    packages:
      - docker.io
      - git
      - curl

# --- group_vars/ci_agents.yml ---
jenkins_url: "https://jenkins.internal.com"
agent_labels: ["linux", "docker", "x86_64"]
docker_version: "24.0"

# --- Registered variables (capture output) ---
  tasks:
    - name: Check Docker version
      command: docker --version
      register: docker_result
      changed_when: false    # Don't mark as "changed"

    - name: Show version
      debug:
        msg: "Docker: {{ docker_result.stdout }}"
```

---

## 🛠️ Jinja2 Templates

```yaml
# templates/jenkins-agent.service.j2
[Unit]
Description=Jenkins Agent {{ agent_name }}
After=network.target

[Service]
User={{ jenkins_user }}
ExecStart=/usr/bin/java -jar /opt/jenkins/agent.jar \
  -url {{ jenkins_url }} \
  -secret {{ agent_secret }} \
  -name {{ agent_name }} \
  -workDir /var/jenkins
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```yaml
# In playbook:
    - name: Deploy Jenkins agent service
      template:
        src: jenkins-agent.service.j2
        dest: /etc/systemd/system/jenkins-agent.service
        mode: '0644'
      notify: Restart Jenkins Agent
```

---

## 🛠️ Loops | Loops

```yaml
# Simple loop
    - name: Install packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - docker.io
        - git
        - curl
        - openjdk-17-jdk

# Loop with index
    - name: Create users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        shell: "{{ item.shell | default('/bin/bash') }}"
      loop:
        - {name: jenkins, groups: docker}
        - {name: deploy, groups: "docker,sudo"}

# Loop over dict
    - name: Set sysctl params
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
      loop: "{{ sysctl_params | dict2items }}"
      vars:
        sysctl_params:
          net.core.somaxconn: "65535"
          vm.max_map_count: "262144"
```

---

## 🛠️ Conditionals | நிபந்தனைகள்

```yaml
    - name: Install Docker (Ubuntu)
      apt:
        name: docker.io
        state: present
      when: ansible_os_family == "Debian"

    - name: Install Docker (RHEL)
      yum:
        name: docker
        state: present
      when: ansible_os_family == "RedHat"

    - name: Configure for production
      template:
        src: prod-config.j2
        dest: /etc/app/config.yaml
      when: 
        - environment == "production"
        - enable_monitoring | bool
```

---

## 🛠️ Handlers | Handlers

```yaml
# Handlers run ONCE at end of play, only if notified
  handlers:
    - name: Restart Docker
      service:
        name: docker
        state: restarted

    - name: Reload systemd
      systemd:
        daemon_reload: true

    - name: Restart Jenkins Agent
      service:
        name: jenkins-agent
        state: restarted
      listen: "restart services"    # Multiple tasks can notify this

  tasks:
    - name: Update Docker config
      template:
        src: daemon.json.j2
        dest: /etc/docker/daemon.json
      notify:
        - Reload systemd
        - Restart Docker
```

---

## 🛠️ Blocks & Error Handling

```yaml
    - name: Deploy application
      block:
        - name: Pull image
          docker_image:
            name: "{{ app_image }}"
            source: pull

        - name: Start container
          docker_container:
            name: "{{ app_name }}"
            image: "{{ app_image }}"
            state: started
            
      rescue:
        - name: Rollback — start previous version
          docker_container:
            name: "{{ app_name }}"
            image: "{{ app_image_previous }}"
            state: started

        - name: Alert team
          uri:
            url: "{{ slack_webhook }}"
            method: POST
            body_format: json
            body:
              text: "DEPLOY FAILED: {{ app_name }} on {{ inventory_hostname }}"

      always:
        - name: Clean up temp files
          file:
            path: /tmp/deploy-artifacts
            state: absent
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         PLAYBOOK DEEP-DIVE CHEAT SHEET           │
├──────────────────────────────────────────────────┤
│ VARIABLES:                                       │
│   {{ var }}           = use variable             │
│   {{ var | default }} = with fallback            │
│   register: result   = capture output            │
│   result.stdout, result.rc, result.changed       │
│                                                  │
│ TEMPLATES (Jinja2):                              │
│   {{ variable }}     = insert value              │
│   {% if cond %}      = conditional               │
│   {% for item in %}  = loop                      │
│   {{ var | upper }}  = filter                    │
│                                                  │
│ LOOPS:                                           │
│   loop: [a, b, c]   = simple list               │
│   loop: "{{ list }}" = variable list             │
│   with_dict:         = iterate map               │
│                                                  │
│ ERROR HANDLING:                                  │
│   block/rescue/always = try/catch/finally        │
│   ignore_errors: true = continue on failure      │
│   failed_when:        = custom failure condition │
│   changed_when:       = custom change detection  │
│                                                  │
│ HANDLERS:                                        │
│   Run once at END of play                        │
│   Only if notified (task made a change)          │
│   notify: "Handler Name"                         │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Variable precedence explain முடியும்
- [ ] Jinja2 template write முடியும்
- [ ] Loops (list, dict) use முடியும்
- [ ] Block/rescue/always error handling write முடியும்
- [ ] Handlers vs tasks difference explain முடியும்
