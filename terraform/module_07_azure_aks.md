# Module 07: Azure AKS with Terraform

## 📖 Story: The Container Orchestration Factory

**English:**
Imagine you're running a massive logistics company. AKS is your central dispatch system — it decides which warehouse (node) handles which package (pod), routes deliveries (networking), manages worker shifts (scaling), and ensures if one warehouse burns down, packages get rerouted (HA). Node pools are specialized warehouse types — some for heavy goods (GPU), some for express delivery (high-CPU), some for storage (memory-optimized).

In our TVS OTA project, we ran 3 AKS clusters — OTA delivery, PKI services, and Edge management. Each cluster had separate node pools: system pool for Kubernetes components, MQTT pool for message brokers (memory-optimized), and workload pool for microservices. Workload Identity replaced service principals — pods directly authenticate to Azure services without secrets. This is THE most asked Azure interview topic.

**தமிழ்:**
நீங்கள் ஒரு பெரிய logistics company நடத்துவதாக கற்பனை செய்யுங்கள். AKS உங்கள் central dispatch system — எந்த warehouse (node) எந்த package (pod) handle செய்யும் என்று decide செய்கிறது, deliveries route செய்கிறது (networking), worker shifts manage செய்கிறது (scaling), ஒரு warehouse burn ஆனால் packages reroute செய்கிறது (HA). Node pools specialized warehouse types — heavy goods-க்கு (GPU), express delivery-க்கு (high-CPU), storage-க்கு (memory-optimized).

TVS OTA project-ல், 3 AKS clusters நடத்தினோம் — OTA delivery, PKI services, Edge management. ஒவ்வொரு cluster-லும் separate node pools: system pool (Kubernetes components), MQTT pool (message brokers, memory-optimized), workload pool (microservices). Workload Identity service principals-ஐ replace செய்தது — pods secrets இல்லாமல் நேரடியாக Azure services-ஐ authenticate செய்கின்றன.

---

## 📊 Architecture & Concepts

