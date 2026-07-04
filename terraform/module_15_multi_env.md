# Module 15: Multi-Environment Patterns

## 📖 Story: The Restaurant Chain Problem

**English:**
Imagine you own a restaurant chain. You have the same menu (code) but different locations (environments):
- **Dev** = Test kitchen — experiment freely, break things, no customers affected
- **Staging** = Soft launch location — real setup, but limited audience tests before public
- **Prod** = Main restaurants — customers eating, zero tolerance for mistakes

The challenge: How do you keep all locations running the SAME menu (infrastructure code) but with different sizes (small kitchen for dev, full kitchen for prod)?

Three approaches exist:
1. **Workspaces** = Same kitchen blueprint, just switch a label ("now I'm dev", "now I'm prod") — simple but DANGEROUS (one mistake affects all)
2. **Directories** = Separate kitchen blueprints per location, shared recipe book (modules) — more files but ISOLATED
3. **Terragrunt** = A manager who coordinates all kitchens from one place, ensures DRY recipes — powerful but adds complexity

At TVS, we used the **directory-based** approach with shared modules. At EB, we used the same. It's the industry standard for teams that need isolation and clarity.

**தமிழ்:**
நீங்கள் restaurant chain வைத்திருக்கிறீர்கள் என்று நினைக்கவும். Same menu (code) ஆனால் different locations (environments):
- **Dev** = Test kitchen — freely experiment, break things
- **Staging** = Soft launch — real setup, limited audience
- **Prod** = Main restaurants — customers eating, zero mistakes

Challenge: எல்லா locations-ம் SAME menu (infrastructure code) run ஆக வேண்டும், ஆனால் different sizes (dev-க்கு small, prod-க்கு full).

மூன்று approaches:
1. **Workspaces** = Same blueprint, label switch — simple ஆனால் DANGEROUS
2. **Directories** = ஒவ்வொரு location-க்கும் separate blueprints, shared recipe book (modules) — more files ஆனால் ISOLATED
3. **Terragrunt** = ஒரு manager எல்லா kitchens-ஐயும் coordinate, DRY recipes ensure — powerful ஆனால் complexity add

TVS-ல் **directory-based** approach + shared modules use செய்தோம். EB-லும் same. Industry standard.

---

## 📊 Architecture & Concepts

### Comparison Matrix: The Big Three Approaches

| Criteria | Workspaces | Directories | Terragrunt |
|----------|-----------|-------------|------------|
| **DRY (no repeat)** | ⭐⭐⭐ (one copy) | ⭐⭐ (some repetition) | ⭐⭐⭐ (wrapper keeps DRY) |
| **Isolation** | ⭐ (shared code, separate state) | ⭐⭐⭐ (fully separate) | ⭐⭐⭐ (fully separate) |
| **Complexity** | ⭐ (minimal) | ⭐⭐ (moderate) | ⭐⭐⭐ (tool overhead) |
| **CI/CD friendly** | ⭐⭐ (workspace switching) | ⭐⭐⭐ (natural directory mapping) | ⭐⭐ (needs Terragrunt in CI) |
| **Team clarity** | ⭐ (confusing for new devs) | ⭐⭐⭐ (obvious structure) | ⭐⭐ (learning curve) |
| **Blast radius** | ⭐ (mistake can affect all) | ⭐⭐⭐ (limited to one env) | ⭐⭐⭐ (limited to one env) |
| **Refactoring** | ⭐⭐⭐ (change once) | ⭐⭐ (change in each env) | ⭐⭐⭐ (change once) |
| **Best for** | Personal/small | Teams (5-100 devs) | Large orgs (100+ devs) |

### Approach 1: Workspaces (Know it, Avoid it for Multi-Env)

