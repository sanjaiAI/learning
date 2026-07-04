# Module 16: Advanced Patterns & Techniques

## 📖 Story: The Renovation vs New Construction Problem

**English:**
Imagine you bought an old house with 200 rooms (manually-created cloud resources). You want to manage it professionally (Terraform), but you can't demolish and rebuild — people are living there!

This is the **brownfield problem** — bringing existing infrastructure under Terraform control without disrupting running services. It's the #1 challenge for architects joining established organizations.

The solutions:
- **terraform import** = Taking a photo of each room and adding it to your blueprint (state)
- **import blocks** (Terraform 1.5+) = Declarative way to say "this room already exists, here's its address"
- **moved blocks** = Renaming rooms in your blueprint without demolishing them
- **State splitting** = One giant blueprint → multiple focused blueprints (reduces blast radius)

At EB, when we started, there were 200+ GCP resources created manually via console. We couldn't just `terraform apply` from scratch — that would DELETE everything and recreate it. We had to carefully import each resource, validate the state, then manage going forward.

This module covers the patterns that make you a **Terraform surgeon** — precise, calculated changes to live infrastructure without downtime.

**தமிழ்:**
200 rooms (manually-created cloud resources) உள்ள பழைய வீடு வாங்கினீர்கள் என்று நினைக்கவும். Professionally manage (Terraform) செய்ய விரும்புகிறீர்கள், ஆனால் demolish & rebuild செய்ய முடியாது — people living there!

இதுதான் **brownfield problem** — existing infrastructure-ஐ Terraform control-க்கு கொண்டு வருவது running services disrupt செய்யாமல்.

Solutions:
- **terraform import** = ஒவ்வொரு room-ன் photo எடுத்து blueprint-ல் add
- **import blocks** (Terraform 1.5+) = Declarative — "இந்த room already exists"
- **moved blocks** = Blueprint-ல் rooms rename — demolish இல்லை
- **State splitting** = ஒரு giant blueprint → multiple focused blueprints

EB-ல் start செய்தபோது, 200+ GCP resources manually console-ல் create செய்யப்பட்டிருந்தன. `terraform apply` from scratch செய்ய முடியாது — DELETE & recreate ஆகும். Carefully ஒவ்வொரு resource-ஐயும் import செய்தோம்.

இந்த module Terraform surgeon ஆவதற்கான patterns — live infrastructure-ல் downtime இல்லாமல் precise, calculated changes.

---

## 📊 Architecture & Concepts

### The Import Journey: Manual → Managed

```
┌────────────────────────────────────────────────────────────────┐
│          IMPORTING EXISTING RESOURCES — THE JOURNEY              │
│                                                                  │
│  BEFORE (manual mess):                                          │
│  ┌─────────────────────────────────────────────┐               │
│  │ GCP Console                                  │               │
│  │  ├── VM: prod-api-1 (who created this?)      │               │
│  │  ├── VM: prod-api-2 (when? what config?)     │               │
│  │  ├── VPC: default (everyone uses it)         │               │
│  │  ├── SQL: prod-db (manual backups, maybe?)   │               │
│  │  ├── GCS: random-bucket-2023 (what's in it?)│               │
│  │  └── ... 195 more resources                  │               │
│  └─────────────────────────────────────────────┘               │
│                                                                  │
│  AFTER (Terraform managed):                                     │
│  ┌─────────────────────────────────────────────┐               │
│  │ Terraform State                              │               │
│  │  ├── module.compute.google_compute_instance  │               │
│  │  ├── module.networking.google_compute_network│               │
│  │  ├── module.database.google_sql_instance     │               │
│  │  └── ... all resources tracked + versioned   │               │
│  └─────────────────────────────────────────────┘               │
│                                                                  │
│  IMPORT PROCESS:                                                │
│  1. Write HCL matching existing resource config                 │
│  2. Import: link HCL ↔ real resource in state                  │
│  3. Plan: should show "no changes" (identity)                  │
│  4. Going forward: all changes via Terraform                   │
└────────────────────────────────────────────────────────────────┘
```

### Import Methods Comparison

| Method | Terraform Version | Style | Best For |
|--------|------------------|-------|----------|
| `terraform import` (CLI) | Any | Imperative (one at a time) | Quick single imports |
| `import` blocks | 1.5+ | Declarative (in HCL) | Bulk imports, CI-friendly |
| `terraform import` + `-generate-config-out` | 1.5+ | Auto-generate HCL | Unknown resource configs |

### Moved Blocks: Refactoring Without Destruction

