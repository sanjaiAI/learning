# Module 11: GCP GKE
# மாடுல் 11: GCP GKE (Google Kubernetes Engine)

---

## 🎯 What? | என்ன?

**English:** Provision production-grade GKE clusters — private clusters, node pools, workload identity, and Autopilot mode.

**தமிழ்:** Production-grade GKE clusters provision — private clusters, node pools, workload identity, Autopilot.

---

## 🛠️ Production GKE Cluster

```hcl
resource "google_container_cluster" "main" {
  name     = "gke-${var.project}-${var.environment}"
  location = var.region    # Regional cluster (HA across 3 zones)

  # Remove default node pool (we'll create custom ones)
  remove_default_node_pool = true
  initial_node_count       = 1

  # Networking
  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.gke.id

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  # Private cluster (nodes have no public IPs)
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false    # API server still reachable
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  # Master authorized networks (who can reach API server)
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.admin_cidr
      display_name = "Admin VPN"
    }
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.gcp_project}.svc.id.goog"
  }

  # Security
  release_channel {
    channel = "REGULAR"    # Auto-upgrade within channel
  }

  # Binary Authorization (only signed images!)
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }

  # Logging & Monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    managed_prometheus { enabled = true }
  }

  # Maintenance window
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T02:00:00Z"
      end_time   = "2024-01-01T06:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }
}
```

---

## 🛠️ Node Pools

```hcl
resource "google_container_node_pool" "system" {
  name       = "system"
  cluster    = google_container_cluster.main.id
  node_count = 2

  node_config {
    machine_type = "e2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-ssd"

    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"    # Workload Identity enabled
    }

    labels = {
      nodepool = "system"
    }

    taint {
      key    = "CriticalAddonsOnly"
      value  = "true"
      effect = "PREFER_NO_SCHEDULE"
    }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

resource "google_container_node_pool" "workload" {
  name    = "workload"
  cluster = google_container_cluster.main.id

  autoscaling {
    min_node_count = 2
    max_node_count = 20
  }

  node_config {
    machine_type = "e2-standard-8"
    disk_size_gb = 200
    disk_type    = "pd-ssd"
    spot         = var.environment != "prod"    # Spot for non-prod!

    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    labels = {
      nodepool = "workload"
    }
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

---

## 🛠️ Workload Identity (GKE → GCP Services)

```hcl
# GCP Service Account
resource "google_service_account" "app" {
  account_id   = "sa-app-${var.environment}"
  display_name = "Application Service Account"
}

# Grant SA access to Cloud SQL
resource "google_project_iam_member" "app_sql" {
  project = var.gcp_project
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.app.email}"
}

# Bind K8s SA → GCP SA (Workload Identity)
resource "google_service_account_iam_binding" "workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "serviceAccount:${var.gcp_project}.svc.id.goog[production/app-sa]"
  ]
}

# K8s ServiceAccount annotation (apply with kubectl/helm)
# metadata:
#   annotations:
#     iam.gke.io/gcp-service-account: sa-app-prod@project.iam.gserviceaccount.com
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         GCP GKE TERRAFORM CHEAT SHEET            │
├──────────────────────────────────────────────────┤
│ CLUSTER TYPES:                                   │
│   Regional  = HA (3 zones, recommended)          │
│   Zonal     = single zone (dev only)             │
│   Autopilot = fully managed (Google manages nodes)│
│                                                  │
│ NETWORKING:                                      │
│   VPC-native (alias IP) — always use             │
│   Secondary ranges: pods + services              │
│   Private cluster: nodes no public IP            │
│   Cloud NAT: outbound for private nodes          │
│                                                  │
│ SECURITY:                                        │
│   Workload Identity (K8s SA → GCP SA)            │
│   Binary Authorization (signed images only)      │
│   Shielded Nodes (secure boot)                   │
│   Master authorized networks                     │
│                                                  │
│ AKS vs GKE:                                      │
│   AKS: Azure CNI, Calico, SystemAssigned ID      │
│   GKE: VPC-native, Dataplane V2, Workload ID    │
│   Both: node pools, autoscaling, private mode    │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: AKS vs GKE — architectural differences?**
- Networking: AKS uses Azure CNI (pod gets VNet IP), GKE uses VPC-native with secondary ranges
- Identity: AKS Workload Identity (federated credential), GKE Workload Identity (SA binding)
- Policy: AKS Calico, GKE Dataplane V2 (Cilium-based)
- Management: AKS manual version pin, GKE release channels (auto-upgrade)

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Private GKE cluster provision முடியும்
- [ ] Node pools with autoscaling configure முடியும்
- [ ] Workload Identity setup முடியும்
- [ ] AKS vs GKE differences explain முடியும்
