# Module 09: GCP Networking with Terraform

## 📖 Story: The Global Highway System

**English:**
If Azure networking is like designing a city's highway system, GCP networking is like designing a **country-wide highway system**. The biggest difference? GCP VPC is **global** — one VPC spans ALL regions. Azure VNet is regional — you need peering to connect VNets across regions. Think of it this way: Azure gives you separate city road networks that you connect with bridges (peering). GCP gives you ONE national highway network where you just add lanes (subnets) in different states (regions).

In EB's build infrastructure, we used GCP VPCs for CI agent VMs. The global VPC meant our build agents in us-central1 could talk to artifact stores in asia-south1 **without any peering setup** — they're in the same VPC, just different subnets in different regions. This simplifies multi-region architectures significantly.

**தமிழ்:**
Azure networking ஒரு நகரத்தின் highway system வடிவமைப்பது போன்றது என்றால், GCP networking ஒரு **நாடு முழுவதும் பரந்த highway system** வடிவமைப்பது போன்றது. மிகப்பெரிய வேறுபாடு? GCP VPC **global** — ஒரு VPC எல்லா regions-ஐயும் உள்ளடக்கும். Azure VNet regional — regions இடையே இணைக்க peering தேவை. இப்படி நினைக்கவும்: Azure தனித்தனி நகர சாலை networks கொடுக்கிறது, bridges (peering) மூலம் இணைக்க வேண்டும். GCP ஒரே தேசிய highway network கொடுக்கிறது — வெவ்வேறு மாநிலங்களில் (regions) lanes (subnets) சேர்க்கலாம்.

EB build infrastructure-ல், CI agent VMs-க்கு GCP VPCs பயன்படுத்தினோம். Global VPC காரணமாக us-central1-ல் உள்ள build agents, asia-south1-ல் உள்ள artifact stores-உடன் **எந்த peering setup இல்லாமல்** பேச முடிந்தது — ஒரே VPC-ல், வெவ்வேறு regions-ல் வெவ்வேறு subnets. Multi-region architectures-ஐ இது மிகவும் எளிமையாக்குகிறது.

---

## 📊 Architecture & Concepts