```
┌────────────────────────────────────────────────────────────────┐
│              MOVED BLOCKS — SAFE REFACTORING                    │
│                                                                  │
│  PROBLEM:                                                       │
│  You want to rename a resource or move it into a module.        │
│  Without moved block:                                           │
│    terraform plan → "destroy old + create new" 😱              │
│                                                                  │
│  WITH MOVED BLOCK:                                              │
│    terraform plan → "moved (no destroy)" ✅                    │
│                                                                  │
│  Use Cases:                                                     │
│  ├── Rename resource: aws_instance.web → aws_instance.api      │
│  ├── Move to module: resource.x → module.m.resource.x          │
│  ├── Move between modules: module.a.x → module.b.x            │
│  ├── Rename module: module.old → module.new                    │
│  └── Change for_each key: instances["a"] → instances["app"]   │
│                                                                  │
│  Syntax:                                                        │
│    moved {                                                      │
│      from = google_compute_instance.web                        │
│      to   = module.compute.google_compute_instance.web         │
│    }                                                            │
│                                                                  │
│  Lifecycle:                                                     │
│    1. Add moved block                                          │
│    2. terraform plan → shows "moved" (not destroy+create)      │
│    3. terraform apply                                           │
│    4. Remove moved block (cleanup, after next apply)           │
└────────────────────────────────────────────────────────────────┘
```

### State Splitting: Monolith → Components

```
┌────────────────────────────────────────────────────────────────┐
│          STATE SPLITTING — REDUCING BLAST RADIUS                │
│                                                                  │
│  BEFORE: One giant state (600 resources)                        │
│  ┌──────────────────────────────────────────────┐              │
│  │ terraform.tfstate                             │              │
│  │  ├── VPC, subnets, firewalls (30 resources)  │              │
│  │  ├── GKE cluster + node pools (50 resources) │              │
│  │  ├── Cloud SQL + replicas (20 resources)     │              │
│  │  ├── GCS buckets (15 resources)              │              │
│  │  ├── IAM roles + bindings (40 resources)     │              │
│  │  └── ... 445 more resources                  │              │
│  └──────────────────────────────────────────────┘              │
│  Problem: Any change = plan takes 5 min,                       │
│           risk of affecting unrelated resources                  │
│                                                                  │
│  AFTER: Split by domain (blast radius = smaller)                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐          │
│  │ networking/   │ │ compute/     │ │ data/         │          │
│  │ state         │ │ state        │ │ state         │          │
│  │ (30 resources)│ │ (50 resources│ │ (35 resources)│          │
│  └──────────────┘ └──────────────┘ └──────────────┘          │
│  ┌──────────────┐ ┌──────────────┐                            │
│  │ iam/          │ │ monitoring/  │                            │
│  │ state         │ │ state        │                            │
│  │ (40 resources)│ │ (25 resources│                            │
│  └──────────────┘ └──────────────┘                            │
│                                                                  │
│  Benefits:                                                      │
│  • Plan in seconds (not minutes)                               │
│  • Networking team can't accidentally break database           │
│  • Parallel development (no state lock conflicts)              │
│  • Blast radius limited to one domain                          │
│                                                                  │
│  Cross-state references:                                       │
│    data "terraform_remote_state" "networking" {                 │
│      backend = "gcs"                                           │
│      config = { bucket = "...", prefix = "networking/" }       │
│    }                                                           │
│    # Use: data.terraform_remote_state.networking.outputs.vpc_id│
└────────────────────────────────────────────────────────────────┘
```

### Terraform vs Alternatives

```
┌────────────────────────────────────────────────────────────────┐
│           IaC TOOL COMPARISON (ARCHITECT PERSPECTIVE)            │
│                                                                  │
│  ┌─────────────┬────────────┬────────────┬────────────────┐   │
│  │ Criteria    │ Terraform  │ Pulumi     │ Crossplane     │   │
│  ├─────────────┼────────────┼────────────┼────────────────┤   │
│  │ Language    │ HCL        │ Python/TS/ │ YAML (K8s CRDs)│   │
│  │             │ (DSL)      │ Go/C#      │                │   │
│  │ State       │ External   │ External   │ In-cluster     │   │
│  │             │ (GCS/S3)   │ (Pulumi    │ (etcd)         │   │
│  │             │            │  Cloud/S3) │                │   │
│  │ Drift fix   │ Manual     │ Manual     │ Auto-reconcile │   │
│  │ Learning    │ Medium     │ Low (if    │ High (K8s      │   │
│  │ curve       │            │  you know  │  required)     │   │
│  │             │            │  Python)   │                │   │
│  │ Ecosystem   │ Largest    │ Growing    │ K8s-native     │   │
│  │ Best for    │ General    │ Devs who   │ K8s-centric    │   │
│  │             │ multi-cloud│ hate HCL   │ teams (GitOps) │   │
│  │ Maturity    │ 10+ years  │ 5+ years   │ 3+ years       │   │
│  └─────────────┴────────────┴────────────┴────────────────┘   │
│                                                                  │
│  Also consider:                                                 │
│  • AWS CDK/CDKTF: Write in TypeScript, synthesize to TF/CF    │
│  • OpenTofu: Open-source fork of Terraform (BSL license issue) │
│  • Ansible: Config management, not ideal for cloud resources   │
│                                                                  │
│  ARCHITECT RECOMMENDATION:                                      │
│  "Terraform remains the default choice for multi-cloud IaC.    │
│   Switch only if team strongly prefers programming languages   │
│   (Pulumi) or is 100% K8s-native (Crossplane)."              │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Byheart for Interview

### The "200 Manual Resources" Answer (Framework)

```
Phase 1: DISCOVERY (Days 1-3)
├── List all resources: gcloud asset search-all-resources
├── Categorize by type (compute, network, data, IAM)
├── Identify dependencies (what references what)
└── Prioritize: critical infra first, static resources last

