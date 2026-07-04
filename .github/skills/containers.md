# Containers & Orchestration

> **Certifications:** CKA / CKAD (Kubernetes)

| Tool | Level | Status |
|------|-------|--------|
| Docker | Expert | ⬜ Pending |
| Kubernetes (GKE, AKS, Talos, OKD) | Expert (Certified) | ✅ Complete (15 modules) |
| Helm | Advanced | ⬜ Pending |
| Talos Linux | Advanced | ⬜ Pending |
| OKD / OpenShift | Advanced | ⬜ Pending |

## Key Competencies

- Multi-stage Docker builds for embedded build environments
- Kubernetes cluster architecture across managed (GKE, AKS) and bare-metal (Talos Linux)
- OKD/OpenShift: Routes, SCCs, BuildConfigs, operator lifecycle
- Talos Linux: immutable OS, API-driven management, no SSH, declarative config
- RBAC, namespace isolation, resource governance, and admission control
- Helm chart development and release management
- Containerized CI/CT workloads (Android Goldfish Emulator pipelines)
- Pod security, network policies, and workload identity

## Notable Projects

| Project | Platform | Scale | Key Patterns |
|---------|----------|-------|--------------|
| **TVS Connected Vehicle & OTA** | Azure AKS (3 clusters) | Millions of vehicles | Multi-cluster, mTLS edge, MQTT IoT, multi-VNet isolation |
| Embedded CI/CT Platform | On-prem K8s / GKE | 100+ pipelines | Namespace isolation, HW-in-the-loop, resource governance |
| Platform Services (Jenkins, Gerrit) | AKS | Enterprise | HA, GitOps (Argo CD), centralized auth |