### AKS Cluster Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    AKS Cluster Architecture                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────┐                │
│  │      Control Plane (Microsoft-managed)   │                │
│  │  API Server │ etcd │ Scheduler │ CCM     │                │
│  └──────────────────────┬──────────────────┘                │
│                          │                                    │
│  ┌───────────────────────┼──────────────────────────────┐   │
│  │                 Node Pools                            │   │
│  │                                                       │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │   │
│  │  │ System Pool │  │ Workload Pool│  │ MQTT Pool  │  │   │
│  │  │ Standard_D2 │  │ Standard_D4  │  │ Standard_E4│  │   │
│  │  │ 3 nodes     │  │ 3-20 nodes   │  │ 3 nodes    │  │   │
│  │  │ CriticalOnly│  │ App workloads│  │ Memory-opt │  │   │
│  │  │ taint       │  │ Autoscaler   │  │ Dedicated  │  │   │
│  │  └─────────────┘  └──────────────┘  └────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Networking: Azure CNI │ Network Policy: Calico              │
│  Identity: Workload Identity │ Secrets: Key Vault CSI        │
│  Monitoring: Container Insights │ Logging: Log Analytics     │
└──────────────────────────────────────────────────────────────┘
```

### Networking Models Comparison

| Feature | Kubenet | Azure CNI | Azure CNI Overlay |
|---------|---------|-----------|-------------------|
| Pod IPs | NAT'd (node IP) | Real VNet IPs | Overlay network |
| Subnet size needed | Small (/24 OK) | Large (/21+) | Small (/24 OK) |
| Network Policy | Limited | Full (Calico/Azure) | Full |
| Performance | Good | Best (no NAT) | Good |
| Windows nodes | No | Yes | Yes |
| Max pods/node | 250 | 250 | 250 |
| Best for | Dev/small | Production | Large clusters |

**English:**
For production, use Azure CNI (pods get real VNet IPs — direct communication with Azure services, full network policy support). Azure CNI Overlay is newer — gives you CNI features without requiring huge subnets (pods get overlay IPs, nodes get VNet IPs).

**தமிழ்:**
Production-க்கு Azure CNI (pods-க்கு real VNet IPs — Azure services-உடன் direct communication, full network policy support). Azure CNI Overlay புதியது — huge subnets தேவையில்லாமல் CNI features கிடைக்கும் (pods overlay IPs, nodes VNet IPs).

### Node Pool Strategy

**English:**
Separate node pools by workload characteristics:

| Pool | Purpose | VM Size | Scaling | Taint |
|------|---------|---------|---------|-------|
| system | K8s system pods | D2s_v3 | Fixed 3 | `CriticalAddonsOnly` |
| workload | Application pods | D4s_v3 | Auto 3-20 | None |
| mqtt | Message brokers | E4s_v3 | Fixed 3 | `workload=mqtt:NoSchedule` |
| gpu | ML inference | NC6s_v3 | Auto 0-4 | `workload=gpu:NoSchedule` |
| spot | Batch/non-critical | D4s_v3 | Auto 0-10 | `kubernetes.azure.com/scalesetpriority=spot` |

**தமிழ்:**
Workload characteristics-படி separate node pools:
- **system**: K8s system pods — fixed 3 nodes, taint applied
- **workload**: Application pods — autoscaler 3-20 nodes
- **mqtt**: Message brokers — memory-optimized, dedicated
- **spot**: Batch jobs — 60-90% cheaper, can be evicted

### Workload Identity Flow

```
┌─────────┐    ┌───────────────┐    ┌────────────┐    ┌───────────┐
│   Pod   │───▶│ Service Acct  │───▶│ Federated  │───▶│  Azure    │
│(app.yaml│    │ (annotated    │    │ Identity   │    │  Key Vault│
│)        │    │  with client) │    │ Credential │    │  /Storage │
└─────────┘    └───────────────┘    └────────────┘    └───────────┘
                                          │
                                    OIDC Issuer
                                    (AKS managed)
```

**English:**
Workload Identity = Azure AD identity for pods (replaces pod-identity v1):
1. Create Azure Managed Identity
2. Create Kubernetes ServiceAccount (annotated with client ID)
3. Create Federated Credential (links K8s SA to Azure Identity)
4. Pod uses SA → gets Azure AD token → accesses Azure resources
5. NO secrets stored anywhere — zero credential management

**தமிழ்:**
Workload Identity = Pods-க்கான Azure AD identity:
1. Azure Managed Identity உருவாக்கு
2. Kubernetes ServiceAccount உருவாக்கு (client ID annotated)
3. Federated Credential உருவாக்கு (K8s SA-ஐ Azure Identity-உடன் link)
4. Pod SA-ஐ பயன்படுத்துகிறது → Azure AD token கிடைக்கும் → Azure resources access
5. Secrets எங்கும் store இல்லை — zero credential management

---

## 🧠 Byheart for Interview

### AKS Design Decisions
```
1. Private cluster (API server not publicly accessible)
2. Azure CNI for production (real VNet IPs, network policies)
3. Separate system + workload node pools (isolation)
4. Workload Identity for ALL Azure service authentication
5. Key Vault CSI driver for secrets (not K8s secrets)
6. Calico network policies for pod-to-pod security
7. Container Insights + Prometheus for observability
8. Cluster autoscaler for dynamic workloads
9. Azure Policy + OPA Gatekeeper for governance
10. Maintenance windows for controlled upgrades
```

### Critical Terraform Arguments
```
AKS Cluster:
  - private_cluster_enabled = true
  - sku_tier = "Standard" (SLA for production)
  - azure_policy_enabled = true
  - oidc_issuer_enabled = true (for Workload Identity)
  - workload_identity_enabled = true
  - network_profile.network_plugin = "azure"
  - network_profile.network_policy = "calico"