Phase 2: IMPORT (Days 4-14)
├── Write HCL matching current config (don't change anything!)
├── Use import blocks (Terraform 1.5+) for bulk import
├── Validate: terraform plan must show "No changes"
├── Batch by resource type (all VMs together, all SQL together)
└── Commit + PR per batch (reviewable, revertable)

Phase 3: ORGANIZE (Days 15-21)
├── Use moved blocks to organize into modules
├── Split state if needed (networking, compute, data)
├── Set up CI/CD pipeline
└── Enable drift detection

Phase 4: GOVERN (Ongoing)
├── All future changes via Terraform + CI/CD
├── Monthly drift check
├── Documentation of what's managed vs not-yet-managed
└── Gradual cleanup of non-standard configurations
```

### Key Phrases for Interview

```
"Import, don't recreate" = Existing resources-ஐ state-ல் add, destroy-recreate இல்லை
"Identity plan" = Plan shows NO changes after import (proves state matches reality)
"Blast radius" = How much one mistake can damage (smaller = better)
"State surgery" = Moving resources between states (dangerous, but necessary)
"Moved blocks" = Refactor without destroy (Terraform's rename mechanism)
"Brownfield" = Existing infrastructure (vs greenfield = new from scratch)
```

### Tamil Memory Aids

```
Import = புகைப்படம் எடுத்து album-ல் add (photo → album)
Moved block = Room rename, demolish இல்லை (rename, not demolish)
State split = பெரிய புத்தகத்தை chapters-ஆக பிரிப்பது (big book → chapters)
Drift = யாரோ மாற்றம் செய்தது, Terraform-க்கு தெரியாது (unknown change)
Blast radius = ஒரு தவறு எவ்வளவு damage செய்யும் (damage range)
```

---

## ⚡ Quick Hands-on

### Lab Setup

```bash
# SSH to lab server
ssh root@203.57.85.108

# Create lab directory
mkdir -p ~/tf-lab/advanced && cd ~/tf-lab/advanced
```

### Exercise 1: Import Block (Terraform 1.5+ Way)

```bash
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-project"
  region  = "us-central1"
}

# STEP 1: Write the import block
# This tells Terraform: "This resource already exists in GCP"
import {
  to = google_compute_network.existing_vpc
  id = "projects/my-project/global/networks/existing-vpc"
}

# STEP 2: Write the resource block matching the existing config
# Use: gcloud compute networks describe existing-vpc --format=json
# to get the current configuration
resource "google_compute_network" "existing_vpc" {
  name                    = "existing-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
  # Match ALL existing settings exactly!
}

# Import multiple resources at once
import {
  to = google_compute_subnetwork.existing_subnet
  id = "projects/my-project/regions/us-central1/subnetworks/existing-subnet"
}

resource "google_compute_subnetwork" "existing_subnet" {
  name          = "existing-subnet"
  network       = google_compute_network.existing_vpc.id
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-central1"
}

# Import a Cloud SQL instance
import {
  to = google_sql_database_instance.existing_db
  id = "projects/my-project/instances/existing-db"
}

resource "google_sql_database_instance" "existing_db" {
  name             = "existing-db"
  database_version = "POSTGRES_14"
  region           = "us-central1"
  
  deletion_protection = true  # CRITICAL: don't accidentally delete!

  settings {
    tier = "db-custom-2-4096"
    backup_configuration {
      enabled = true
    }
  }
}
EOF

echo "Import blocks created!"
echo ""
echo "Workflow:"
echo "  1. terraform init"
echo "  2. terraform plan  → should show 'import' actions, NO 'destroy'"
echo "  3. terraform apply → imports into state"
echo "  4. terraform plan  → should show 'No changes' (identity!)"
```

### Exercise 2: Generate Config from Existing Resources

```bash
# Terraform 1.5+ can generate HCL from existing resources!
cat > import_only.tf << 'EOF'
# Use -generate-config-out when you don't know the exact config
# Terraform will write the HCL for you!

import {
  to = google_compute_instance.unknown_vm
  id = "projects/my-project/zones/us-central1-a/instances/mystery-vm"
}

# Don't write the resource block — let Terraform generate it!
# Run: terraform plan -generate-config-out=generated.tf
# Terraform creates generated.tf with the full resource config
EOF

echo "To generate config automatically:"
echo "  terraform plan -generate-config-out=generated.tf"
echo ""
echo "This creates generated.tf with the exact current config of the resource."
echo "Review it, clean it up, then import normally."
```

### Exercise 3: Moved Blocks (Refactoring)

```bash
mkdir -p refactor-demo

cat > refactor-demo/before.tf << 'EOF'
# BEFORE: Resources are flat (not in modules)
resource "google_compute_network" "main" {
  name                    = "production-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "app" {
  name          = "app-subnet"
  network       = google_compute_network.main.id
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-central1"
}

resource "google_compute_instance" "web_1" {
  name         = "web-server-1"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.app.id
  }
}

