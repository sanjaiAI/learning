# Ansible Learning Path
# Ansible கற்றல் பாதை

> **Scope:** Server Provisioning | Hardware CT Lab | K8s Nodes | CI/CD Agents | Cloud  
> **Format:** Bilingual (English + Tamil) | Analogies | Commands | Cheat Sheets | Interview Q&A

---

## 📋 Module Index | பாட அட்டவணை

| # | Module | Topic | Status |
|---|--------|-------|--------|
| 01 | [Fundamentals](module_01_fundamentals.md) | Inventory, ad-hoc, playbooks, YAML | ⬜ |
| 02 | [Playbook Deep-Dive](module_02_playbooks.md) | Tasks, handlers, variables, templates, loops | ⬜ |
| 03 | [Roles & Collections](module_03_roles_collections.md) | Role structure, Galaxy, namespace, reusability | ⬜ |
| 04 | [Server Provisioning & Hardening](module_04_server_hardening.md) | CIS benchmarks, SSH, firewall, users, patching | ⬜ |
| 05 | [K8s Node Configuration](module_05_k8s_nodes.md) | Talos prep, bare-metal, kubeadm, containerd | ⬜ |
| 06 | [CI/CD Agent Setup](module_06_cicd_agents.md) | Jenkins agents, GitHub runners, build tools, Docker | ⬜ |
| 07 | [Hardware CT Lab — Peripherals](module_07_hardware_peripherals.md) | Arduino, MiniProg, UART, Radmoon setup | ⬜ |
| 08 | [Hardware CT Lab — Power & Flash](module_08_power_flash.md) | Power sequencing, flash orchestration, device farm | ⬜ |
| 09 | [Android Emulator Pipelines](module_09_android_emulator.md) | Goldfish emulator, AVD provisioning, CTS/VTS | ⬜ |
| 10 | [Networking & Load Balancers](module_10_networking.md) | HAProxy, Nginx, firewall rules, DNS | ⬜ |
| 11 | [Application Deployment](module_11_app_deployment.md) | Services, rolling updates, blue-green, configs | ⬜ |
| 12 | [Secrets & Vault Integration](module_12_secrets_vault.md) | Ansible Vault, HashiCorp Vault, encrypted vars | ⬜ |
| 13 | [Testing & Validation](module_13_testing.md) | Molecule, ansible-lint, check mode, CI integration | ⬜ |
| 14 | [Cloud Provisioning](module_14_cloud.md) | Azure modules, GCP modules, dynamic inventory | ⬜ |
| 15 | [Advanced Patterns](module_15_advanced.md) | Custom modules, callback plugins, AWX/Tower, EDA | ⬜ |
| 16 | [Observability & Logging](module_16_observability.md) | ARA, callback plugins, audit trails, centralized logs | ⬜ |

---

## 🏗️ Use Case Coverage

| Use Case | Modules | Your Experience |
|----------|---------|-----------------|
| **Server Provisioning** | 04, 10, 14 | ✅ Production (bare-metal, cloud VMs) |
| **Hardware CT Lab** | 07, 08, 09 | ✅ Production (EB — Arduino, UART, flash) |
| **K8s Nodes** | 05 | ✅ Talos, bare-metal, kubeadm |
| **CI/CD Agents** | 06 | ✅ Jenkins agents, GH runners |
| **Application Deploy** | 11 | ✅ Services, configs, rolling updates |
| **Cloud** | 14 | ✅ Azure & GCP provisioning |

---

## 📅 Study Plan | படிப்பு திட்டம்

| Week | Modules | Focus |
|------|---------|-------|
| 1 | 01-04 | Core Ansible (playbooks, roles, server hardening) |
| 2 | 05-08 | Infrastructure (K8s nodes, CI/CD agents, hardware CT lab) |
| 3 | 09-12 | Workloads (emulators, networking, apps, secrets) |
| 4 | 13-16 | Quality & advanced (testing, cloud, custom modules, observability) |

---

## 🎯 Interview Focus Areas

- Idempotency and declarative vs imperative
- Role design patterns for large-scale infra
- Hardware-in-the-loop automation (unique differentiator!)
- Vault integration and secrets management
- Real projects: EB CT Lab automation, bare-metal K8s provisioning
