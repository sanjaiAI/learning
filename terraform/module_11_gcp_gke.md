# Module 11: GCP GKE with Terraform

## 📖 Story: The Smart Warehouse

**English:**
If Compute Engine MIG is a factory assembly line (you manage the machines), **GKE** is a smart warehouse with robots. You tell it "I need 50 packages shipped" and it figures out how many robots (nodes), where to place them, and handles all the logistics. **Standard GKE** gives you control over the warehouse floor plan (node pools, machine types). **Autopilot GKE** is fully managed — Google even decides the floor plan, you just submit work orders (pods).

The key interview insight: AKS and GKE are architecturally similar (both Kubernetes), but differ in **identity integration**, **networking models**, and **management philosophy**. GKE Autopilot has no Azure equivalent — Azure's closest is AKS with Virtual Nodes, but it's not the same.

In EB, we used GKE Standard with **Workload Identity** (GCP's version of Azure AD Pod Identity) and **VPC-native networking** (alias IPs, like Azure CNI but native to GCP).

**தமிழ்:**
Compute Engine MIG ஒரு factory assembly line (machines-ஐ நீங்கள் manage செய்வீர்கள்) என்றால், **GKE** robots உள்ள smart warehouse. "50 packages ship செய்ய வேண்டும்" என்று சொன்னால், எத்தனை robots (nodes), எங்கே வைக்க வேண்டும், logistics எல்லாம் அது handle செய்யும். **Standard GKE** warehouse floor plan control கொடுக்கும் (node pools, machine types). **Autopilot GKE** fully managed — floor plan-ம் Google decide செய்யும், நீங்கள் work orders (pods) மட்டும் submit செய்யுங்கள்.

முக்கிய interview insight: AKS, GKE architecturally similar (இரண்டும் Kubernetes), ஆனால் **identity integration**, **networking models**, **management philosophy**-ல் differ ஆகும். GKE Autopilot-க்கு Azure equivalent இல்லை.

EB-ல், **Workload Identity** (Azure AD Pod Identity-ன் GCP version) மற்றும் **VPC-native networking** (alias IPs, Azure CNI போன்றது) உடன் GKE Standard பயன்படுத்தினோம்.

---

## 📊 Architecture & Concepts

### GKE Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                     GKE CLUSTER                                    │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              CONTROL PLANE (Google-managed)                  │  │
│  │  ┌────────┐  ┌──────────┐  ┌─────────┐  ┌────────────┐   │  │
│  │  │ API    │  │ Scheduler│  │  etcd   │  │ Controller │   │  │
│  │  │ Server │  │          │  │         │  │  Manager   │   │  │
│  │  └────────┘  └──────────┘  └─────────┘  └────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│  ┌───────────────────────────▼────────────────────────────────┐  │
│  │                    NODE POOLS                                │  │
│  │                                                             │  │
│  │  ┌─────────────────┐    ┌─────────────────┐               │  │
│  │  │ Pool: system     │    │ Pool: ci-agents  │               │  │
│  │  │ e2-standard-4   │    │ e2-standard-8   │               │  │
│  │  │ 3 nodes (on-dem)│    │ 0-20 (spot)     │               │  │
│  │  │ taints: system   │    │ taints: ci-only  │               │  │
│  │  └─────────────────┘    └─────────────────┘               │  │
│  │                                                             │  │
│  │  ┌─────────────────┐                                       │  │
│  │  │ Pool: gpu-ml     │                                       │  │
│  │  │ a2-highgpu-1g   │                                       │  │
│  │  │ 0-5 (spot)      │                                       │  │
│  │  │ taints: gpu-only │                                       │  │
│  │  └─────────────────┘                                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  Networking: VPC-native (alias IPs)                               │
│  Identity: Workload Identity                                       │
│  Security: Private cluster + Authorized Networks                   │
└──────────────────────────────────────────────────────────────────┘
```

### AKS vs GKE — Complete Comparison

| Feature | Azure AKS | GCP GKE |
|---------|-----------|---------|
| **Managed control plane** | Free (API server) | Free (Standard), charged (Autopilot compute) |
| **Cluster modes** | Standard only | Standard + **Autopilot** |
| **Node pools** | System + User pools | Default + Custom pools |
| **Networking** | Azure CNI / Kubenet | VPC-native (alias IP) / Routes-based |
| **Pod IP model** | Azure CNI: IPs from subnet | Alias IPs from secondary ranges |
| **Identity** | Azure AD + Workload Identity | Google IAM + Workload Identity |
| **Pod-to-GCP auth** | Workload Identity Federation | Workload Identity (KSA→GSA mapping) |
| **Private cluster** | Private cluster (API + nodes) | Private cluster (API + nodes) |
| **Ingress** | AGIC / nginx | GKE Ingress (Cloud LB) / Gateway API |
| **Service mesh** | Azure Service Mesh (Istio) | Anthos Service Mesh (Istio) |
| **Autoscaling** | Cluster Autoscaler + KEDA | Cluster Autoscaler + HPA/VPA |
| **Spot nodes** | Spot node pools | Spot node pools (preemptible) |
| **OS** | Ubuntu / Azure Linux / Windows | Container-Optimized OS (COS) / Ubuntu |
| **Upgrades** | Manual + Auto-upgrade channels | Release channels (Rapid/Regular/Stable) |
| **Max pods/node** | 250 (Azure CNI) | 110 (default), 256 (max) |
| **Multi-cluster** | Azure Fleet Manager | Anthos / GKE Fleet |
| **Cost** | Free control plane | Free control plane (Standard) |

### Standard vs Autopilot

**English:**
| Aspect | GKE Standard | GKE Autopilot |
|--------|-------------|---------------|
| Node management | You manage node pools | Google manages nodes |
| Pricing | Per node (VM cost) | Per pod (resource requests) |
| SSH to nodes | Yes | No |
| DaemonSets | Full control | Limited (Google-approved only) |
| Privileged pods | Allowed | Not allowed |
| GPU | Full support | Limited support |
| Best for | Custom requirements, cost control | Simple workloads, hands-off |

**Interview answer**: "Standard for enterprise workloads with custom requirements (privileged containers, specific machine types, DaemonSets). Autopilot for simpler workloads where you want Google to optimize — no node pool management, pay-per-pod."

**தமிழ்:**
**Interview answer**: "Standard — enterprise workloads, custom requirements (privileged containers, specific machine types, DaemonSets). Autopilot — simpler workloads, Google optimize செய்ய வேண்டும், node pool management வேண்டாம், pay-per-pod."

### Workload Identity (Critical for Interview)

**English:**
Workload Identity is GCP's secure way for pods to authenticate to GCP services. Instead of storing service account keys in secrets (insecure!), you create a mapping:

```
Kubernetes ServiceAccount (KSA) ←→ GCP Service Account (GSA)
```

Pod uses KSA → GKE automatically provides GSA credentials → Pod accesses GCP APIs.

Azure equivalent: **AKS Workload Identity Federation** (formerly AAD Pod Identity).

**தமிழ்:**
Workload Identity — pods GCP services-ஐ authenticate செய்ய GCP-ன் secure way. Service account keys secrets-ல் store செய்வதற்கு பதிலாக (insecure!), mapping create:

```
Kubernetes ServiceAccount (KSA) ←→ GCP Service Account (GSA)
```

Pod KSA use → GKE automatically GSA credentials provide → Pod GCP APIs access.

Azure equivalent: **AKS Workload Identity Federation** (முன்பு AAD Pod Identity).

### VPC-Native Networking

**English:**
GKE VPC-native uses **alias IP ranges** from the subnet's secondary ranges:
- Node IPs: From primary subnet CIDR
- Pod IPs: From secondary range #1 (e.g., /16 = 65K pods)
- Service IPs: From secondary range #2 (e.g., /20 = 4K services)

This is conceptually similar to Azure CNI where pods get IPs from the VNet subnet, but GKE uses secondary ranges to keep pod IPs separate from node IPs.

**தமிழ்:**
GKE VPC-native subnet-ன் secondary ranges-லிருந்து **alias IP ranges** பயன்படுத்தும்:
- Node IPs: Primary subnet CIDR-லிருந்து
- Pod IPs: Secondary range #1-லிருந்து (e.g., /16 = 65K pods)
- Service IPs: Secondary range #2-லிருந்து (e.g., /20 = 4K services)

Azure CNI போன்றது — ஆனால் GKE secondary ranges பயன்படுத்தி pod IPs-ஐ node IPs-லிருந்து separate ஆக வைக்கும்.

---

## 🧠 Byheart for Interview

### GKE Key Facts
```
1. Standard = you manage nodes; Autopilot = Google manages nodes (pay-per-pod)
2. VPC-native (alias IPs) is DEFAULT and RECOMMENDED (routes-based is legacy)
3. Workload Identity: KSA → GSA mapping (NEVER use node SA or key files)
4. Private cluster: no public IP on nodes + optional private API endpoint
5. Release channels: Rapid (latest) → Regular (default) → Stable (conservative)
6. Node auto-repair: detects unhealthy nodes, drains and recreates
7. Node auto-upgrade: applies patches within release channel
8. Binary Authorization: only deploy signed container images
9. Shielded GKE Nodes: secure boot + integrity monitoring
10. GKE Dataplane V2 (Cilium): eBPF-based networking (replaces iptables)
```

### IP Planning for GKE
```
Subnet primary:    /20 (4096 IPs) → up to 4096 nodes
Pod secondary:     /14 (262144 IPs) → 110 pods/node × 2000+ nodes
Service secondary: /20 (4096 IPs) → 4096 services

Formula: nodes × max_pods_per_node = total pod IPs needed
Example: 100 nodes × 110 pods = 11000 → need /18 minimum for pods
```

### Interview Golden Answers
```
Q: "AKS vs GKE — when would you choose each?"
A: "Choose AKS when: Azure-heavy environment, tight Azure AD integration, 
    Windows containers needed, Azure DevOps CI/CD, existing Azure networking.
    Choose GKE when: need Autopilot (no node management), multi-cloud Anthos 
    strategy, Istio service mesh (first-class support), advanced networking 
    (Dataplane V2/Cilium), AI/ML workloads (TPU support).
    Both are excellent — the choice is ecosystem, not technology."

Q: "How does Workload Identity differ between AKS and GKE?"
A: "Same concept, different implementation:
    - GKE: KSA annotated with GSA email, GSA has IAM binding to KSA
    - AKS: Federated identity credential on Azure AD app, annotated on KSA
    Both eliminate the need for secret keys. GKE's is slightly simpler because 
    it's native to the platform — AKS Workload Identity requires OIDC issuer setup."

Q: "Design a production GKE cluster for CI/CD workloads"
A: "Private cluster with authorized networks, VPC-native networking, 
    3 node pools (system/ci-build/ci-test), spot nodes for ci pools, 
    Workload Identity for accessing Artifact Registry and Secret Manager, 
    Binary Authorization for supply chain security, release channel = Regular, 
    GKE Dataplane V2 for network policy enforcement."
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/gcp-gke && cd ~/tf-lab/gcp-gke
```

### Exercise 1: GKE Standard Private Cluster

```hcl
cat > main.tf << 'EOF'
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# ─── GKE CLUSTER (Private, VPC-native) ────────────────────────────
resource "google_container_cluster" "main" {
  name     = "gke-${var.environment}"
  location = var.region  # Regional cluster (multi-zone HA)

  # Remove default node pool (we'll create custom ones)
  remove_default_node_pool = true
  initial_node_count       = 1

  # ─── NETWORKING ──────────────────────────────────────────────
  network    = var.vpc_id
  subnetwork = var.subnet_id

  # VPC-native (alias IPs) — ALWAYS use this
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"      # Secondary range name
    services_secondary_range_name = "services"  # Secondary range name
  }

  # Private cluster — no public IPs on nodes
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false  # true = kubectl only from VPC
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  # Authorized networks for kubectl access
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.admin_cidr
      display_name = "Admin network"
    }
  }

  # ─── SECURITY ───────────────────────────────────────────────
  # Workload Identity (critical!)
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Shielded nodes
  node_config {
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }

  # Binary Authorization
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  # ─── MAINTENANCE & UPGRADES ─────────────────────────────────
  release_channel {
    channel = "REGULAR"  # Balanced between features and stability
  }

  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"  # Weekends only
    }
  }

  # ─── ADDONS ─────────────────────────────────────────────────
  addons_config {
    http_load_balancing {
      disabled = false  # Enable GKE Ingress
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
    gce_persistent_disk_csi_driver_config {
      enabled = true
    }
  }

  # Dataplane V2 (Cilium-based, eBPF networking)
  datapath_provider = "ADVANCED_DATAPATH"

  # Logging and monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    managed_prometheus {
      enabled = true
    }
  }
}
EOF
```

### Exercise 2: Node Pools (System + CI)

```hcl
cat > node_pools.tf << 'EOF'
# ─── SYSTEM NODE POOL (always-on, on-demand) ──────────────────────
resource "google_container_node_pool" "system" {
  name     = "system"
  cluster  = google_container_cluster.main.id
  location = var.region

  node_count = 2

  node_config {
    machine_type = "e2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-ssd"

    # Container-Optimized OS (hardened, minimal)
    image_type = "COS_CONTAINERD"

    # Workload Identity at node level
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # Shielded nodes
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    # OAuth scopes (minimal — Workload Identity handles the rest)
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]

    labels = {
      role = "system"
    }

    # Taint: only system pods run here
    taint {
      key    = "dedicated"
      value  = "system"
      effect = "NO_SCHEDULE"
    }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# ─── CI AGENT NODE POOL (spot, autoscaling) ───────────────────────
resource "google_container_node_pool" "ci_agents" {
  name     = "ci-agents"
  cluster  = google_container_cluster.main.id
  location = var.region

  # Autoscaling: 0 to 20 nodes
  autoscaling {
    min_node_count = 0
    max_node_count = 20
  }

  node_config {
    machine_type = "e2-standard-8"
    disk_size_gb = 200
    disk_type    = "pd-ssd"
    image_type   = "COS_CONTAINERD"

    # SPOT VMs for cost savings
    spot = true

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]

    labels = {
      role = "ci-agent"
    }

    taint {
      key    = "dedicated"
      value  = "ci"
      effect = "NO_SCHEDULE"
    }

    # Network tags for firewall rules
    tags = ["gke-ci-agent"]
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
EOF
```

### Exercise 3: Workload Identity Setup

```hcl
cat > workload_identity.tf << 'EOF'
# ─── GCP Service Account for CI workloads ─────────────────────────
resource "google_service_account" "ci_workload" {
  account_id   = "gke-ci-workload"
  display_name = "GKE CI Workload Identity"
}

# Grant permissions to the GSA
resource "google_project_iam_member" "ci_artifact_reader" {
  project = var.project_id
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.ci_workload.email}"
}

resource "google_project_iam_member" "ci_storage_viewer" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.ci_workload.email}"
}