resource "google_compute_instance" "web_2" {
  name         = "web-server-2"
  machine_type = "e2-medium"
  zone         = "us-central1-b"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.app.id
  }
}
EOF

cat > refactor-demo/after.tf << 'EOF'
# AFTER: Organized into modules with moved blocks
# No destroy+recreate — Terraform just updates the state address

# MOVED BLOCKS — tell Terraform about the refactoring
moved {
  from = google_compute_network.main
  to   = module.networking.google_compute_network.main
}

moved {
  from = google_compute_subnetwork.app
  to   = module.networking.google_compute_subnetwork.app
}

moved {
  from = google_compute_instance.web_1
  to   = module.compute.google_compute_instance.web["web-1"]
}

moved {
  from = google_compute_instance.web_2
  to   = module.compute.google_compute_instance.web["web-2"]
}

# Now use modules
module "networking" {
  source = "./modules/networking"
}

module "compute" {
  source     = "./modules/compute"
  subnet_id  = module.networking.subnet_id
  instances  = {
    "web-1" = { zone = "us-central1-a" }
    "web-2" = { zone = "us-central1-b" }
  }
}
EOF

echo "Refactoring demo created!"
echo ""
echo "What happens:"
echo "  terraform plan with moved blocks →"
echo "    'google_compute_instance.web_1 has moved to"
echo "     module.compute.google_compute_instance.web[\"web-1\"]'"
echo "  NO destroy! NO recreate! Just state address change."
```

### Exercise 4: State Splitting

```bash
cat > state_split_guide.sh << 'EOF'
#!/bin/bash
# STATE SPLITTING — Moving resources from monolith state to component states
# 
# Scenario: One state file has everything. Split into networking + compute.
#
# STEP 1: List all resources in current state
terraform state list

# STEP 2: Move networking resources to separate state
# Create new directory for networking
mkdir -p components/networking
cd components/networking

# Write backend config for new state
cat > backend.tf << 'INNER_EOF'
terraform {
  backend "gcs" {
    bucket = "my-tf-state"
    prefix = "components/networking"
  }
}
INNER_EOF

# Write the networking resources (copy from monolith)
# ... main.tf with VPC, subnets, firewalls ...

terraform init

# STEP 3: Move resources from old state to new state
# From the ORIGINAL directory:
cd /original/terraform
terraform state mv \
  -state-out=../components/networking/terraform.tfstate \
  google_compute_network.main \
  google_compute_network.main

terraform state mv \
  -state-out=../components/networking/terraform.tfstate \
  google_compute_subnetwork.app \
  google_compute_subnetwork.app

# STEP 4: Verify — plan should show NO changes in both states
cd ../components/networking
terraform plan  # Should be: "No changes"

cd /original/terraform
terraform plan  # Should be: removed resources no longer in plan

# STEP 5: Cross-reference using remote state
# In compute state, reference networking:
cat > data.tf << 'INNER_EOF'
data "terraform_remote_state" "networking" {
  backend = "gcs"
  config = {
    bucket = "my-tf-state"
    prefix = "components/networking"
  }
}

# Use: data.terraform_remote_state.networking.outputs.vpc_id
INNER_EOF

echo "State split complete!"
echo "Monolith: was 200 resources, now split into focused states"
echo "Each state: 30-50 resources, fast plan, limited blast radius"
EOF

