# Module 05: K8s Node Configuration
# மாடுல் 05: Kubernetes Node Configuration

---

## 🎯 What? | என்ன?

**English:** Use Ansible to prepare bare-metal/VM nodes for Kubernetes — install containerd, configure kernel params, setup kubeadm, and prepare Talos Linux nodes.

**தமிழ்:** Ansible-ல் K8s nodes prepare — containerd install, kernel params, kubeadm setup, Talos nodes prepare.

---

## 🛠️ K8s Node Preparation (kubeadm)

```yaml
# roles/k8s_node/tasks/main.yml
---
- name: Disable swap (K8s requirement!)
  command: swapoff -a
  changed_when: false

- name: Remove swap from fstab
  lineinfile:
    path: /etc/fstab
    regexp: '.*swap.*'
    state: absent

- name: Load kernel modules
  modprobe:
    name: "{{ item }}"
  loop: [overlay, br_netfilter]

- name: Persist kernel modules
  copy:
    content: |
      overlay
      br_netfilter
    dest: /etc/modules-load.d/k8s.conf

- name: Set sysctl for K8s networking
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    sysctl_file: /etc/sysctl.d/k8s.conf
  loop: "{{ k8s_sysctl | dict2items }}"
  vars:
    k8s_sysctl:
      net.bridge.bridge-nf-call-iptables: "1"
      net.bridge.bridge-nf-call-ip6tables: "1"
      net.ipv4.ip_forward: "1"

- name: Install containerd
  apt:
    name: containerd.io
    state: present

- name: Configure containerd (SystemdCgroup)
  template:
    src: containerd-config.toml.j2
    dest: /etc/containerd/config.toml
  notify: Restart containerd

- name: Install kubeadm, kubelet, kubectl
  apt:
    name:
      - "kubeadm={{ k8s_version }}*"
      - "kubelet={{ k8s_version }}*"
      - "kubectl={{ k8s_version }}*"
    state: present

- name: Hold K8s packages (prevent auto-upgrade)
  dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop: [kubeadm, kubelet, kubectl]
```

---

## 🛠️ Talos Linux Preparation

```yaml
# Talos = immutable OS, no SSH! Ansible prepares the HOSTING environment
# roles/talos_host/tasks/main.yml
---
- name: Install talosctl
  get_url:
    url: "https://github.com/siderolabs/talos/releases/download/{{ talos_version }}/talosctl-linux-amd64"
    dest: /usr/local/bin/talosctl
    mode: '0755'

- name: Generate Talos machine configs
  command: >
    talosctl gen config {{ cluster_name }} https://{{ control_plane_vip }}:6443
    --output-dir /etc/talos/configs
    --with-docs=false
  args:
    creates: /etc/talos/configs/controlplane.yaml

- name: Apply config to control plane nodes
  command: >
    talosctl apply-config --insecure
    --nodes {{ item }}
    --file /etc/talos/configs/controlplane.yaml
  loop: "{{ groups['talos_masters'] }}"

- name: Apply config to worker nodes
  command: >
    talosctl apply-config --insecure
    --nodes {{ item }}
    --file /etc/talos/configs/worker.yaml
  loop: "{{ groups['talos_workers'] }}"

- name: Bootstrap etcd on first control plane
  command: >
    talosctl bootstrap --nodes {{ groups['talos_masters'][0] }}
    --talosconfig /etc/talos/configs/talosconfig
  run_once: true

- name: Get kubeconfig
  command: >
    talosctl kubeconfig --nodes {{ groups['talos_masters'][0] }}
    --talosconfig /etc/talos/configs/talosconfig
    --force /etc/kubernetes/admin.conf
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│      K8S NODE CONFIGURATION CHEAT SHEET          │
├──────────────────────────────────────────────────┤
│ KUBEADM NODE PREP:                               │
│   1. Disable swap                                │
│   2. Load overlay + br_netfilter modules         │
│   3. Enable IP forwarding + bridge-nf-call       │
│   4. Install containerd (SystemdCgroup!)         │
│   5. Install kubeadm + kubelet + kubectl         │
│   6. Hold packages (prevent auto-upgrade)        │
│                                                  │
│ TALOS (no SSH!):                                 │
│   Ansible manages the ADMIN workstation          │
│   talosctl gen config → apply → bootstrap        │
│   No direct node access needed                   │
│                                                  │
│ CONTAINERD CONFIG:                               │
│   SystemdCgroup = true (MUST for K8s!)           │
│   sandbox_image = appropriate pause image        │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] K8s node prerequisites (kernel, swap, sysctl) configure முடியும்
- [ ] containerd install and configure முடியும்
- [ ] kubeadm cluster bootstrap automate முடியும்
- [ ] Talos cluster setup via Ansible explain முடியும்