```
┌────────────────────────────────────────────────────────────────┐
│                  WORKSPACE APPROACH                              │
│                                                                  │
│  terraform/                                                     │
│  ├── main.tf          ← ONE copy of all code                   │
│  ├── variables.tf                                               │
│  ├── dev.tfvars       ← Different values per env               │
│  ├── staging.tfvars                                             │
│  └── prod.tfvars                                                │
│                                                                  │
│  Usage:                                                         │
│    terraform workspace select dev                               │
│    terraform plan -var-file=dev.tfvars                          │
│                                                                  │
│  State:                                                         │
│    gs://bucket/terraform.tfstate.d/dev/default.tfstate          │
│    gs://bucket/terraform.tfstate.d/staging/default.tfstate      │
│    gs://bucket/terraform.tfstate.d/prod/default.tfstate         │
│                                                                  │
│  ⚠️ PROBLEMS:                                                   │
│  • terraform workspace select prod + wrong var-file = DISASTER  │
│  • All environments use exact same code (no env-specific stuff) │
│  • CI/CD: workspace switching adds risk of applying wrong env   │
│  • One broken main.tf = ALL environments broken                 │
│  • Hard to have different module versions per environment       │
└────────────────────────────────────────────────────────────────┘
```

### Approach 2: Directory-Based (RECOMMENDED)

```
┌────────────────────────────────────────────────────────────────┐
│              DIRECTORY-BASED APPROACH (RECOMMENDED)              │
│                                                                  │
│  terraform/                                                     │
│  ├── modules/                    ← Shared, reusable modules     │
│  │   ├── networking/                                            │
│  │   │   ├── main.tf                                           │
│  │   │   ├── variables.tf                                      │
│  │   │   └── outputs.tf                                        │
│  │   ├── compute/                                              │
│  │   └── database/                                             │
│  │                                                              │
│  └── environments/               ← Per-environment configs      │
│      ├── dev/                                                   │
│      │   ├── main.tf             ← Calls shared modules        │
│      │   ├── variables.tf                                      │
│      │   ├── terraform.tfvars    ← Dev-specific values         │
│      │   └── backend.tf          ← Dev state bucket            │
│      ├── staging/                                              │
│      │   ├── main.tf                                           │
│      │   ├── variables.tf                                      │
│      │   ├── terraform.tfvars                                  │
│      │   └── backend.tf                                        │
│      └── prod/                                                  │
│          ├── main.tf                                            │
│          ├── variables.tf                                      │
│          ├── terraform.tfvars                                  │
│          └── backend.tf                                        │
│                                                                  │
│  ✅ BENEFITS:                                                   │
│  • Clear isolation: cd into env, run terraform — no confusion  │
│  • CI/CD natural: one pipeline per directory                   │
│  • Different module versions per env possible                  │
│  • Blast radius limited to one environment                     │
│  • Team can own specific environments                          │
│  • PR changes clearly show WHICH environment is affected       │
└────────────────────────────────────────────────────────────────┘
```

### Approach 3: Terragrunt (DRY Wrapper)

```
┌────────────────────────────────────────────────────────────────┐
│              TERRAGRUNT APPROACH                                 │
│                                                                  │
│  infrastructure/                                                │
│  ├── terragrunt.hcl              ← Root config (DRY settings)  │
│  ├── modules/                    ← Terraform modules            │
│  │   ├── networking/                                            │
│  │   ├── compute/                                              │
│  │   └── database/                                             │
│  └── environments/                                             │
│      ├── dev/                                                   │
│      │   ├── terragrunt.hcl      ← Just variables + includes  │
│      │   ├── networking/                                       │
│      │   │   └── terragrunt.hcl  ← 10 lines (source + vars)  │
│      │   ├── compute/                                          │
│      │   │   └── terragrunt.hcl                               │
│      │   └── database/                                         │
│      │       └── terragrunt.hcl                               │
│      ├── staging/                                              │
│      │   └── ... (same structure, different vars)              │
│      └── prod/                                                  │
│          └── ...                                                │
│                                                                  │
│  Root terragrunt.hcl:                                          │
│    remote_state { ... }          ← Configure backend once      │
│    generate "provider" { ... }   ← Generate provider once      │
│                                                                  │
│  ✅ BENEFITS: Maximum DRY, consistent backends                  │
│  ⚠️ COST: Extra tool to learn, maintain, and debug             │
└────────────────────────────────────────────────────────────────┘
```

