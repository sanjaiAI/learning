# Module 09: GCP Networking
# மாடுல் 09: GCP Networking

---

## 🎯 What? | என்ன?

**English:** Provision GCP networking with Terraform — VPCs, subnets, firewall rules, Cloud NAT, Cloud Router. GCP networking is global (unlike Azure's regional VNets).

**தமிழ்:** GCP networking-ஐ Terraform-ல் provision — VPCs, subnets, firewall rules, Cloud NAT. GCP networking global (Azure-ன் regional VNets-இல் இருந்து வேறுபட்டது).

---

## ⚔️ Azure vs GCP Networking

| Concept | Azure | GCP |
|---------|-------|-----|
| Network | VNet (regional) | VPC (global!) |
| Subnet | Regional | Regional |
| Firewall | NSG (per subnet/NIC) | Firewall rules (per VPC, tag-based) |
| Peering | VNet Peering | VPC Peering |
| NAT | NAT Gateway | Cloud NAT (per region) |
| Private access | Private Endpoint | Private Service Connect / Private Google Access |

---

## 🛠️ VPC & Subnets

```hcl
# VPC (Global — subnets are regional)
resource "google_compute_network" "main" {
  name                    = "vpc-${var.project}-${var.environment}"
  auto_create_subnetworks = false    # Custom mode (always use this!)
  routing_mode            = "GLOBAL"
}

# Subnets (regional)
resource "google_compute_subnetwork" "gke" {
  name          = "snet-gke-${var.environment}"
  ip_cidr_range = "10.10.0.0/20"     # Primary range for nodes
  region        = var.region
  network       = google_compute_network.main.id

  # Secondary ranges for GKE pods and services
  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.20.0.0/14"    # /14 = 262k pod IPs
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.24.0.0/20"    # /20 = 4k service IPs
  }

  private_ip_google_access = true     # Access Google APIs without public IP
}

resource "google_compute_subnetwork" "data" {
  name          = "snet-data-${var.environment}"
  ip_cidr_range = "10.10.16.0/20"
  region        = var.region
  network       = google_compute_network.main.id

  private_ip_google_access = true
}
```

---

## 🛠️ Firewall Rules

```hcl
# Allow internal communication
resource "google_compute_firewall" "allow_internal" {
  name    = "fw-allow-internal"
  network = google_compute_network.main.id

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "icmp"
  }

  source_ranges = ["10.10.0.0/16"]    # All internal subnets
}

# Allow SSH from IAP only (no direct SSH from internet!)
resource "google_compute_firewall" "allow_iap_ssh" {
  name    = "fw-allow-iap-ssh"
  network = google_compute_network.main.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"]   # Google IAP range
  target_tags   = ["allow-ssh"]          # Only VMs with this tag
}

# Allow health checks (for load balancers)
resource "google_compute_firewall" "allow_health_check" {
  name    = "fw-allow-health-check"
  network = google_compute_network.main.id

  allow {
    protocol = "tcp"
  }

  source_ranges = ["130.211.0.0/22", "35.191.0.0/16"]  # Google LB ranges
  target_tags   = ["gke-node"]
}
```

---

## 🛠️ Cloud NAT (Outbound internet for private nodes)

```hcl
# Cloud Router (required for NAT)
resource "google_compute_router" "main" {
  name    = "router-${var.environment}"
  region  = var.region
  network = google_compute_network.main.id
}

# Cloud NAT (private GKE nodes need this for outbound)
resource "google_compute_router_nat" "main" {
  name                               = "nat-${var.environment}"
  router                             = google_compute_router.main.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         GCP NETWORKING CHEAT SHEET               │
├──────────────────────────────────────────────────┤
│ VPC:                                             │
│   Global (not regional like Azure VNet!)         │
│   auto_create_subnetworks = false (always!)      │
│                                                  │
│ GKE NETWORKING:                                  │
│   Primary range = node IPs                       │
│   Secondary ranges = pod IPs + service IPs       │
│   private_ip_google_access = true                │
│                                                  │
│ FIREWALL:                                        │
│   Tag-based (not subnet-based like Azure NSG)    │
│   target_tags = ["gke-node", "allow-ssh"]        │
│   Priority: lower number = higher priority       │
│                                                  │
│ PRIVATE ACCESS:                                  │
│   Cloud NAT = outbound internet for private VMs  │
│   Private Google Access = reach Google APIs       │
│   Private Service Connect = private access to    │
│     managed services (Cloud SQL, etc.)           │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: Azure VNet vs GCP VPC — key differences?**
- VPC is global (spans all regions), VNet is regional
- GCP firewall is tag-based (flexible), Azure NSG is subnet/NIC-based
- GKE needs secondary IP ranges (pods + services), AKS uses Azure CNI directly
- GCP IAP for SSH (no bastion needed), Azure needs bastion/jumpbox

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] VPC + custom subnets provision முடியும்
- [ ] Secondary IP ranges for GKE configure முடியும்
- [ ] Firewall rules (tag-based) create முடியும்
- [ ] Cloud NAT for private nodes setup முடியும்
