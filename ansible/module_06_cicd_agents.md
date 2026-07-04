# Module 06: CI/CD Agent Setup
# மாடுல் 06: CI/CD Agent Setup

---

## 🎯 What? | என்ன?

**English:** Automate provisioning of Jenkins agents, GitHub Actions runners, and build tool chains on bare-metal/VMs using Ansible roles.

**தமிழ்:** Jenkins agents, GitHub Actions runners, build tools-ஐ Ansible roles-ல் automate செய்வது.

---

## 🛠️ Jenkins Agent Role

```yaml
# roles/jenkins_agent/defaults/main.yml
jenkins_url: "https://jenkins.internal.com"
jenkins_agent_name: "{{ inventory_hostname }}"
jenkins_user: jenkins
jenkins_home: /var/jenkins
jenkins_labels: ["linux", "docker"]
java_version: "17"
build_tools:
  - gradle
  - maven
  - cmake
  - ninja-build
  - python3-pip

# roles/jenkins_agent/tasks/main.yml
---
- name: Create Jenkins user
  user:
    name: "{{ jenkins_user }}"
    home: "{{ jenkins_home }}"
    shell: /bin/bash
    groups: [docker]

- name: Install Java
  apt:
    name: "openjdk-{{ java_version }}-jdk"
    state: present

- name: Install build tools
  apt:
    name: "{{ build_tools }}"
    state: present

- name: Create agent directory
  file:
    path: "{{ jenkins_home }}"
    state: directory
    owner: "{{ jenkins_user }}"
    mode: '0755'

- name: Download agent.jar
  get_url:
    url: "{{ jenkins_url }}/jnlpJars/agent.jar"
    dest: "{{ jenkins_home }}/agent.jar"
    owner: "{{ jenkins_user }}"

- name: Deploy systemd service
  template:
    src: jenkins-agent.service.j2
    dest: /etc/systemd/system/jenkins-agent.service
  notify: [Reload systemd, Restart Jenkins Agent]

- name: Start agent service
  service:
    name: jenkins-agent
    state: started
    enabled: true
```

---

## 🛠️ GitHub Actions Runner

```yaml
# roles/github_runner/tasks/main.yml
---
- name: Create runner user
  user:
    name: github-runner
    home: /opt/actions-runner
    shell: /bin/bash
    groups: [docker]

- name: Download Actions Runner
  get_url:
    url: "https://github.com/actions/runner/releases/download/v{{ runner_version }}/actions-runner-linux-x64-{{ runner_version }}.tar.gz"
    dest: /tmp/actions-runner.tar.gz

- name: Extract runner
  unarchive:
    src: /tmp/actions-runner.tar.gz
    dest: /opt/actions-runner
    remote_src: true
    owner: github-runner

- name: Configure runner
  command: >
    ./config.sh --url {{ github_org_url }}
    --token {{ runner_token }}
    --name {{ inventory_hostname }}
    --labels {{ runner_labels | join(',') }}
    --unattended
  args:
    chdir: /opt/actions-runner
    creates: /opt/actions-runner/.runner
  become_user: github-runner

- name: Install as service
  command: ./svc.sh install github-runner
  args:
    chdir: /opt/actions-runner
    creates: /etc/systemd/system/actions.runner.*.service

- name: Start runner service
  command: ./svc.sh start
  args:
    chdir: /opt/actions-runner
```

---

## 🛠️ Build Environment (Embedded — AOSP/QNX)

```yaml
# roles/embedded_build_agent/tasks/main.yml
---
- name: Install AOSP build dependencies
  apt:
    name:
      - git-core
      - gnupg
      - flex
      - bison
      - build-essential
      - zip
      - curl
      - zlib1g-dev
      - libc6-dev-i386
      - x11proto-core-dev
      - libx11-dev
      - lib32z1-dev
      - libgl1-mesa-dev
      - libxml2-utils
      - xsltproc
      - unzip
      - fontconfig
      - python3
      - repo
    state: present

- name: Set Git config for repo tool
  git_config:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    scope: global
  loop:
    - {name: user.email, value: "ci@internal.com"}
    - {name: user.name, value: "CI Build Agent"}
  become_user: "{{ jenkins_user }}"

- name: Mount NFS build cache
  mount:
    path: /mnt/build-cache
    src: "{{ nfs_server }}:/export/build-cache"
    fstype: nfs
    opts: "rw,hard,intr"
    state: mounted

- name: Create ccache directory
  file:
    path: /mnt/build-cache/ccache
    state: directory
    owner: "{{ jenkins_user }}"

- name: Configure ccache
  copy:
    content: |
      max_size = 50G
      cache_dir = /mnt/build-cache/ccache
    dest: "/home/{{ jenkins_user }}/.ccache/ccache.conf"
    owner: "{{ jenkins_user }}"
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│       CI/CD AGENT SETUP CHEAT SHEET              │
├──────────────────────────────────────────────────┤
│ JENKINS AGENT:                                   │
│   Java + agent.jar + systemd service             │
│   Labels for job routing                         │
│   Docker group for container builds              │
│                                                  │
│ GITHUB RUNNER:                                   │
│   actions-runner + config.sh + svc.sh            │
│   Registration token (short-lived!)              │
│   Labels for workflow targeting                  │
│                                                  │
│ BUILD TOOLS (Embedded):                          │
│   AOSP: repo, Java, ccache, large disk           │
│   QNX:  QNX SDP, license server access           │
│   NFS mount for shared build cache               │
│                                                  │
│ OPTIMIZATION:                                    │
│   ccache = compiler cache (50%+ faster rebuilds) │
│   NFS build cache = shared across agents         │
│   SSD for workspace = faster I/O                 │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Jenkins agent role create and deploy முடியும்
- [ ] GitHub runner setup automate முடியும்
- [ ] Embedded build environment configure முடியும்
- [ ] Build cache strategy explain முடியும்