### Environment Promotion Flow

```
┌────────────────────────────────────────────────────────────────┐
│           ENVIRONMENT PROMOTION PATTERN                          │
│                                                                  │
│  Module Change Flow:                                            │
│                                                                  │
│  1. Developer updates module (e.g., modules/networking/)        │
│  2. PR targets dev environment FIRST                            │
│                                                                  │
│     modules/networking/ (v1.2.0)                                │
│           │                                                     │
│           ▼                                                     │
│     environments/dev/main.tf                                    │
│       source = "../modules/networking"  ← Latest                │
│           │                                                     │
│           │ ✅ Works in dev (1-2 days soak)                     │
│           ▼                                                     │
│     environments/staging/main.tf                                │
│       source = "../modules/networking"  ← Promote               │
│           │                                                     │
│           │ ✅ Works in staging (3-5 days soak)                 │
│           ▼                                                     │
│     environments/prod/main.tf                                   │
│       source = "../modules/networking"  ← Promote (approved)    │
│                                                                  │
│  With versioned modules (registry):                             │
│     dev:     source = "app.terraform.io/org/networking/gcp"     │
│              version = "~> 1.2"    ← Gets latest 1.2.x         │
│     staging: version = "1.2.0"    ← Pinned                     │
│     prod:    version = "1.1.5"    ← Previous stable            │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Byheart for Interview

### Decision Framework (Memorize This!)

```
"Which multi-env approach should I use?"

IF team < 5 AND simple infra → Workspaces (quick, minimal)
IF team 5-100 AND need isolation → Directories (recommended default)
IF team > 100 AND many services → Terragrunt (DRY at scale)
IF using HCP Terraform → Workspaces work better (native support)

My default recommendation: DIRECTORY-BASED
  Reason: "Clear isolation, natural CI/CD mapping, team-friendly"
```

### The "5 Environments Without Duplication" Answer

```
Structure:
  terraform/
  ├── modules/     (shared logic — DRY)
  └── environments/
      ├── dev/     (thin wrapper — calls modules)
      ├── staging/
      ├── prod/
      ├── dr/
      └── sandbox/

Each environment main.tf is ~50 lines:
  - Module source reference
  - Environment-specific variables
  - Backend configuration

The MODULE does the heavy lifting (200+ lines).
The ENVIRONMENT just passes parameters.

"It's like a function call — the function is written once,
 called 5 times with different arguments"
```

### Key Tamil Memory Aids

```
Directory approach = ஒவ்வொரு environment-க்கும் தனி அறை (separate room for each)
Workspaces = ஒரே அறை, label மாற்றுவது (same room, change the label)
Terragrunt = எல்லா அறைகளையும் manage செய்யும் manager
Module = Recipe (சமையல் குறிப்பு) — ஒருமுறை எழுது, பலமுறை use
Promotion = dev → staging → prod (படிப்படியாக உயர்த்துவது)
```

---

## ⚡ Quick Hands-on

### Lab Setup

```bash
# SSH to lab server
ssh root@203.57.85.108

# Create lab directory
mkdir -p ~/tf-lab/multi-env && cd ~/tf-lab/multi-env
```

### Exercise 1: Build the Directory Structure

```bash
# Create the recommended directory-based structure
mkdir -p modules/networking
mkdir -p modules/compute
mkdir -p environments/{dev,staging,prod}

# Create the shared networking module
cat > modules/networking/main.tf << 'EOF'
# Shared Networking Module
# Called by all environments with different parameters

variable "project_id" {
  type = string
}

variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_cidrs" {
  type = map(string)
  default = {
    app = "10.0.1.0/24"
    db  = "10.0.2.0/24"
  }
}

