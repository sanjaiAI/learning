# Terraform Learning Path
# Terraform கற்றல் பாதை

> **Scope:** Azure + GCP | Infrastructure-as-Code | Interview Preparation  
> **Format:** Bilingual (English + Tamil) | Analogies | Commands | Cheat Sheets | Interview Q&A

---

## 📋 Module Index | பாட அட்டவணை

| # | Module | Topic | Status |
|---|--------|-------|--------|
| 01 | [Fundamentals](module_01_fundamentals.md) | HCL, Providers, Resources, State basics | ⬜ |
| 02 | [State Management](module_02_state_management.md) | Remote state, locking, workspaces, import | ⬜ |
| 03 | [Modules & Reusability](module_03_modules.md) | Module structure, registry, composition | ⬜ |
| 04 | [Variables & Expressions](module_04_variables_expressions.md) | Types, locals, functions, dynamic blocks | ⬜ |
| 05 | [Azure Networking](module_05_azure_networking.md) | VNets, subnets, NSGs, peering, Private Endpoints | ⬜ |
| 06 | [Azure Compute](module_06_azure_compute.md) | VMs, VMSS, availability zones, managed disks | ⬜ |
| 07 | [Azure AKS](module_07_azure_aks.md) | Cluster provisioning, node pools, workload identity | ⬜ |
| 08 | [Azure Storage & Data](module_08_azure_data.md) | Blob, PostgreSQL, Key Vault, Private Endpoints | ⬜ |
| 09 | [GCP Networking](module_09_gcp_networking.md) | VPCs, subnets, firewall rules, Cloud NAT | ⬜ |
| 10 | [GCP Compute](module_10_gcp_compute.md) | GCE, instance groups, preemptible VMs | ⬜ |
| 11 | [GCP GKE](module_11_gcp_gke.md) | Cluster, node pools, workload identity, Autopilot | ⬜ |
| 12 | [GCP Storage & Data](module_12_gcp_data.md) | GCS, Cloud SQL, Secret Manager | ⬜ |
| 13 | [Security & Compliance](module_13_security_compliance.md) | Checkov, tfsec, Sentinel, OPA | ⬜ |
| 14 | [CI/CD for Terraform](module_14_cicd_terraform.md) | GitHub Actions, Jenkins, GitOps, Atlantis | ⬜ |
| 15 | [Multi-Environment Patterns](module_15_multi_env.md) | Workspaces vs directories, promotion, DRY patterns | ⬜ |
| 16 | [Advanced Patterns](module_16_advanced.md) | Import, moved blocks, drift detection, Crossplane | ⬜ |

---

## 🏗️ Cloud Platforms Covered

| Platform | Services | Your Experience |
|----------|----------|-----------------|
| **Azure** | VNets, AKS, VMs, PostgreSQL, Blob, Key Vault, Private Endpoints | ✅ Production (TVS, EB) |
| **GCP** | VPCs, GKE, GCE, GCS, Cloud SQL, Secret Manager | ✅ Production |

---

## 📅 Study Plan | படிப்பு திட்டம்

| Week | Modules | Focus |
|------|---------|-------|
| 1 | 01-04 | Core Terraform (HCL, state, modules, variables) |
| 2 | 05-08 | Azure (networking, compute, AKS, data) |
| 3 | 09-12 | GCP (networking, compute, GKE, data) |
| 4 | 13-16 | Security, CI/CD, patterns, advanced |

---

## 🎯 Interview Focus Areas

- State management (locking, backends, team workflows)
- Module design patterns (composition vs inheritance)
- Multi-environment promotion strategies
- Security scanning in CI/CD (shift-left)
- Real project: TVS OTA Platform (Azure AKS + VNets + PostgreSQL + Blob)