# ─── BIND KSA to GSA (Workload Identity) ──────────────────────────
# This allows pods using the KSA to act as the GSA
resource "google_service_account_iam_binding" "ci_workload_identity" {
  service_account_id = google_service_account.ci_workload.name
  role               = "roles/iam.workloadIdentityUser"

  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[ci/ci-agent-sa]"
    # Format: serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]
  ]
}

# ─── Kubernetes ServiceAccount (annotated) ─────────────────────────
# Note: In practice, apply this via kubectl or Helm
# The annotation tells GKE which GSA to use
#
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   name: ci-agent-sa
#   namespace: ci
#   annotations:
#     iam.gke.io/gcp-service-account: gke-ci-workload@PROJECT.iam.gserviceaccount.com
EOF
```

### Exercise 4: Variables

```hcl
cat > variables.tf << 'EOF'
variable "project_id" {
  type = string
}

variable "region" {
  type    = string
  default = "us-central1"
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "vpc_id" {
  type        = string
  description = "VPC network ID"
}

variable "subnet_id" {
  type        = string
  description = "Subnet ID with secondary ranges for pods/services"
}

variable "admin_cidr" {
  type        = string
  description = "CIDR for kubectl access"
  default     = "10.0.0.0/8"
}
EOF
```

```bash
terraform init
terraform validate
terraform plan -var="project_id=my-project" -var="vpc_id=projects/my-project/global/networks/vpc-dev" -var="subnet_id=projects/my-project/regions/us-central1/subnetworks/subnet-gke"
```

---

## 🔥 Scenario Challenge

### Scenario: Production GKE for Mixed Workloads

**English:**
Design a GKE cluster that supports:
1. Always-on microservices (API gateway, backend services)
2. CI/CD build agents (bursty, cost-sensitive)
3. ML training jobs (GPU, batch)
4. Must be secure (private, Workload Identity, Binary Auth)
5. Multi-zone HA within a region

**தமிழ்:**
GKE cluster design செய்யுங்கள்:
1. Always-on microservices (API gateway, backend services)
2. CI/CD build agents (bursty, cost-sensitive)
3. ML training jobs (GPU, batch)
4. Secure (private, Workload Identity, Binary Auth)
5. Region-க்குள் multi-zone HA

### Solution Architecture

```hcl
resource "google_container_cluster" "production" {
  name     = "gke-prod"
  location = "us-central1"  # Regional = multi-zone HA

  remove_default_node_pool = true
  initial_node_count       = 1

  # Private + VPC-native
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = true  # No public API endpoint
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  datapath_provider = "ADVANCED_DATAPATH"  # Cilium
  release_channel { channel = "REGULAR" }
}

# Pool 1: System (ingress controllers, monitoring)
resource "google_container_node_pool" "system" {
  name       = "system"
  cluster    = google_container_cluster.production.id
  node_count = 3  # Fixed, HA

  node_config {
    machine_type = "e2-standard-4"
    spot         = false  # On-demand for reliability
    taint {
      key = "dedicated"; value = "system"; effect = "NO_SCHEDULE"
    }
  }
}

# Pool 2: Application (microservices)
resource "google_container_node_pool" "app" {
  name    = "application"
  cluster = google_container_cluster.production.id

  autoscaling {
    min_node_count = 3
    max_node_count = 30
  }

  node_config {
    machine_type = "n2-standard-8"
    spot         = false  # On-demand for SLA
    labels       = { role = "app" }
  }
}

# Pool 3: CI Agents (spot, scale to zero)
resource "google_container_node_pool" "ci" {
  name    = "ci-agents"
  cluster = google_container_cluster.production.id

  autoscaling {
    min_node_count = 0
    max_node_count = 50
  }

  node_config {
    machine_type = "e2-standard-8"
    spot         = true  # Spot for cost savings
    taint {
      key = "dedicated"; value = "ci"; effect = "NO_SCHEDULE"
    }
    labels = { role = "ci" }
  }
}

# Pool 4: GPU (ML training, spot)
resource "google_container_node_pool" "gpu" {
  name    = "gpu-ml"
  cluster = google_container_cluster.production.id

  autoscaling {
    min_node_count = 0
    max_node_count = 5
  }

  node_config {
    machine_type = "a2-highgpu-1g"
    spot         = true

    guest_accelerator {
      type  = "nvidia-tesla-a100"
      count = 1
    }

    taint {
      key = "nvidia.com/gpu"; value = "present"; effect = "NO_SCHEDULE"
    }
    labels = { role = "gpu" }
  }
}
```

---

## 🏗️ Real Project Reference (EB)

**English:**
GKE in EB's infrastructure:
- **Cluster type**: Standard (needed DaemonSets for log collection, privileged containers for Docker-in-Docker builds)
- **Node pools**: system (3 on-demand) + ci-build (0-30 spot) + ci-test (0-10 spot)
- **Workload Identity**: CI pods access Artifact Registry and Secret Manager without keys
- **Private cluster**: No public endpoint, kubectl via IAP tunnel
- **VPC-native**: Secondary ranges /16 for pods, /20 for services
- **Binary Authorization**: Only images from our Artifact Registry allowed
- **Cost strategy**: Spot node pools for CI (70% savings), scale-to-zero off-hours

Why not Autopilot? Needed privileged containers for Docker builds and custom DaemonSets for Datadog.

**தமிழ்:**
EB infrastructure-ல் GKE:
- **Cluster type**: Standard (DaemonSets, Docker-in-Docker builds-க்கு privileged containers தேவை)
- **Node pools**: system (3 on-demand) + ci-build (0-30 spot) + ci-test (0-10 spot)
- **Workload Identity**: CI pods keys இல்லாமல் Artifact Registry, Secret Manager access
- **Private cluster**: Public endpoint இல்லை, IAP tunnel வழியாக kubectl
- **Cost strategy**: CI-க்கு Spot node pools (70% savings), off-hours scale-to-zero

Autopilot ஏன் இல்லை? Docker builds-க்கு privileged containers, Datadog-க்கு custom DaemonSets தேவை.

---

## 🎤 Interview Q&A

### Q1: "AKS vs GKE — when would you choose each?"

**English:**
"The choice is primarily about ecosystem fit:

**Choose AKS when:**
- Organization is Azure-heavy (Active Directory, Azure DevOps, Key Vault)
- Windows container workloads needed
- Tight integration with Azure Monitor, Defender for Cloud
- Virtual Nodes (ACI) for serverless burst

**Choose GKE when:**
- Need Autopilot (true serverless Kubernetes — no node management)
- Multi-cloud strategy with Anthos
- Advanced networking needs (Dataplane V2/Cilium, native Network Policy)
- AI/ML workloads (TPU support, better GPU ecosystem)
- Want managed Istio (Anthos Service Mesh)

Both are Kubernetes underneath — apps are portable. The lock-in is in the ecosystem services (identity, monitoring, networking), not the workloads."

**தமிழ்:**
"Choice primarily ecosystem fit பற்றியது:

**AKS choose**: Azure-heavy organization, Windows containers, Azure DevOps integration.
**GKE choose**: Autopilot (true serverless K8s), multi-cloud Anthos, Dataplane V2/Cilium, AI/ML (TPU), managed Istio.

இரண்டும் Kubernetes — apps portable. Lock-in ecosystem services-ல் (identity, monitoring), workloads-ல் அல்ல."

### Q2: "Explain Workload Identity in GKE and how it compares to AKS"

**English:**
"Workload Identity eliminates service account keys by creating a trust binding:

**GKE Workload Identity:**
1. Enable `workload_pool` on cluster
2. Create GCP Service Account (GSA) with needed permissions
3. Create Kubernetes ServiceAccount (KSA) with annotation pointing to GSA
4. Add IAM binding: GSA trusts `PROJECT.svc.id.goog[NAMESPACE/KSA]`
5. Pods using KSA automatically get GSA credentials

**AKS Workload Identity:**
1. Enable OIDC issuer on cluster
2. Create Azure AD App Registration / Managed Identity
3. Create federated identity credential (trust OIDC issuer + subject)
4. Annotate KSA with client-id
5. Pod gets federated token via projected volume

Same concept, GKE's implementation is slightly simpler — one annotation + one IAM binding."

**தமிழ்:**
"Workload Identity service account keys-ஐ eliminate செய்யும்:

**GKE**: Cluster-ல் workload_pool enable → GSA create → KSA annotate → IAM binding
**AKS**: OIDC issuer enable → Azure AD App → Federated credential → KSA annotate

Same concept, GKE implementation சற்று simpler — ஒரு annotation + ஒரு IAM binding."

### Q3: "GKE Standard vs Autopilot — decision criteria?"

**English:**
"**Standard** when you need:
- DaemonSets (custom log collectors, security agents)
- Privileged containers (Docker-in-Docker)
- Specific machine types or GPUs
- SSH access to nodes for debugging
- Full control over node pool sizing and cost

**Autopilot** when:
- You want zero node management (Google handles everything)
- Pay-per-pod pricing (no idle node waste)
- Security hardening is critical (no SSH, no privileged pods = smaller attack surface)
- Simple workloads that don't need low-level access

In our EB CI pipeline, we chose Standard because Docker-in-Docker builds require privileged pods, and we needed custom DaemonSets for Datadog. But for a standard web API platform, I'd recommend Autopilot."

**தமிழ்:**
"**Standard**: DaemonSets, privileged containers, specific machine types, node SSH, full control.
**Autopilot**: Zero node management, pay-per-pod, security hardening, simple workloads.

EB-ல் Standard choose செய்தோம் — Docker-in-Docker privileged pods தேவை, Datadog-க்கு custom DaemonSets. Standard web API platform-க்கு Autopilot recommend செய்வேன்."

### Q4: "How do you secure a GKE cluster?"

**English:**
"Seven-layer security approach:
1. **Private cluster**: No public IPs on nodes, optional private API endpoint
2. **Authorized networks**: Restrict kubectl access to specific CIDRs
3. **Workload Identity**: No service account keys ever
4. **Binary Authorization**: Only signed images from trusted registries
5. **Shielded nodes**: Secure boot + integrity monitoring
6. **Network Policy (Dataplane V2)**: Pod-to-pod traffic control
7. **RBAC + Namespace isolation**: Least-privilege per team

Azure equivalent mapping:
- Private cluster = AKS Private Cluster
- Authorized networks = AKS Authorized IP Ranges
- Binary Authorization ≈ Azure Policy for AKS + Notary
- Shielded nodes ≈ Confidential VMs on AKS"

**தமிழ்:**
"ஏழு-layer security:
1. Private cluster, 2. Authorized networks, 3. Workload Identity,
4. Binary Authorization, 5. Shielded nodes, 6. Network Policy (Cilium),
7. RBAC + Namespace isolation

Azure equivalent: Private cluster = AKS Private, Binary Auth ≈ Azure Policy + Notary"

---

## ✅ Self-Check

| # | Question | Can You Answer? |
|---|----------|-----------------|
| 1 | GKE Standard vs Autopilot — 3 key differences? | ☐ |
| 2 | Workload Identity setup steps (GKE vs AKS)? | ☐ |
| 3 | VPC-native networking — why and how? | ☐ |
| 4 | Private cluster + authorized networks config? | ☐ |
| 5 | Node pool strategy for mixed workloads? | ☐ |
| 6 | Spot node pools — Terraform config? | ☐ |
| 7 | Binary Authorization purpose and setup? | ☐ |
| 8 | IP planning for GKE (pods, services, nodes)? | ☐ |
| 9 | AKS vs GKE — when to choose each? | ☐ |
| 10 | GKE Dataplane V2 — what and why? | ☐ |

---

*Next: [Module 12 — GCP Storage & Data](module_12_gcp_data.md)*