Node Pool:
  - enable_auto_scaling = true
  - only_critical_addons_enabled = true (system pool)
  - os_disk_type = "Ephemeral" (faster, cheaper)
  - node_taints for dedicated pools
  - zones = [1, 2, 3]
```

### AKS + Networking Integration
```
Inbound: Internet → AppGW/WAF → AKS Ingress → Service → Pod
Outbound: Pod → Azure Firewall (UDR) → Internet
Internal: Pod → Private Endpoint → PaaS (Blob/PostgreSQL/KeyVault)
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/azure-aks && cd ~/tf-lab/azure-aks
```

### Exercise 1: Production AKS Cluster

```hcl
cat > main.tf << 'EOF'
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {}
}

# ─── AKS CLUSTER ─────────────────────────────────
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-${var.project}-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = "aks-${var.project}-${var.environment}"
  kubernetes_version  = var.kubernetes_version

  # Production settings
  sku_tier                  = "Standard"         # SLA-backed (99.95%)
  private_cluster_enabled   = true               # API server not public
  azure_policy_enabled      = true               # Azure Policy integration
  oidc_issuer_enabled       = true               # For Workload Identity
  workload_identity_enabled = true               # Pod identity

  # ─── System Node Pool (only K8s components) ────
  default_node_pool {
    name                         = "system"
    vm_size                      = "Standard_D2s_v3"
    node_count                   = 3
    zones                        = [1, 2, 3]
    only_critical_addons_enabled = true  # Only system pods scheduled here
    os_disk_type                 = "Ephemeral"
    os_disk_size_gb              = 30
    vnet_subnet_id               = var.aks_subnet_id

    upgrade_settings {
      max_surge = "33%"
    }
  }

  # ─── Network Configuration ─────────────────────
  network_profile {
    network_plugin    = "azure"      # Azure CNI
    network_policy    = "calico"     # Network policies
    service_cidr      = "172.16.0.0/16"
    dns_service_ip    = "172.16.0.10"
    load_balancer_sku = "standard"

    load_balancer_profile {
      managed_outbound_ip_count = 2
    }
  }

  # ─── Identity ──────────────────────────────────
  identity {
    type = "SystemAssigned"
  }

  # ─── Monitoring ────────────────────────────────
  oms_agent {
    log_analytics_workspace_id = var.log_analytics_workspace_id
  }

  # ─── Key Vault CSI ────────────────────────────
  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    rotation_poll_interval   = "2m"
  }

  # ─── Maintenance Window ────────────────────────
  maintenance_window {
    allowed {
      day   = "Sunday"
      hours = [2, 3, 4]  # 2AM-4AM
    }
  }

  tags = var.tags
}

# ─── WORKLOAD NODE POOL (Application pods) ───────
resource "azurerm_kubernetes_cluster_node_pool" "workload" {
  name                  = "workload"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"
  zones                 = [1, 2, 3]
  os_disk_type          = "Ephemeral"
  os_disk_size_gb       = 128
  vnet_subnet_id        = var.aks_subnet_id

  # Autoscaling
  enable_auto_scaling = true
  min_count           = 3
  max_count           = 20
  node_count          = 3

  node_labels = {
    "role" = "workload"
  }

  upgrade_settings {
    max_surge = "33%"
  }

  tags = var.tags
}

# ─── MQTT NODE POOL (Message brokers) ────────────
resource "azurerm_kubernetes_cluster_node_pool" "mqtt" {
  name                  = "mqtt"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_E4s_v3"  # Memory-optimized
  zones                 = [1, 2, 3]
  os_disk_type          = "Ephemeral"
  os_disk_size_gb       = 64
  vnet_subnet_id        = var.aks_subnet_id

  enable_auto_scaling = false
  node_count          = 3

  node_labels = {
    "role" = "mqtt"
  }

  node_taints = [
    "workload=mqtt:NoSchedule"  # Only MQTT pods scheduled here
  ]

  tags = var.tags
}