chmod +x state_split_guide.sh
echo "State split guide created: ./state_split_guide.sh"
```

### Exercise 5: Drift Detection & Remediation

```bash
cat > drift_check.sh << 'EOF'
#!/bin/bash
# DRIFT DETECTION SCRIPT
# Run via cron or CI schedule to detect manual changes

set -e

ENVIRONMENTS=("dev" "staging" "prod")
ALERT_WEBHOOK="${SLACK_WEBHOOK_URL}"

for ENV in "${ENVIRONMENTS[@]}"; do
  echo "=== Checking drift in: ${ENV} ==="
  
  cd "environments/${ENV}"
  terraform init -input=false > /dev/null 2>&1
  
  # -detailed-exitcode: 0=no changes, 1=error, 2=changes detected
  terraform plan -detailed-exitcode -input=false -no-color > plan_output.txt 2>&1
  EXIT_CODE=$?
  
  if [ ${EXIT_CODE} -eq 2 ]; then
    echo "⚠️  DRIFT DETECTED in ${ENV}!"
    
    # Alert via webhook
    if [ -n "${ALERT_WEBHOOK}" ]; then
      DRIFT_SUMMARY=$(grep -E "Plan:|to add|to change|to destroy" plan_output.txt | head -5)
      curl -s -X POST "${ALERT_WEBHOOK}" \
        -H "Content-Type: application/json" \
        -d "{\"text\": \"🚨 Drift in *${ENV}*:\n\`\`\`${DRIFT_SUMMARY}\`\`\`\"}"
    fi
    
    # Options for remediation:
    # Option A: Auto-apply (DANGEROUS — only for dev)
    # if [ "${ENV}" == "dev" ]; then
    #   terraform apply -auto-approve plan_output.tfplan
    # fi
    
    # Option B: Create PR with drift fix (RECOMMENDED)
    # git checkout -b "fix/drift-${ENV}-$(date +%Y%m%d)"
    # terraform apply -auto-approve
    # git add . && git commit -m "fix: reconcile drift in ${ENV}"
    # git push && create PR
    
  elif [ ${EXIT_CODE} -eq 0 ]; then
    echo "✅ No drift in ${ENV}"
  else
    echo "❌ Error checking ${ENV}"
  fi
  
  cd - > /dev/null
done
EOF

chmod +x drift_check.sh
echo "Drift detection script created: ./drift_check.sh"
```

---

## 🔥 Scenario Challenge

### Scenario: "200 Manually-Created Resources — Bring Under Terraform"

**The Interview Question:**
"You've joined a company with 200+ cloud resources created manually via console. No IaC exists. How do you bring them under Terraform management?"

**Senior Architect Answer (with timeline):**

```
┌────────────────────────────────────────────────────────────────┐
│        BROWNFIELD IMPORT — COMPLETE STRATEGY                    │
│                                                                  │
│  WEEK 1: DISCOVERY & PLANNING                                   │
│  ├── Export all resources:                                      │
│  │   gcloud asset search-all-resources --project=X > all.json  │
│  ├── Categorize (network: 30, compute: 80, data: 40, IAM: 50) │
│  ├── Map dependencies (VPC → subnets → instances)              │
│  ├── Identify "untouchable" (prod DB, critical VMs)            │
│  └── Create import plan with batches                           │
│                                                                  │
│  WEEK 2-3: IMPORT (batch by type)                               │
│  ├── Batch 1: Networking (VPC, subnets, firewalls)             │
│  │   → Lowest risk, no data involved                           │
│  ├── Batch 2: IAM (roles, service accounts)                    │
│  │   → Important for governance                                │
│  ├── Batch 3: Compute (VMs, GKE, Cloud Run)                   │
│  │   → Medium risk, set deletion_protection                    │
│  ├── Batch 4: Data (SQL, GCS, Redis)                          │
│  │   → Highest risk, extra validation                          │
│  └── Each batch: write HCL → import → plan (no changes) → PR  │
│                                                                  │
│  WEEK 4: ORGANIZE & GOVERN                                      │
│  ├── Organize into modules (moved blocks)                      │
│  ├── Split state by domain                                     │
│  ├── Set up CI/CD pipeline                                     │
│  ├── Enable drift detection                                    │
│  └── Document: what's managed vs not-yet-managed               │
│                                                                  │
│  KEY RISKS & MITIGATIONS:                                       │
│  ├── Risk: Import wrong config → drift on first plan           │
│  │   Fix: Always verify "terraform plan = no changes"          │
│  ├── Risk: Miss a resource → Terraform tries to create new     │
│  │   Fix: Compare resource count before and after              │
│  ├── Risk: Accidental deletion during import                   │
│  │   Fix: deletion_protection = true on EVERYTHING first       │
│  └── Risk: Team makes manual changes during import period      │
│      Fix: Announce "config freeze" during import weeks         │
└────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Real Project Reference: EB Brownfield Import