### GCP VPC Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    GCP VPC (GLOBAL)                                │
│                                                                    │
│  ┌─────────────────────┐       ┌─────────────────────┐           │
│  │  Region: us-central1│       │  Region: asia-south1│           │
│  │                     │       │                     │           │
│  │  Subnet: build-agents       │  Subnet: artifacts  │           │
│  │  10.1.0.0/24        │       │  10.2.0.0/24        │           │
│  │  ┌────┐ ┌────┐     │       │  ┌────┐             │           │
│  │  │ VM │ │ VM │     │       │  │ GCS│             │           │
│  │  └────┘ └────┘     │       │  └────┘             │           │
│  └─────────────────────┘       └─────────────────────┘           │
│                                                                    │
│  ┌─────────────────────┐       ┌─────────────────────┐           │
│  │  Region: europe-west1       │  Region: us-east1   │           │
│  │                     │       │                     │           │
│  │  Subnet: monitoring │       │  Subnet: gke-nodes  │           │
│  │  10.3.0.0/24        │       │  10.4.0.0/20        │           │
│  └─────────────────────┘       └─────────────────────┘           │
│                                                                    │
│  Firewall Rules (VPC-level, tag-based)                            │
│  Cloud Router + Cloud NAT (per region)                            │
└──────────────────────────────────────────────────────────────────┘
```

### Azure vs GCP Networking — Key Comparison

| Feature | Azure VNet | GCP VPC |
|---------|-----------|---------|
| **Scope** | Regional | **Global** |
| **Subnets** | Regional (within VNet) | Regional (within global VPC) |
| **Cross-region** | Requires VNet Peering | Same VPC, automatic |
| **Firewall** | NSG (subnet/NIC level) | Firewall Rules (VPC-level, tag-based) |
| **NAT** | NAT Gateway (per subnet) | Cloud NAT (per region, per router) |
| **Route tables** | UDR per subnet | Routes per VPC (can filter by tags) |
| **DNS** | Private DNS Zones | Cloud DNS Private Zones |
| **Peering** | VNet Peering (non-transitive) | VPC Peering (non-transitive) |
| **Load Balancing** | Regional + Global (Front Door) | Global by default (most LBs) |
| **Private access** | Private Endpoints | Private Google Access + PSC |
| **IP ranges** | Defined at VNet + Subnet | Defined at Subnet only |
| **Default network** | No default VNet | Default VPC exists (delete it!) |

### Firewall Rules: Tag-Based (GCP) vs Subnet-Based (Azure)

**English:**
This is the #1 interview differentiator. Azure NSGs are attached to subnets or NICs — rules apply based on **where** a resource lives. GCP firewall rules use **network tags** — rules apply based on **what** a resource is, regardless of subnet.

Example: Tag a VM with `ci-agent` → all firewall rules targeting `ci-agent` tag apply, no matter which subnet/region the VM is in. In Azure, you'd need to replicate NSG rules across every subnet where CI agents live.

**தமிழ்:**
இது #1 interview differentiator. Azure NSGs subnets அல்லது NICs-உடன் இணைக்கப்படும் — resource **எங்கே** உள்ளது என்பதை வைத்து rules apply ஆகும். GCP firewall rules **network tags** பயன்படுத்தும் — resource **என்ன** என்பதை வைத்து rules apply ஆகும், subnet எதுவாக இருந்தாலும்.

உதாரணம்: VM-க்கு `ci-agent` tag கொடுக்கவும் → `ci-agent` tag-ஐ target செய்யும் எல்லா firewall rules apply ஆகும், VM எந்த subnet/region-ல் இருந்தாலும். Azure-ல், CI agents இருக்கும் ஒவ்வொரு subnet-லும் NSG rules replicate செய்ய வேண்டும்.

### Cloud NAT Architecture

**English:**
Cloud NAT provides outbound internet access for VMs **without external IPs**. Unlike Azure NAT Gateway (attached to subnet), GCP Cloud NAT is attached to a **Cloud Router** in a specific region. The Cloud Router handles BGP routing, and Cloud NAT handles the NAT translation.

Key design: One Cloud Router per region → attach Cloud NAT → select which subnets get NAT.

**தமிழ்:**
Cloud NAT, external IPs **இல்லாத** VMs-க்கு outbound internet access வழங்குகிறது. Azure NAT Gateway (subnet-உடன் இணைக்கப்படும்) போலல்லாமல், GCP Cloud NAT ஒரு specific region-ல் **Cloud Router**-உடன் இணைக்கப்படும். Cloud Router BGP routing handle செய்யும், Cloud NAT translation handle செய்யும்.

முக்கிய design: ஒரு region-க்கு ஒரு Cloud Router → Cloud NAT attach → எந்த subnets NAT பெறும் என்று select செய்யவும்.

### Shared VPC (GCP's Hub-Spoke Equivalent)

**English:**
In Azure, you use Hub-Spoke topology with peering. In GCP, the equivalent is **Shared VPC**:
- A **host project** owns the VPC and subnets
- **Service projects** use subnets from the host project
- Centralized networking team controls the network; app teams deploy resources

This is GCP's recommended multi-project networking pattern — cleaner than peering for organizations.

**தமிழ்:**
Azure-ல், Hub-Spoke topology peering-உடன் பயன்படுத்துவீர்கள். GCP-ல், equivalent **Shared VPC**:
- **Host project** VPC மற்றும் subnets-ஐ own செய்யும்
- **Service projects** host project-லிருந்து subnets பயன்படுத்தும்
- Centralized networking team network control செய்யும்; app teams resources deploy செய்யும்

இது GCP-ன் recommended multi-project networking pattern — organizations-க்கு peering-ஐ விட cleaner.

---

## 🧠 Byheart for Interview

### GCP Networking Key Facts
```
1. VPC is GLOBAL — subnets are REGIONAL (opposite of Azure where VNet is regional)
2. Firewall rules are VPC-level, applied via NETWORK TAGS (not subnet-attached like NSG)
3. Firewall rule priority: lower number = higher priority (0-65535)
4. Default VPC has allow-internal + allow-ssh + allow-rdp + allow-icmp rules — DELETE IT
5. Cloud NAT = Cloud Router + NAT config (per region)
6. Shared VPC = host project + service projects (GCP's hub-spoke equivalent)
7. VPC Peering is non-transitive (same as Azure) — routes don't propagate
8. Private Google Access: VMs without external IP can reach Google APIs
9. Subnets can have SECONDARY ranges (used for GKE pod/service CIDRs)
10. Alias IP ranges allow multiple IPs per NIC (used by GKE pods)
```

### Firewall Rules Priority Logic
```
Priority 0 (highest) ────────────► Priority 65535 (lowest)
│                                                         │
│  1000: allow-internal (auto)                           │
│  1000: allow-ssh (auto)                                │
│  65534: deny-all-ingress (implied, not shown)         │
│  65535: allow-all-egress (implied, not shown)         │
│                                                         │
│  Custom rules: use 100-900 range for your rules        │
└─────────────────────────────────────────────────────────┘
```

### Interview Golden Answers
```
Q: "Why is GCP VPC global a big deal?"
A: "Cross-region communication happens within the same VPC without peering. 
    This means simpler multi-region designs, no peering limits, and internal 
    IPs work across regions. In Azure, each VNet is regional and you need 
    peering + gateway transit for cross-region, which adds hops and cost."

Q: "How do GCP firewall rules differ from Azure NSGs?"
A: "Three key differences:
    1. Scope: GCP rules are VPC-wide with tag targeting; Azure NSGs are per-subnet/NIC
    2. Identity: GCP uses network tags + service accounts; Azure uses IP/subnet
    3. Priority: GCP uses numeric priority; Azure uses priority within each NSG"

Q: "When would you use Shared VPC vs VPC Peering?"
A: "Shared VPC for organizational multi-project setups where networking team 
    manages centrally. VPC Peering for connecting VPCs across organizations 
    or when projects need fully independent networking. Similar to Azure's 
    Hub-Spoke (Shared VPC) vs cross-tenant peering."
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/gcp-network && cd ~/tf-lab/gcp-network
```

### Exercise 1: Custom VPC with Regional Subnets

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

# ─── CUSTOM VPC (delete-default-routes = no auto subnet creation) ───
resource "google_compute_network" "main" {
  name                            = "vpc-${var.environment}"
  auto_create_subnetworks         = false          # Custom mode (ALWAYS use this)
  routing_mode                    = "GLOBAL"       # Global dynamic routing
  delete_default_routes_on_create = false
}

# ─── SUBNET: Build Agents (us-central1) ────────────────────────────
resource "google_compute_subnetwork" "build_agents" {
  name          = "subnet-build-agents"
  ip_cidr_range = "10.1.0.0/24"
  region        = "us-central1"
  network       = google_compute_network.main.id

  # Enable Private Google Access (VMs without external IP can reach Google APIs)
  private_ip_google_access = true

  # Secondary ranges for GKE (if needed later)
  secondary_ip_range {
    range_name    = "pods"
    ip_cidr_range = "10.10.0.0/16"
  }
  secondary_ip_range {
    range_name    = "services"
    ip_cidr_range = "10.20.0.0/20"
  }

  log_config {
    aggregation_interval = "INTERVAL_5_SEC"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# ─── SUBNET: Application (asia-south1) ────────────────────────────
resource "google_compute_subnetwork" "app" {
  name                     = "subnet-app"
  ip_cidr_range            = "10.2.0.0/24"
  region                   = "asia-south1"
  network                  = google_compute_network.main.id
  private_ip_google_access = true
}
EOF
```

### Exercise 2: Firewall Rules with Tags

```hcl
cat > firewall.tf << 'EOF'
# ─── ALLOW INTERNAL (all VMs within VPC) ───────────────────────────
resource "google_compute_firewall" "allow_internal" {
  name    = "fw-allow-internal"
  network = google_compute_network.main.name

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

  source_ranges = ["10.0.0.0/8"]  # All internal
  priority      = 1000
}

# ─── ALLOW SSH ONLY TO TAGGED VMs ─────────────────────────────────
resource "google_compute_firewall" "allow_ssh" {
  name    = "fw-allow-ssh"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"]  # IAP range (secure SSH via Identity-Aware Proxy)
  target_tags   = ["allow-ssh"]         # Only VMs with this tag get SSH
  priority      = 100
}

# ─── ALLOW HTTP/S TO CI AGENTS ────────────────────────────────────
resource "google_compute_firewall" "allow_ci_egress" {
  name      = "fw-allow-ci-egress"
  network   = google_compute_network.main.name
  direction = "EGRESS"

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  target_tags        = ["ci-agent"]     # Only CI agent VMs
  destination_ranges = ["0.0.0.0/0"]
  priority           = 100
}

# ─── DENY ALL INGRESS (explicit, overrides default) ───────────────
resource "google_compute_firewall" "deny_all_ingress" {
  name    = "fw-deny-all-ingress"
  network = google_compute_network.main.name

  deny {
    protocol = "all"
  }

  source_ranges = ["0.0.0.0/0"]
  priority      = 65000  # Low priority — other rules override
}
EOF
```

### Exercise 3: Cloud Router + Cloud NAT

```hcl
cat > nat.tf << 'EOF'
# ─── CLOUD ROUTER (required for Cloud NAT) ───────────────────────
resource "google_compute_router" "main" {
  name    = "router-${var.region}"
  region  = var.region
  network = google_compute_network.main.id

  bgp {
    asn = 64514  # Private ASN for Cloud Router
  }
}

# ─── CLOUD NAT (outbound internet for private VMs) ───────────────
resource "google_compute_router_nat" "main" {
  name                               = "nat-${var.region}"
  router                             = google_compute_router.main.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"  # GCP assigns IPs
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }

  # Min ports per VM for connection scaling
  min_ports_per_vm = 64
}
EOF
```

### Exercise 4: Variables

```hcl
cat > variables.tf << 'EOF'
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "Primary region"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
EOF
```

```bash
# Validate the configuration
terraform init
terraform validate
terraform plan -var="project_id=my-project"
```

---

## 🔥 Scenario Challenge

### Scenario: Multi-Region CI Infrastructure

**English:**
Your company needs CI build agents in 3 regions (us-central1, europe-west1, asia-south1). Requirements:
1. All agents must access internal artifact registry (no public internet for artifacts)
2. Agents need outbound internet for pulling public packages
3. Only IAP-based SSH (no public SSH)
4. Different firewall rules for "build-agent" vs "test-runner" VMs

Design the networking solution.

**தமிழ்:**
உங்கள் company-க்கு 3 regions-ல் CI build agents தேவை. Requirements:
1. எல்லா agents-ம் internal artifact registry access செய்ய வேண்டும் (artifacts-க்கு public internet வேண்டாம்)
2. Public packages pull செய்ய outbound internet தேவை
3. IAP-based SSH மட்டும் (public SSH இல்லை)
4. "build-agent" vs "test-runner" VMs-க்கு வெவ்வேறு firewall rules

### Solution Architecture

```hcl
# Single global VPC — no peering needed for cross-region!
resource "google_compute_network" "ci" {
  name                    = "vpc-ci-infra"
  auto_create_subnetworks = false
  routing_mode            = "GLOBAL"
}

# Subnet per region
resource "google_compute_subnetwork" "ci" {
  for_each = {
    us      = { region = "us-central1", cidr = "10.1.0.0/24" }
    europe  = { region = "europe-west1", cidr = "10.2.0.0/24" }
    asia    = { region = "asia-south1", cidr = "10.3.0.0/24" }
  }

  name                     = "subnet-ci-${each.key}"
  ip_cidr_range            = each.value.cidr
  region                   = each.value.region
  network                  = google_compute_network.ci.id
  private_ip_google_access = true  # Access Artifact Registry without external IP
}

# Cloud NAT per region (for outbound internet)
resource "google_compute_router" "ci" {
  for_each = toset(["us-central1", "europe-west1", "asia-south1"])
  name     = "router-ci-${each.key}"
  region   = each.key
  network  = google_compute_network.ci.id
  bgp { asn = 64514 }
}

resource "google_compute_router_nat" "ci" {
  for_each                           = google_compute_router.ci
  name                               = "nat-ci-${each.key}"
  router                             = each.value.name
  region                             = each.value.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Tag-based firewall: build agents get different rules than test runners
resource "google_compute_firewall" "build_agent_egress" {
  name      = "fw-build-agent-egress"
  network   = google_compute_network.ci.name
  direction = "EGRESS"
  allow {
    protocol = "tcp"
    ports    = ["443"]
  }
  target_tags        = ["build-agent"]
  destination_ranges = ["0.0.0.0/0"]
  priority           = 100
}

resource "google_compute_firewall" "test_runner_limited" {
  name      = "fw-test-runner-egress"
  network   = google_compute_network.ci.name
  direction = "EGRESS"
  allow {
    protocol = "tcp"
    ports    = ["443", "5432", "6379"]  # HTTPS + DB for integration tests
  }
  target_tags        = ["test-runner"]
  destination_ranges = ["10.0.0.0/8"]  # Internal only
  priority           = 100
}

# IAP SSH only
resource "google_compute_firewall" "iap_ssh" {
  name    = "fw-allow-iap-ssh"
  network = google_compute_network.ci.name
  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  source_ranges = ["35.235.240.0/20"]  # Google IAP range
  target_tags   = ["allow-ssh"]
  priority      = 100
}
```

---

## 🏗️ Real Project Reference (EB)

**English:**
In EB's embedded CI/CT pipeline, GCP networking was designed for:
- **Global VPC** for all CI infrastructure — build agents in us-central1, artifact stores accessible from all regions
- **Private Google Access** enabled on all subnets — agents pull from Artifact Registry without external IPs
- **Cloud NAT** per region — for pulling external packages (apt, pip, npm)
- **Network tags**: `ci-build`, `ci-test`, `ci-deploy` — different egress rules per role
- **No public IPs** on any VM — all SSH via IAP tunnel

This is the "secure CI on GCP" pattern interviewers look for.

**தமிழ்:**
EB embedded CI/CT pipeline-ல், GCP networking:
- எல்லா CI infrastructure-க்கும் **Global VPC** — us-central1-ல் build agents, எல்லா regions-லிருந்தும் artifact stores access
- எல்லா subnets-லும் **Private Google Access** — agents external IPs இல்லாமல் Artifact Registry-லிருந்து pull செய்யும்
- Region-க்கு **Cloud NAT** — external packages pull செய்ய (apt, pip, npm)
- **Network tags**: `ci-build`, `ci-test`, `ci-deploy` — role-க்கு ஏற்ற egress rules
- எந்த VM-லும் **public IPs இல்லை** — எல்லா SSH-ம் IAP tunnel வழியாக

---

## 🎤 Interview Q&A

### Q1: "Explain the fundamental difference between Azure VNet and GCP VPC"

**English:**
"The fundamental difference is scope. Azure VNet is **regional** — you need VNet Peering to connect across regions, and each peering adds latency and cost. GCP VPC is **global** — subnets are regional but the VPC spans all regions. This means VMs in different regions within the same VPC can communicate using internal IPs without any additional configuration.

For enterprise architecture, this means GCP multi-region designs are simpler — one VPC, subnets per region, done. Azure requires hub-spoke with peering, gateway transit, and UDRs for the same outcome."

**தமிழ்:**
"அடிப்படை வேறுபாடு scope. Azure VNet **regional** — regions இடையே இணைக்க VNet Peering தேவை, ஒவ்வொரு peering latency மற்றும் cost சேர்க்கும். GCP VPC **global** — subnets regional ஆனால் VPC எல்லா regions-ஐயும் span செய்யும். ஒரே VPC-ல் வெவ்வேறு regions-ல் உள்ள VMs internal IPs மூலம் எந்த additional configuration இல்லாமல் communicate செய்யலாம்."

### Q2: "How do you secure VMs without public IPs in GCP?"

**English:**
"Three-layer approach:
1. **Private Google Access**: Subnets access Google APIs (Storage, Artifact Registry) via internal routing
2. **Cloud NAT**: Outbound internet through NAT — no inbound possible
3. **IAP Tunneling**: SSH/RDP through Google's Identity-Aware Proxy — authenticated, audited, no exposed ports

This eliminates the attack surface of public IPs while maintaining full functionality. In Azure, the equivalent would be Bastion + NAT Gateway + Private Endpoints."

**தமிழ்:**
"மூன்று-layer approach:
1. **Private Google Access**: Google APIs-ஐ internal routing வழியாக access
2. **Cloud NAT**: NAT மூலம் outbound internet — inbound சாத்தியமில்லை
3. **IAP Tunneling**: Google Identity-Aware Proxy வழியாக SSH/RDP — authenticated, audited

Public IPs-ன் attack surface-ஐ இது நீக்குகிறது. Azure-ல் equivalent: Bastion + NAT Gateway + Private Endpoints."

### Q3: "When would you use Shared VPC vs VPC Peering in GCP?"

**English:**
"**Shared VPC** when: Centralized networking team, multiple projects need same network, consistent firewall policies, hub-spoke organizational model. The host project owns networking, service projects consume subnets.

**VPC Peering** when: Cross-organization connectivity, projects need independent networking control, connecting to third-party GCP projects, different security domains.

Azure equivalent: Shared VPC ≈ Hub-Spoke with centralized management. VPC Peering ≈ cross-tenant VNet Peering."

**தமிழ்:**
"**Shared VPC**: Centralized networking team, பல projects-க்கு ஒரே network, consistent firewall policies. Host project networking own செய்யும், service projects subnets consume செய்யும்.

**VPC Peering**: Cross-organization connectivity, independent networking control தேவை, third-party GCP projects இணைப்பு.

Azure equivalent: Shared VPC ≈ Hub-Spoke centralized management. VPC Peering ≈ cross-tenant VNet Peering."

### Q4: "Design a network for a microservices platform on GKE"

**English:**
"I'd design with VPC-native GKE in mind:
- Custom VPC with `auto_create_subnetworks = false`
- GKE subnet with **secondary ranges** for pods (/16) and services (/20)
- Private Google Access for pulling images from GCR/Artifact Registry
- Cloud NAT for outbound (external package managers)
- Firewall rules using GKE's auto-created network tags
- Private cluster (no public endpoint) + authorized networks for kubectl
- Cloud Armor for external-facing services (WAF equivalent of Azure Front Door)"

**தமிழ்:**
"VPC-native GKE-ஐ மனதில் வைத்து design செய்வேன்:
- `auto_create_subnetworks = false` உடன் Custom VPC
- Pods (/16) மற்றும் services (/20) க்கு **secondary ranges** உடன் GKE subnet
- GCR/Artifact Registry-லிருந்து images pull செய்ய Private Google Access
- Outbound-க்கு Cloud NAT
- GKE auto-created network tags பயன்படுத்தி firewall rules
- Private cluster + authorized networks
- External services-க்கு Cloud Armor (Azure Front Door WAF equivalent)"

---

## ✅ Self-Check

| # | Question | Can You Answer? |
|---|----------|-----------------|
| 1 | What makes GCP VPC global vs Azure VNet regional? | ☐ |
| 2 | How do GCP firewall rules use network tags? | ☐ |
| 3 | What is `auto_create_subnetworks = false` and why always use it? | ☐ |
| 4 | Explain Cloud Router + Cloud NAT relationship | ☐ |
| 5 | What are secondary IP ranges and why needed for GKE? | ☐ |
| 6 | Shared VPC vs VPC Peering — when to use each? | ☐ |
| 7 | How does Private Google Access replace Private Endpoints? | ☐ |
| 8 | IAP tunneling — how does it eliminate public IP need? | ☐ |
| 9 | Firewall rule priority — what number is highest priority? | ☐ |
| 10 | Design multi-region networking without peering in GCP | ☐ |

---

*Next: [Module 10 — GCP Compute](module_10_gcp_compute.md)*
