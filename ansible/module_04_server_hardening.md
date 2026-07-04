# Module 04: Server Provisioning & Hardening
# மாடுல் 04: Server Provisioning & Hardening (சர்வர் பாதுகாப்பு)

---

## 🎯 What? | என்ன?

**English:** Secure baseline configuration for all servers — SSH hardening, firewall, user management, CIS benchmarks, patching, and audit logging.

**தமிழ்:** எல்லா servers-க்கும் secure baseline — SSH hardening, firewall, user management, CIS benchmarks, patching.

---

## 🛠️ Complete Hardening Playbook

```yaml
# playbooks/harden_server.yml
---
- name: Server Hardening (CIS Baseline)
  hosts: all
  become: true
  
  vars:
    allowed_ssh_users: ["azureuser", "deploy"]
    ssh_port: 22
    ntp_servers: ["time.google.com", "time.cloudflare.com"]
    
  tasks:
    # --- SSH Hardening ---
    - name: Harden SSH config
      template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
        validate: "sshd -t -f %s"    # Validate before apply!
      notify: Restart SSH

    # --- Firewall (UFW) ---
    - name: Install UFW
      apt: {name: ufw, state: present}

    - name: Set default deny incoming
      ufw: {direction: incoming, policy: deny}

    - name: Allow SSH
      ufw:
        rule: allow
        port: "{{ ssh_port }}"
        proto: tcp

    - name: Allow from internal network
      ufw:
        rule: allow
        src: "10.0.0.0/8"

    - name: Enable UFW
      ufw: {state: enabled}

    # --- User Management ---
    - name: Create service accounts
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups | default(omit) }}"
        shell: "{{ item.shell | default('/bin/bash') }}"
        create_home: true
      loop: "{{ service_accounts }}"

    - name: Deploy SSH authorized keys
      authorized_key:
        user: "{{ item.name }}"
        key: "{{ item.ssh_key }}"
        exclusive: true    # Remove unauthorized keys!
      loop: "{{ service_accounts }}"
      when: item.ssh_key is defined

    - name: Disable root login
      user:
        name: root
        password: "!"    # Lock root password

    # --- System Hardening ---
    - name: Set sysctl security params
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        sysctl_file: /etc/sysctl.d/99-security.conf
      loop: "{{ security_sysctl | dict2items }}"
      vars:
        security_sysctl:
          net.ipv4.conf.all.rp_filter: "1"
          net.ipv4.conf.default.rp_filter: "1"
          net.ipv4.icmp_echo_ignore_broadcasts: "1"
          net.ipv4.conf.all.accept_redirects: "0"
          net.ipv4.conf.all.send_redirects: "0"
          kernel.randomize_va_space: "2"

    # --- Automatic Security Updates ---
    - name: Install unattended-upgrades
      apt:
        name: [unattended-upgrades, apt-listchanges]
        state: present

    - name: Configure auto security updates
      template:
        src: 50unattended-upgrades.j2
        dest: /etc/apt/apt.conf.d/50unattended-upgrades

    # --- Audit Logging ---
    - name: Install auditd
      apt: {name: auditd, state: present}

    - name: Configure audit rules
      copy:
        content: |
          # Monitor sudo usage
          -w /etc/sudoers -p wa -k sudo_changes
          # Monitor SSH keys
          -w /home -p wa -k home_changes
          # Monitor cron
          -w /etc/crontab -p wa -k cron_changes
        dest: /etc/audit/rules.d/custom.rules
      notify: Restart auditd

    # --- NTP ---
    - name: Configure NTP
      template:
        src: chrony.conf.j2
        dest: /etc/chrony/chrony.conf
      notify: Restart chronyd

    # --- Remove unnecessary packages ---
    - name: Remove insecure packages
      apt:
        name: [telnet, rsh-client, rsh-server]
        state: absent

  handlers:
    - name: Restart SSH
      service: {name: sshd, state: restarted}
    - name: Restart auditd
      service: {name: auditd, state: restarted}
    - name: Restart chronyd
      service: {name: chronyd, state: restarted}
```

### SSH Config Template

```jinja2
# templates/sshd_config.j2
Port {{ ssh_port }}
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers {{ allowed_ssh_users | join(' ') }}
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
PermitEmptyPasswords no
UsePAM yes
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│      SERVER HARDENING CHEAT SHEET                │
├──────────────────────────────────────────────────┤
│ SSH:                                             │
│   ✓ Key-only auth (no passwords)                 │
│   ✓ Disable root login                           │
│   ✓ AllowUsers whitelist                         │
│   ✓ MaxAuthTries 3                               │
│                                                  │
│ FIREWALL:                                        │
│   ✓ Default deny incoming                        │
│   ✓ Allow only required ports                    │
│   ✓ Allow internal network CIDR                  │
│                                                  │
│ USERS:                                           │
│   ✓ Named accounts (no shared root)              │
│   ✓ SSH key only (exclusive: true)               │
│   ✓ Principle of least privilege                 │
│                                                  │
│ PATCHING:                                        │
│   ✓ unattended-upgrades for security             │
│   ✓ Regular full patching schedule               │
│                                                  │
│ AUDIT:                                           │
│   ✓ auditd for file monitoring                   │
│   ✓ Centralized logging (ELK/Splunk)            │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: How do you ensure consistent server hardening across 100+ servers?**
- Ansible role: `hardening` applied to ALL servers via base playbook
- Idempotent: safe to re-run (detects and fixes drift)
- CI: Molecule tests validate role correctness
- Compliance: periodic audit playbook reports non-compliant servers

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] SSH hardening playbook write முடியும்
- [ ] Firewall rules configure முடியும்
- [ ] User management automate முடியும்
- [ ] CIS benchmark checks explain முடியும்