### EB Project — Importing 200+ GCP Resources

```
┌────────────────────────────────────────────────────────────────┐
│           EB PROJECT — BROWNFIELD IMPORT                        │
│                                                                  │
│  Context:                                                       │
│  • 200+ GCP resources created manually over 2 years            │
│  • Mix of console clicks and gcloud commands                    │
│  • No documentation of original intent                         │
│  • Production workloads running on these resources              │
│                                                                  │
│  Our Approach:                                                  │
│                                                                  │
│  1. INVENTORY:                                                  │
│     gcloud asset search-all-resources \                         │
│       --project=eb-prod \                                       │
│       --asset-types="compute.googleapis.com/*,                  │
│                      sqladmin.googleapis.com/*,                  │
│                      storage.googleapis.com/*" \                │
│       --format=json > inventory.json                           │
│     Result: 247 resources identified                           │
│                                                                  │
│  2. BATCH IMPORT PLAN:                                          │
│     Week 1: VPC + Subnets + Firewalls (32 resources)           │
│     Week 2: GKE clusters + Node pools (45 resources)           │
│     Week 3: Cloud SQL + GCS (28 resources)                     │
│     Week 4: IAM + Service Accounts (52 resources)              │
│     Week 5: Remaining (90 resources)                           │
│                                                                  │
│  3. TECHNIQUE (per batch):                                      │
│     a. terraform plan -generate-config-out=generated.tf        │
│        (auto-generates HCL from existing resources)            │
│     b. Review + clean up generated code                        │
│     c. terraform plan → verify "No changes"                    │
│     d. PR with "Import batch N: {description}"                 │
│     e. CI validates plan is clean                              │
│                                                                  │
│  4. RESULT:                                                     │
│     • 247 resources imported in 5 weeks                        │
│     • Zero downtime during import                              │
│     • 3 drift findings fixed (manual changes detected)         │
│     • Going forward: all changes via CI/CD pipeline            │
│                                                                  │
│  LESSON LEARNED:                                                │
│     "Start with deletion_protection on EVERYTHING.             │
│      We almost deleted a prod SQL instance in week 3           │
│      because the import didn't match perfectly."                │
└────────────────────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A

### Q1: "You have 200 manually-created resources. How do you bring them under Terraform?"

**English Answer:**
"I follow a four-phase approach: discover, import, organize, govern.

**Discovery:** Export all resources using cloud asset inventory (`gcloud asset search-all-resources`). Categorize by type and map dependencies. This gives me the full picture before touching anything.

**Import:** Batch by resource type — networking first (lowest risk), data last (highest risk). For each batch: write HCL matching current config (or use `-generate-config-out`), add import blocks, run plan (must show 'no changes'), submit as PR. Critical: set `deletion_protection = true` on everything before starting.

**Organize:** Once imported, use moved blocks to organize into modules. Split state by domain (networking, compute, data) to reduce blast radius.

**Govern:** Set up CI/CD pipeline, enable drift detection, document what's managed. Going forward, all changes via Terraform + PR.

At EB, we imported 247 resources in 5 weeks with zero downtime. The key was batching and always validating with a 'no changes' plan."

**தமிழ் Answer:**
"நான்கு-phase approach follow செய்கிறேன்: discover, import, organize, govern.

**Discovery:** Cloud asset inventory use செய்து எல்லா resources-ஐயும் export. Type-ப்படி categorize, dependencies map.

**Import:** Resource type-ப்படி batch — networking first (lowest risk), data last (highest risk). ஒவ்வொரு batch-க்கும்: HCL எழுது, import blocks add, plan run ('no changes' வர வேண்டும்), PR submit. `deletion_protection = true` எல்லாவற்றிலும் first.

**Organize:** Import ஆன பிறகு, moved blocks use செய்து modules-ல் organize. State-ஐ domain-ப்படி split.

**Govern:** CI/CD pipeline setup, drift detection enable. Going forward, எல்லா changes Terraform + PR வழியாக மட்டுமே.

EB-ல், 247 resources 5 weeks-ல் import — zero downtime. Key: batching + 'no changes' plan validation."

---

### Q2: "How do you refactor Terraform code without destroying resources?"

**English Answer:**
"Terraform has two mechanisms for safe refactoring:

1. **Moved blocks** (preferred, Terraform 1.1+): Declare in HCL where a resource moved from and to. Terraform updates state without destroying anything.

```hcl
moved {
  from = google_compute_instance.web
  to   = module.compute.google_compute_instance.web
}
```

2. **terraform state mv** (older approach): Manually move resources in state. Works but not declarative, not reviewable in PR.

The moved block approach is superior because:
- It's in code (reviewable in PR)
- It's declarative (team can see what happened)
- It's idempotent (apply multiple times safely)
- It works across modules and for_each refactoring

Common refactoring patterns:
- Flat resources → modules (most common)
- count → for_each (better keys)
- Rename resources (e.g., after naming convention change)
- Split one module into two"

**தமிழ் Answer:**
"Terraform-ல் safe refactoring-க்கு இரண்டு mechanisms:

1. **Moved blocks** (preferred): HCL-ல் resource எங்கிருந்து எங்கு move ஆனது என்று declare. Terraform state update — destroy இல்லை.

2. **terraform state mv** (older): Manually state-ல் move. Works ஆனால் declarative இல்லை, PR-ல் review செய்ய முடியாது.

Moved block superior:
- Code-ல் இருக்கிறது (PR-ல் reviewable)
- Declarative (team-க்கு தெரியும்)
- Idempotent (safely multiple times apply)

Common patterns:
- Flat resources → modules
- count → for_each
- Resource rename
- One module → two modules"

---

### Q3: "When should you split Terraform state?"

**English Answer:**
"Split state when you see these signals:

1. **Plan takes too long:** If `terraform plan` takes >2 minutes, you have too many resources in one state. Each plan refreshes ALL resources.

2. **Team conflicts:** If developers frequently hit state lock conflicts, they're working on unrelated resources in the same state.

3. **Blast radius concern:** If a mistake in networking code could affect database resources because they share state, split them.

4. **Ownership boundaries:** When different teams own different infrastructure layers (platform team = networking, app team = compute), split along ownership lines.

**How to split:**
- Use `terraform state mv` to move resources to new state files
- Set up `terraform_remote_state` data sources for cross-state references
- Verify: plan in both states shows 'no changes'

**When NOT to split:** If resources are tightly coupled (e.g., a VPC and its subnets), keep them together. Over-splitting creates dependency management headaches."

**தமிழ் Answer:**
"இந்த signals இருக்கும்போது state split செய்ய வேண்டும்:

1. **Plan too long:** `terraform plan` >2 minutes ஆனால், ஒரு state-ல் too many resources.

2. **Team conflicts:** Developers frequently state lock conflicts hit. Unrelated resources same state-ல்.

3. **Blast radius:** Networking code mistake database resources-ஐ affect செய்ய முடிந்தால் — split.

4. **Ownership boundaries:** Different teams different layers own — ownership lines-ல் split.

**எப்படி split:** `terraform state mv` + `terraform_remote_state` data sources for cross-references.

**எப்போது split வேண்டாம்:** Resources tightly coupled (VPC + subnets) — together வை. Over-splitting dependency headaches create."

---

### Q4: "Terraform vs Pulumi vs Crossplane — how do you choose?"

**English Answer:**
"My decision framework:

**Terraform** (default choice): Multi-cloud, mature ecosystem, largest community, every cloud resource supported. HCL is domain-specific but learnable. Best for: most teams, especially operations/infrastructure teams.

**Pulumi** (dev-heavy teams): Real programming languages (Python, TypeScript). Better for teams who hate DSLs and want loops/conditionals/testing natively. Trade-off: smaller community, state management options are more limited.

**Crossplane** (K8s-native): Kubernetes CRDs for infrastructure. Auto-reconciliation (if someone makes a manual change, Crossplane reverts it automatically). Best for: teams that are 100% Kubernetes-native and want GitOps for infrastructure.

**My recommendation:** Start with Terraform. Only switch if you have a STRONG reason — the ecosystem, tooling, and hiring advantage of Terraform is massive. At both TVS and EB, Terraform was the clear choice because of multi-cloud support and team familiarity."

**தமிழ் Answer:**
"என் decision framework:

**Terraform** (default choice): Multi-cloud, mature ecosystem, largest community. Operations/infrastructure teams-க்கு best.

**Pulumi** (dev-heavy teams): Real programming languages. DSLs hate செய்யும் teams-க்கு. Trade-off: smaller community.

**Crossplane** (K8s-native): Kubernetes CRDs for infrastructure. Auto-reconciliation. 100% K8s-native teams-க்கு best.

**Recommendation:** Terraform-லிருந்து start. STRONG reason இருந்தால் மட்டுமே switch. Ecosystem, tooling, hiring advantage massive. TVS, EB இரண்டிலும் Terraform clear choice."

---

### Q5: "How do you handle Terraform at large scale (1000+ resources)?"

**English Answer:**
"Large-scale Terraform requires architectural patterns beyond basic usage:

1. **State segregation:** Split by domain (networking, compute, data, IAM). No single state has more than 100-150 resources. Cross-reference via `terraform_remote_state`.

2. **Module registry:** Internal module registry (or Git tags) for versioned, tested, approved modules. Teams consume modules, don't write raw resources.

3. **Blast radius controls:** Changes to networking require platform team review. Changes to prod require 2 approvals. Automated checks flag PRs that modify >20 resources.

4. **Parallel execution:** Different states can be planned/applied in parallel since they have separate locks.

5. **Governance layer:** OPA policies, mandatory tags, allowed resource types. The platform team defines WHAT you can create, application teams define HOW MANY.

6. **Self-service:** Internal portal where developers request infrastructure via forms → generates Terraform PR → auto-approves for dev → manual approve for prod.

This is the difference between 'using Terraform' and 'running Terraform as a platform'."

**தமிழ் Answer:**
"Large-scale Terraform-க்கு basic usage-க்கு மேலான architectural patterns தேவை:

1. **State segregation:** Domain-ப்படி split. ஒரு state-ல் 100-150 resources max. `terraform_remote_state` வழியாக cross-reference.

2. **Module registry:** Internal registry — versioned, tested, approved modules. Teams modules consume செய்கின்றன.

3. **Blast radius controls:** Networking changes platform team review. Prod 2 approvals. >20 resources modify PRs flag.

4. **Parallel execution:** Different states parallel plan/apply.

5. **Governance:** OPA policies, mandatory tags, allowed resource types.

6. **Self-service:** Internal portal — forms → Terraform PR generate → auto-approve dev → manual approve prod.

'Terraform use செய்வது' vs 'Terraform-ஐ platform-ஆக run செய்வது' — இதுதான் வேறுபாடு."

---

## ✅ Self-Check

### Knowledge Verification

- [ ] Can you explain the difference between `terraform import` and import blocks?
- [ ] Can you write a moved block from memory?
- [ ] Do you know when and how to split state?
- [ ] Can you describe the brownfield import strategy in 4 phases?
- [ ] Can you compare Terraform vs Pulumi vs Crossplane?
- [ ] Can you explain blast radius and how to reduce it?

### Interview Readiness

- [ ] Can you answer the "200 manual resources" question with a timeline?
- [ ] Can you explain refactoring without destruction?
- [ ] Can you design a large-scale Terraform architecture?
- [ ] Can you talk about the EB import project with specifics?

### Command Competence

```bash
# Can you use these from memory?
terraform import google_compute_instance.vm projects/X/zones/Z/instances/NAME
terraform state list
terraform state mv old_address new_address
terraform state rm resource_address  # Remove from state (NOT delete resource)
terraform plan -generate-config-out=generated.tf
terraform plan -detailed-exitcode  # Exit 2 = changes detected
```

---

## 📋 Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│           ADVANCED TERRAFORM — CHEAT SHEET                      │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  IMPORT:                                                        │
│    import { to = resource.name; id = "cloud-id" }              │
│    terraform plan → "will import"                               │
│    terraform plan -generate-config-out=gen.tf                   │
│                                                                  │
│  MOVED (refactor without destroy):                              │
│    moved { from = old.name; to = new.name }                    │
│    terraform plan → "has moved to" (not destroy+create)        │
│                                                                  │
│  STATE SURGERY:                                                 │
│    terraform state list         # See all resources             │
│    terraform state mv A B       # Rename in state              │
│    terraform state rm X         # Remove from state (not cloud)│
│    terraform state pull > backup.json  # Backup state          │
│                                                                  │
│  DRIFT:                                                         │
│    terraform plan -detailed-exitcode                            │
│    Exit 0 = clean, Exit 2 = drift                              │
│                                                                  │
│  LARGE SCALE PATTERNS:                                          │
│    • Split state by domain (< 150 resources each)              │
│    • Module registry (versioned, tested)                       │
│    • Cross-state: terraform_remote_state data source           │
│    • Blast radius: limit PR scope, approval gates              │
│                                                                  │
│  BROWNFIELD IMPORT PHASES:                                      │
│    1. Discover (asset inventory)                               │
│    2. Import (batch by type, validate "no changes")            │
│    3. Organize (modules, state split)                          │
│    4. Govern (CI/CD, drift detection)                          │
│                                                                  │
│  ARCHITECT MINDSET:                                             │
│    "Import, don't recreate"                                    │
│    "deletion_protection = true on EVERYTHING first"            │
│    "Identity plan before moving forward"                       │
│    "Batch imports, never big-bang"                              │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```