# ─── SPOT NODE POOL (Non-critical/batch) ─────────
resource "azurerm_kubernetes_cluster_node_pool" "spot" {
  name                  = "spot"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D4s_v3"
  zones                 = [1, 2, 3]
  os_disk_type          = "Ephemeral"
  vnet_subnet_id        = var.aks_subnet_id

  priority        = "Spot"
  eviction_policy = "Delete"
  spot_max_price  = -1  # Pay up to on-demand price

  enable_auto_scaling = true
  min_count           = 0
  max_count           = 10
  node_count          = 0

  node_labels = {
    "kubernetes.azure.com/scalesetpriority" = "spot"
  }

  node_taints = [
    "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
  ]

  tags = var.tags
}
EOF
```

### Exercise 2: Workload Identity Setup

```hcl
cat > workload_identity.tf << 'EOF'
# ─── Managed Identity for the application ───────
resource "azurerm_user_assigned_identity" "app" {
  name                = "id-${var.project}-app-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
}

# ─── Federated Credential (links K8s SA → Azure Identity) ──
resource "azurerm_federated_identity_credential" "app" {
  name                = "fed-${var.project}-app"
  resource_group_name = var.resource_group_name
  parent_id           = azurerm_user_assigned_identity.app.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject             = "system:serviceaccount:${var.app_namespace}:${var.app_service_account}"
}

# ─── Grant Identity access to Key Vault ──────────
resource "azurerm_role_assignment" "keyvault_reader" {
  scope                = var.keyvault_id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.app.principal_id
}