variable "enable_nat" {
  type    = bool
  default = true
}

resource "google_compute_network" "vpc" {
  name                    = "${var.environment}-vpc"
  project                 = var.project_id
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnets" {
  for_each = var.subnet_cidrs

  name          = "${var.environment}-${each.key}-subnet"
  project       = var.project_id
  network       = google_compute_network.vpc.id
  ip_cidr_range = each.value
  region        = "us-central1"

  log_config {
    aggregation_interval = "INTERVAL_5_SEC"
    flow_sampling        = var.environment == "prod" ? 1.0 : 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

# NAT for private instances (all environments)
resource "google_compute_router" "router" {
  count   = var.enable_nat ? 1 : 0
  name    = "${var.environment}-router"
  project = var.project_id
  network = google_compute_network.vpc.id
  region  = "us-central1"
}

resource "google_compute_router_nat" "nat" {
  count  = var.enable_nat ? 1 : 0
  name   = "${var.environment}-nat"
  router = google_compute_router.router[0].name
  region = "us-central1"

  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"

  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

output "vpc_id" {
  value = google_compute_network.vpc.id
}

output "subnet_ids" {
  value = { for k, v in google_compute_subnetwork.subnets : k => v.id }
}
EOF

cat > modules/networking/outputs.tf << 'EOF'
# Outputs already in main.tf for this demo
# In production, separate file for clarity
EOF

echo "Shared networking module created!"
```

### Exercise 2: Create Environment Configurations

```bash
# DEV environment — small, cost-effective
cat > environments/dev/main.tf << 'EOF'
# Dev Environment — calls shared modules with dev-specific values
terraform {
  required_version = ">= 1.7.0"
  
  backend "gcs" {
    bucket = "myproject-tf-state"
    prefix = "environments/dev"
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

# Use shared networking module
module "networking" {
  source = "../../modules/networking"

  project_id  = var.project_id
  environment = "dev"
  vpc_cidr    = "10.10.0.0/16"
  subnet_cidrs = {
    app = "10.10.1.0/24"
    db  = "10.10.2.0/24"
  }
  enable_nat = true
}

# Dev-specific: smaller resources, auto-shutdown
# (Additional resources specific to dev only)
EOF

cat > environments/dev/variables.tf << 'EOF'
variable "project_id" {
  type    = string
  default = "myproject-dev"
}
EOF

cat > environments/dev/terraform.tfvars << 'EOF'
project_id = "myproject-dev"
EOF

# STAGING environment — mirrors prod size at smaller scale
cat > environments/staging/main.tf << 'EOF'
# Staging Environment — same structure as prod, smaller scale
terraform {
  required_version = ">= 1.7.0"
  
  backend "gcs" {
    bucket = "myproject-tf-state"
    prefix = "environments/staging"
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

module "networking" {
  source = "../../modules/networking"

  project_id  = var.project_id
  environment = "staging"
  vpc_cidr    = "10.20.0.0/16"
  subnet_cidrs = {
    app = "10.20.1.0/24"
    db  = "10.20.2.0/24"
    gke = "10.20.3.0/22"  # Staging has GKE like prod
  }
  enable_nat = true
}
EOF

cat > environments/staging/variables.tf << 'EOF'
variable "project_id" {
  type    = string
  default = "myproject-staging"
}
EOF

cat > environments/staging/terraform.tfvars << 'EOF'
project_id = "myproject-staging"
EOF

# PROD environment — full size, all features
cat > environments/prod/main.tf << 'EOF'
# Production Environment — full size, all security features
terraform {
  required_version = ">= 1.7.0"
  
  backend "gcs" {
    bucket = "myproject-tf-state-prod"  # Separate bucket for prod!
    prefix = "environments/prod"
  }
}

provider "google" {
  project = var.project_id
  region  = "us-central1"
}

module "networking" {
  source = "../../modules/networking"

  project_id  = var.project_id
  environment = "prod"
  vpc_cidr    = "10.30.0.0/16"
  subnet_cidrs = {
    app     = "10.30.1.0/24"
    db      = "10.30.2.0/24"
    gke     = "10.30.4.0/22"
    gke_svc = "10.30.8.0/22"
    gke_pod = "10.30.16.0/20"
  }
  enable_nat = true
}

# Prod-specific: HA, monitoring, backup policies
# These don't exist in dev/staging
resource "google_monitoring_alert_policy" "vpc_flow_alert" {
  display_name = "Unusual VPC Traffic"
  project      = var.project_id
  combiner     = "OR"

  conditions {
    display_name = "High egress traffic"
    condition_threshold {
      filter          = "resource.type=\"gce_subnetwork\""
      comparison      = "COMPARISON_GT"
      threshold_value = 1000000000  # 1GB/s
      duration        = "300s"
    }
  }

  notification_channels = var.notification_channels
}
EOF

cat > environments/prod/variables.tf << 'EOF'
variable "project_id" {
  type    = string
  default = "myproject-prod"
}

variable "notification_channels" {
  type    = list(string)
  default = []
}
EOF

cat > environments/prod/terraform.tfvars << 'EOF'
project_id             = "myproject-prod"
notification_channels  = ["projects/myproject-prod/notificationChannels/12345"]
EOF

echo ""
echo "All environments created! Structure:"
find . -name "*.tf" -o -name "*.tfvars" | sort
```

### Exercise 3: Terragrunt Example (DRY Version)

```bash
mkdir -p ~/tf-lab/multi-env/terragrunt-example/{dev,staging,prod}/networking

# Root terragrunt.hcl — shared config
cat > ~/tf-lab/multi-env/terragrunt-example/terragrunt.hcl << 'EOF'
# Root terragrunt.hcl — applies to ALL environments
# This is the DRY magic — configure once, inherit everywhere

remote_state {
  backend = "gcs"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket   = "myproject-tf-state"
    prefix   = "${path_relative_to_include()}/terraform.tfstate"
    project  = "myproject-shared"
    location = "US"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOT
    provider "google" {
      project = var.project_id
      region  = "us-central1"
    }
  EOT
}

# Common inputs for all environments
inputs = {
  region = "us-central1"
}
EOF

# Dev networking — just 10 lines!
cat > ~/tf-lab/multi-env/terragrunt-example/dev/networking/terragrunt.hcl << 'EOF'
# Dev networking — Terragrunt wrapper
# This is ALL you need per environment per component

include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/networking"
}

inputs = {
  project_id  = "myproject-dev"
  environment = "dev"
  vpc_cidr    = "10.10.0.0/16"
  subnet_cidrs = {
    app = "10.10.1.0/24"
    db  = "10.10.2.0/24"
  }
  enable_nat = true
}
EOF

# Prod networking — different values, same structure
cat > ~/tf-lab/multi-env/terragrunt-example/prod/networking/terragrunt.hcl << 'EOF'
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/networking"
}

inputs = {
  project_id  = "myproject-prod"
  environment = "prod"
  vpc_cidr    = "10.30.0.0/16"
  subnet_cidrs = {
    app     = "10.30.1.0/24"
    db      = "10.30.2.0/24"
    gke     = "10.30.4.0/22"
    gke_svc = "10.30.8.0/22"
    gke_pod = "10.30.16.0/20"
  }
  enable_nat = true
}
EOF

echo ""
echo "Terragrunt example created!"
echo "Notice: Each environment file is ~15 lines vs ~50 lines in directory approach"
echo "Trade-off: Extra tool (Terragrunt) vs. slightly more repetition"
```

### Exercise 4: Compare Side by Side

```bash
echo "========================================"
echo "  COMPARISON: Lines of Code per Approach"
echo "========================================"
echo ""
echo "DIRECTORY-BASED (environments/dev/):"
wc -l environments/dev/*.tf environments/dev/*.tfvars 2>/dev/null || echo "  ~50 lines per environment"
echo ""
echo "TERRAGRUNT (terragrunt-example/dev/networking/):"
wc -l ~/tf-lab/multi-env/terragrunt-example/dev/networking/terragrunt.hcl 2>/dev/null || echo "  ~15 lines per component per environment"
echo ""
echo "KEY INSIGHT:"
echo "  Directory: More repetition, but ZERO extra tooling"
echo "  Terragrunt: Less repetition, but MUST install + maintain Terragrunt"
echo ""
echo "For MOST teams (5-50 devs): Directory wins on simplicity"
echo "For LARGE orgs (50+ services): Terragrunt wins on DRY"
```

---

## 🔥 Scenario Challenge

### Scenario: "You Inherited 5 Environments in One Terraform Workspace"

**Situation:** A new client has ALL environments (dev, staging, prod, DR, sandbox) managed by ONE big Terraform workspace. One developer accidentally applied dev changes to prod last week. You need to fix the architecture.

**Senior Architect Migration Plan:**

```
Phase 1: ASSESSMENT (Week 1)
├── Map all resources per environment
├── Identify shared vs environment-specific resources
├── Document current state file structure
└── Identify dependencies between resources

Phase 2: CREATE NEW STRUCTURE (Week 2)
├── Create directory-based structure (environments/*)
├── Write shared modules extracting common patterns
├── Create new backend configurations (separate state per env)
└── Test with dev environment FIRST

Phase 3: MIGRATE STATE (Week 3-4)
├── terraform state mv (move resources to new structure)
├── OR: terraform import (re-import into new state files)
├── Validate: terraform plan shows no changes (identity)
├── One environment at a time: dev → staging → prod
└── Keep old state as backup until verified

Phase 4: GOVERNANCE (Week 5)
├── Separate CI pipelines per environment
├── Different approval chains (auto for dev, manual for prod)
├── RBAC: dev team can't trigger prod pipeline
└── Drift detection per environment

RISK MITIGATION:
├── Blue-green: new structure runs alongside old for 1 week
├── Canary: migrate dev first, validate for 3 days
├── Rollback: old state file preserved, can revert
└── Communication: all stakeholders informed of migration
```

---

## 🏗️ Real Project Reference: TVS & EB

### TVS Project — Directory-Based with Shared Modules

```
┌────────────────────────────────────────────────────────────────┐
│           TVS PROJECT — Multi-Environment Structure             │
│                                                                  │
│  tvs-infrastructure/                                            │
│  ├── modules/                                                   │
│  │   ├── gke-cluster/        ← Shared GKE module               │
│  │   ├── cloud-sql/          ← Shared database module          │
│  │   ├── networking/         ← Shared VPC/firewall module      │
│  │   └── monitoring/         ← Shared observability module     │
│  │                                                              │
│  └── environments/                                             │
│      ├── dev/                                                   │
│      │   ├── main.tf         ← Small GKE (3 nodes)            │
│      │   ├── terraform.tfvars                                  │
│      │   └── backend.tf      ← Dev state bucket               │
│      ├── staging/                                              │
│      │   ├── main.tf         ← Medium GKE (5 nodes)           │
│      │   └── ...                                               │
│      └── prod/                                                  │
│          ├── main.tf         ← Large GKE (10+ nodes, HA)      │
│          ├── terraform.tfvars                                  │
│          ├── backend.tf      ← SEPARATE prod state bucket     │
│          └── dr.tf           ← DR-specific resources          │
│                                                                  │
│  Key Decisions:                                                 │
│  • Separate state BUCKET for prod (not just prefix)            │
│  • Module versioning via git tags                              │
│  • Dev auto-applies, prod requires 2 approvals                 │
│  • DR environment mirrors prod (separate region)               │
└────────────────────────────────────────────────────────────────┘
```

### Environment-Specific Differences (TVS)

```
┌─────────────────┬──────────────┬──────────────┬──────────────┐
│ Resource        │ Dev          │ Staging      │ Prod         │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ GKE nodes       │ 3 (e2-std-4)│ 5 (e2-std-4)│ 10 (n2-std-8)│
│ Cloud SQL       │ db-f1-micro  │ db-custom-2  │ db-custom-8  │
│ SQL HA          │ No           │ No           │ Yes (regional)│
│ SQL backups     │ Daily/7d     │ Daily/14d    │ Hourly/30d   │
│ Monitoring      │ Basic        │ Full         │ Full + PD    │
│ VPC Flow logs   │ 50% sample   │ 100%         │ 100%         │
│ Deletion protect│ No           │ Yes          │ Yes          │
│ State bucket    │ Shared       │ Shared       │ SEPARATE     │
│ CI approval     │ Auto         │ 1 approver   │ 2 approvers  │
└─────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## 🎤 Interview Q&A

### Q1: "How do you manage 5 environments without code duplication?"

**English Answer:**
"I use the directory-based approach with shared modules — it's the sweet spot between DRY and isolation.

**Structure:** Shared modules contain 80% of the logic (networking, compute, database patterns). Each environment directory is a thin wrapper (~50 lines) that calls these modules with environment-specific values.

**Why not workspaces?** They share code too tightly — one mistake propagates everywhere. Also, CI/CD is awkward with workspace switching.

**Why not Terragrunt?** It adds operational complexity (another tool to install, upgrade, debug). For teams under 100 developers, the directory approach gives sufficient DRY-ness without the tooling overhead.

**The key insight:** The duplication in directory-based is intentional — it's the VARIABLE part (sizes, counts, feature flags). The LOGIC lives in shared modules and is written once."

**தமிழ் Answer:**
"Directory-based approach + shared modules use செய்கிறேன் — DRY + isolation-க்கு இடையிலான sweet spot.

**Structure:** Shared modules 80% logic contain — networking, compute, database patterns. ஒவ்வொரு environment directory thin wrapper (~50 lines) — modules-ஐ environment-specific values-உடன் call செய்கிறது.

**Workspaces ஏன் வேண்டாம்?** Code too tightly share. ஒரு mistake எல்லா இடத்திலும் propagate.

**Terragrunt ஏன் வேண்டாம்?** Operational complexity add. 100 developers-க்கு கீழே, directory approach sufficient DRY-ness tooling overhead இல்லாமல் தருகிறது.

**Key insight:** Directory-based-ல் duplication intentional — VARIABLE part (sizes, counts). LOGIC shared modules-ல் ஒருமுறை எழுதப்படுகிறது."

---

### Q2: "When would you choose Terragrunt over directories?"

**English Answer:**
"Terragrunt makes sense in three situations:

1. **Many services × many environments:** If you have 20 services across 5 environments, that's 100 directories. Terragrunt reduces each to ~15 lines instead of ~50.

2. **Cross-environment orchestration:** Terragrunt's `run-all` can plan/apply all environments at once — useful for infra-wide changes.

3. **Strict DRY requirement:** If your organization requires that backend configuration, provider blocks, and common variables are defined ONCE and only once.

But I'd warn: Terragrunt is a DEPENDENCY. Your team must maintain it, upgrade it, debug it. If someone new joins, they need to learn Terraform AND Terragrunt. For most teams I've worked with (TVS, EB), directory-based was sufficient and simpler to onboard."

**தமிழ் Answer:**
"Terragrunt மூன்று situations-ல் makes sense:

1. **Many services × many environments:** 20 services × 5 environments = 100 directories. Terragrunt ஒவ்வொன்றையும் ~15 lines-க்கு reduce.

2. **Cross-environment orchestration:** `run-all` — எல்லா environments-ஐ ஒரே நேரத்தில் plan/apply.

3. **Strict DRY:** Backend, provider, common variables ONCE மட்டும் define.

Warning: Terragrunt DEPENDENCY. Team maintain, upgrade, debug செய்ய வேண்டும். New joiners Terraform AND Terragrunt learn செய்ய வேண்டும். TVS, EB-ல் directory-based sufficient + simpler."

---

### Q3: "How do you promote changes from dev to prod safely?"

**English Answer:**
"Promotion follows a graduated validation approach:

1. **Module change:** Developer updates the shared module and creates a PR targeting dev first.

2. **Dev validation:** PR merged → CI applies to dev → automated tests run → 1-2 day soak period.

3. **Staging promotion:** New PR updates staging environment to use the updated module → CI applies → integration tests → 3-5 day soak.

4. **Prod promotion:** Final PR updates prod → security team review → manual approval from 2 tech leads → apply during maintenance window.

**Version pinning:** In strict environments, modules use git tags. Dev uses `main`, staging uses release candidate tag, prod uses stable tag. This ensures prod only gets code that's been validated for weeks.

The key: NEVER promote directly from a developer's branch to prod. Always go through the pipeline."

**தமிழ் Answer:**
"Promotion graduated validation approach follow செய்கிறது:

1. **Module change:** Developer shared module update, dev-ஐ target செய்து PR create.
2. **Dev validation:** Merge → apply → tests → 1-2 நாள் soak.
3. **Staging promotion:** New PR staging update → integration tests → 3-5 நாள் soak.
4. **Prod promotion:** Final PR → security review → 2 tech leads approval → maintenance window.

**Version pinning:** Dev `main` use, staging release candidate tag, prod stable tag. Prod-க்கு weeks validated code மட்டுமே.

Key: Developer branch-லிருந்து நேரடியாக prod-க்கு NEVER promote. Always pipeline through."

---

## ✅ Self-Check

### Knowledge Verification

- [ ] Can you explain workspaces vs directories vs Terragrunt in 2 sentences each?
- [ ] Can you draw the directory structure from memory?
- [ ] Do you know when to recommend each approach?
- [ ] Can you explain environment promotion flow?
- [ ] Can you describe the TVS/EB multi-env architecture?

### Interview Readiness

- [ ] Can you answer "5 environments without duplication" confidently?
- [ ] Can you explain Terragrunt trade-offs honestly?
- [ ] Can you design a migration from workspaces to directories?
- [ ] Can you talk about prod safety measures?

### Architecture Competence

```bash
# Can you create this structure from memory?
mkdir -p terraform/{modules/{networking,compute,database},environments/{dev,staging,prod}}
# Each environment has: main.tf, variables.tf, terraform.tfvars, backend.tf
# Each module has: main.tf, variables.tf, outputs.tf
```

---

## 📋 Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│       MULTI-ENVIRONMENT — CHEAT SHEET                           │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RECOMMENDED: Directory-based + Shared Modules                  │
│                                                                  │
│  terraform/                                                     │
│  ├── modules/          (write once, use everywhere)             │
│  └── environments/                                             │
│      ├── dev/          (thin wrapper + dev values)              │
│      ├── staging/      (thin wrapper + staging values)          │
│      └── prod/         (thin wrapper + prod values)             │
│                                                                  │
│  DECISION MATRIX:                                               │
│    Workspaces:  < 5 people, simple infra                       │
│    Directories: 5-100 people, need isolation (DEFAULT CHOICE)  │
│    Terragrunt:  100+ people, many services × many envs         │
│                                                                  │
│  PROMOTION:                                                     │
│    dev (auto) → staging (1 approval) → prod (2 approvals)      │
│    Soak time: dev 1-2 days, staging 3-5 days                   │
│                                                                  │
│  KEY PRINCIPLES:                                                │
│    • Prod state in SEPARATE bucket (not just prefix)           │
│    • Module = logic (written once)                              │
│    • Environment = variables (different per env)                │
│    • CI/CD maps naturally to directories                       │
│    • Never promote without soak time                           │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```