# ─── Grant Identity access to Storage ────────────
resource "azurerm_role_assignment" "storage_reader" {
  scope                = var.storage_account_id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = azurerm_user_assigned_identity.app.principal_id
}
EOF
```

### Exercise 3: Monitoring & Diagnostics

```hcl
cat > monitoring.tf << 'EOF'
# ─── Log Analytics Workspace ─────────────────────
resource "azurerm_log_analytics_workspace" "aks" {
  name                = "log-aks-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# ─── Diagnostic Settings ─────────────────────────
resource "azurerm_monitor_diagnostic_setting" "aks" {
  name                       = "aks-diagnostics"
  target_resource_id         = azurerm_kubernetes_cluster.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.aks.id

  enabled_log {
    category = "kube-apiserver"
  }
  enabled_log {
    category = "kube-audit-admin"
  }
  enabled_log {
    category = "guard"
  }

  metric {
    category = "AllMetrics"
  }
}

# ─── Alert: Node CPU > 80% ──────────────────────
resource "azurerm_monitor_metric_alert" "node_cpu" {
  name                = "alert-aks-cpu-${var.environment}"
  resource_group_name = var.resource_group_name
  scopes              = [azurerm_kubernetes_cluster.main.id]
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"

  criteria {
    metric_namespace = "Insights.Container/nodes"
    metric_name      = "cpuUsagePercentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = var.action_group_id
  }
}
EOF
```

```bash
terraform init
terraform validate
terraform plan
```

---

## 🔥 Scenario Challenge

### Scenario: "Design production AKS for IoT platform serving 1M devices"

**Requirements:**
- 1 million connected devices sending telemetry every 30 seconds
- MQTT brokers handling 33K messages/second
- Firmware OTA delivery (large blob downloads)
- PKI certificate issuance and rotation
- Zero-downtime deployments
- Cost optimization (not all workloads are critical)

**Your Design Answer:**

```
┌─────────────────────────────────────────────────────────────┐
│ AKS Design for IoT Platform                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Cluster Configuration:                                      │
│ ├── Private cluster (no public API)                         │
│ ├── Azure CNI (real IPs for service mesh)                   │
│ ├── Calico network policies                                 │
│ ├── Kubernetes version: latest stable (1.28+)               │
│ └── SKU: Standard (SLA 99.95%)                             │
│                                                             │
│ Node Pools:                                                 │
│ ├── system: D2s_v3 × 3 (fixed, zone-spread)               │
│ ├── mqtt: E8s_v3 × 5 (memory-opt, dedicated taint)        │
│ ├── workload: D4s_v3 × 5-30 (autoscaler)                  │
│ ├── ota: D8s_v3 × 3-10 (high bandwidth for downloads)     │
│ └── spot: D4s_v3 × 0-10 (batch jobs, non-critical)        │
│                                                             │
│ Security:                                                   │
│ ├── Workload Identity for ALL service auth                  │
│ ├── Key Vault CSI for secrets                               │
│ ├── Azure Policy / OPA Gatekeeper                           │
│ ├── Network policies: default deny, explicit allow          │
│ └── Private Endpoints for all PaaS                          │
│                                                             │
│ Scaling Strategy:                                           │
│ ├── HPA: CPU/memory + custom metrics (queue depth)          │
│ ├── Cluster Autoscaler: node pool level                     │
│ ├── KEDA: Event-driven scaling for MQTT consumers           │
│ └── Spot pool: non-critical batch processing                │
│                                                             │
│ Availability:                                               │
│ ├── 3 AZ spread for all pools                              │
│ ├── PDB: minAvailable for critical services                 │
│ ├── Blue-green deployments via Argo Rollouts                │
│ └── Maintenance window: Sunday 2AM-4AM                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Real Project Reference: TVS Connected Vehicle Platform

**English:**
In TVS OTA platform (serving 500K+ connected vehicles):

**3-Cluster Architecture:**
1. **OTA Cluster**: Firmware delivery, campaign management, rollout orchestration
2. **PKI Cluster**: Certificate issuance, rotation, HSM integration (isolated for compliance)
3. **Edge Cluster**: Device shadow, MQTT broker, telemetry ingestion

**Key Design Decisions:**
- Separate MQTT node pool (E4s_v3, memory-optimized) — brokers need high memory for connection state
- Workload Identity for all service-to-Azure auth — zero stored credentials
- Key Vault CSI driver — PKI certificates injected as volumes, auto-rotated
- Private cluster + Private Endpoints — zero public surface
- KEDA for event-driven scaling — MQTT queue depth triggers pod scaling
- Spot nodes for batch firmware validation — 70% cost savings

**Results:**
- 99.99% uptime (multi-AZ + PDB + rolling updates)
- $45K/month total (vs $120K estimated without optimization)
- Zero security incidents (no public endpoints, no stored secrets)

**தமிழ்:**
TVS OTA platform (500K+ connected vehicles):

**3-Cluster Architecture:**
1. **OTA Cluster**: Firmware delivery, campaign management
2. **PKI Cluster**: Certificate issuance, rotation (compliance-க்கு isolated)
3. **Edge Cluster**: Device shadow, MQTT broker, telemetry ingestion

**Key Design Decisions:**
- Separate MQTT node pool (E4s_v3) — brokers-க்கு high memory தேவை
- Workload Identity — zero stored credentials
- Key Vault CSI driver — certificates auto-rotated
- Private cluster + Private Endpoints — zero public surface
- KEDA — MQTT queue depth triggers pod scaling
- Spot nodes batch firmware validation-க்கு — 70% cost savings

**Results:**
- 99.99% uptime
- $45K/month (vs $120K without optimization)
- Zero security incidents

---

## 🎤 Interview Q&A

### Q1: "Design a production AKS cluster — walk me through your decisions"

**English Answer:**
"I start with security: private cluster so API server isn't publicly accessible, Azure CNI for real VNet IPs enabling network policies, and Calico for pod-level segmentation. Node pools are separated by function — system pool with CriticalAddonsOnly taint ensures K8s components are isolated, workload pool has cluster autoscaler for dynamic scaling, and dedicated pools for stateful workloads like message brokers with appropriate VM sizes and taints. For identity, Workload Identity replaces all service principals — pods get Azure AD tokens via OIDC federation, zero secrets stored. Secrets come from Key Vault via CSI driver with auto-rotation. Monitoring via Container Insights + Prometheus, with alerts on node CPU, pod restarts, and OOM kills. Standard SKU for SLA backing, maintenance windows for controlled upgrades."

**தமிழ் Answer:**
"Security-லிருந்து start செய்வேன்: private cluster (API server public இல்லை), Azure CNI (real VNet IPs, network policies enable), Calico (pod-level segmentation). Node pools function-படி separate — system pool CriticalAddonsOnly taint-உடன், workload pool cluster autoscaler-உடன், stateful workloads-க்கு dedicated pools appropriate VM sizes + taints-உடன். Identity-க்கு Workload Identity — pods OIDC federation வழியாக Azure AD tokens கிடைக்கும், zero secrets. Key Vault CSI driver auto-rotation-உடன். Container Insights + Prometheus monitoring. Standard SKU SLA-க்கு, maintenance windows controlled upgrades-க்கு."

### Q2: "Kubenet vs Azure CNI — which do you choose and why?"

**English Answer:**
"Azure CNI for production, always. Three reasons: First, pods get real VNet IPs — they can directly communicate with Azure PaaS services via Private Endpoints without NAT. Second, full network policy support with Calico — I can implement zero-trust pod-to-pod security. Third, Windows node support if ever needed. The tradeoff is larger subnet requirements — I plan /21 for AKS subnets. For dev clusters where cost and IP space matter, Azure CNI Overlay is a good middle ground — gives CNI features with overlay networking so pods don't consume VNet IPs."

**தமிழ் Answer:**
"Production-க்கு Azure CNI, எப்போதும். மூன்று காரணங்கள்: ஒன்று, pods-க்கு real VNet IPs — Private Endpoints வழியாக Azure PaaS-உடன் NAT இல்லாமல் நேரடி communication. இரண்டு, Calico-உடன் full network policy support — zero-trust pod-to-pod security. மூன்று, Windows node support. Tradeoff: larger subnet requirements — AKS subnets-க்கு /21 plan செய்வேன். Dev clusters-க்கு Azure CNI Overlay நல்ல middle ground — VNet IPs consume செய்யாமல் CNI features."

### Q3: "How do you handle secrets in AKS?"

**English Answer:**
"Three layers, no Kubernetes secrets in etcd: First, Workload Identity — pods authenticate to Azure services directly via OIDC federation, no credentials stored. Second, Key Vault CSI driver — secrets are mounted as volumes into pods, auto-rotated every 2 minutes. Third, for non-Azure secrets, External Secrets Operator syncs from Key Vault to K8s secrets but encrypted at rest with customer-managed keys. The principle is: secrets never exist in Git, never in plain K8s secrets, always sourced from Key Vault with identity-based access."

**தமிழ் Answer:**
"மூன்று layers, Kubernetes secrets etcd-ல் இல்லை: ஒன்று, Workload Identity — pods OIDC federation வழியாக Azure services-ஐ directly authenticate, credentials store இல்லை. இரண்டு, Key Vault CSI driver — secrets pods-க்கு volumes-ஆக mount, 2 minutes-க்கு ஒரு முறை auto-rotated. மூன்று, non-Azure secrets-க்கு External Secrets Operator Key Vault-லிருந்து sync செய்யும். Principle: secrets Git-ல் இல்லை, plain K8s secrets-ல் இல்லை, எப்போதும் Key Vault-லிருந்து identity-based access."

### Q4: "How do you handle AKS upgrades without downtime?"

**English Answer:**
"Controlled upgrade strategy: First, maintenance windows restrict when upgrades can happen — we use Sunday 2-4AM. Second, max_surge=33% means 1/3 new nodes come up before old ones drain — always capacity available. Third, Pod Disruption Budgets ensure minimum replicas stay running during node drain. Fourth, I upgrade in sequence: system pool first (test K8s components), then workload pools one at a time. Fifth, we stay on N-1 version — never bleeding edge. In Terraform, I pin kubernetes_version and update it intentionally with a PR review process."

**தமிழ் Answer:**
"Controlled upgrade strategy: Maintenance windows upgrades-ஐ restrict செய்யும் — Sunday 2-4AM. max_surge=33%: 1/3 புதிய nodes ready ஆகும் before old ones drain — capacity எப்போதும் available. Pod Disruption Budgets minimum replicas running-ல் ensure செய்யும். Sequence: system pool முதலில் (K8s components test), பின் workload pools ஒவ்வொன்றாக. N-1 version-ல் இருப்போம் — bleeding edge இல்லை. Terraform-ல் kubernetes_version pin செய்து PR review process-உடன் update."

### Q5: "System pool vs user pool — why separate?"

**English Answer:**
"Isolation and reliability. System pool runs critical K8s components: CoreDNS, metrics-server, kube-proxy, CSI drivers. If a workload pod has a memory leak and takes down a node, it shouldn't affect DNS resolution for the entire cluster. With CriticalAddonsOnly taint on system pool, application pods can't schedule there. I use smaller VMs (D2s_v3) for system pool since components are lightweight, and right-sized VMs for workload pools based on application needs. Also, system pool is fixed-count (no autoscaler) — you never want CoreDNS scaling to zero."

**தமிழ் Answer:**
"Isolation மற்றும் reliability. System pool critical K8s components run செய்கிறது: CoreDNS, metrics-server, kube-proxy. ஒரு workload pod memory leak-ல் node-ஐ down செய்தால், cluster-ன் DNS resolution-ஐ affect செய்யக்கூடாது. CriticalAddonsOnly taint-உடன் application pods system pool-ல் schedule ஆகாது. System pool-க்கு சிறிய VMs (D2s_v3), workload pools-க்கு application needs-படி right-sized VMs. System pool fixed-count (autoscaler இல்லை) — CoreDNS zero-க்கு scale ஆகவே கூடாது."

### Q6: "How do you optimize AKS costs?"

**English Answer:**
"Five strategies: One, Spot node pools for fault-tolerant workloads — batch processing, non-critical jobs at 60-90% discount. Two, cluster autoscaler with proper min/max — scale to 3 nodes during off-hours, up to 20 during peak. Three, right-size node pools — don't use D8 for workloads that need D4. Four, resource requests and limits on all pods — prevents overprovisioning. Five, node pool consolidation — use KEDA to scale pods to zero for event-driven workloads, autoscaler removes empty nodes. In TVS, these strategies saved us $75K/month."

**தமிழ் Answer:**
"ஐந்து strategies: ஒன்று, Spot node pools fault-tolerant workloads-க்கு — 60-90% discount. இரண்டு, cluster autoscaler proper min/max-உடன் — off-hours 3 nodes, peak 20. மூன்று, right-size node pools — D4 போதுமானபோது D8 வேண்டாம். நான்கு, resource requests + limits அனைத்து pods-லும் — overprovisioning தடுக்கும். ஐந்து, KEDA event-driven workloads-க்கு pods zero-க்கு scale, autoscaler empty nodes remove செய்யும். TVS-ல் இவை $75K/month save செய்தன."

---

## ✅ Self-Check

- [ ] Can you design a production AKS cluster from scratch (private, CNI, node pools)?
- [ ] Can you explain Workload Identity flow (identity → federated credential → SA → pod)?
- [ ] Can you justify node pool separation (system/workload/mqtt/spot)?
- [ ] Can you explain Azure CNI vs Kubenet vs CNI Overlay with real tradeoffs?
- [ ] Can you describe Key Vault CSI driver integration?
- [ ] Can you explain AKS upgrade strategy (maintenance window, max_surge, PDB)?
- [ ] Can you design scaling strategy (HPA + cluster autoscaler + KEDA)?
- [ ] Can you explain the TVS 3-cluster architecture with reasoning?
- [ ] Can you discuss cost optimization strategies with numbers?
- [ ] Can you explain network policies (default deny + explicit allow)?
