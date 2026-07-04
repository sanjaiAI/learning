# Module 02: State Management

---

## 📖 The Story

**English:**

Think of state file like a property document (பத்திரம்). You built a house (cloud resources). The document records what you own. Without it, you can't prove ownership or make changes properly.

Imagine you built a house in your village. You have the construction plan (`.tf` code = what you WANT). But you also have the property document — the legal record of what you ACTUALLY built. If you want to add a new room, you check the document first: "What exists already?" Then you plan the addition.

Now imagine two family members try to modify the house at the same time — one adds a room, another demolishes a wall. Chaos! So you add a rule: only ONE person can hold the property document at a time (this is **state locking**). And you keep the document in a bank locker (this is **remote backend**), not under your mattress.

**தமிழ்:**

State file என்பது பத்திரம் (property document) மாதிரி. நீ ஒரு வீடு கட்டினாய் (cloud resources). பத்திரம் உனக்கு என்ன சொந்தம் என்று பதிவு செய்கிறது. அது இல்லாமல், உரிமையை நிரூபிக்கவோ, மாற்றங்கள் செய்யவோ முடியாது.

நீ உன் கிராமத்தில் ஒரு வீடு கட்டினாய் என்று நினை. கட்டுமான plan இருக்கிறது (`.tf` code = நீ என்ன வேண்டும் என்று நினைப்பது). ஆனால் பத்திரமும் இருக்கிறது — நீ நிஜமாக என்ன கட்டினாய் என்ற சட்டப் பதிவு. புதிய அறை சேர்க்கணும் என்றால், முதலில் பத்திரத்தை பார்க்கிறாய்: "ஏற்கனவே என்ன இருக்கு?" பிறகு சேர்ப்பதை plan செய்கிறாய்.

இப்போது இரண்டு குடும்ப உறுப்பினர்கள் ஒரே நேரத்தில் வீட்டை மாற்ற முயற்சிக்கிறார்கள் என்று நினை — ஒருவர் அறை சேர்க்கிறார், மற்றவர் சுவர் இடிக்கிறார். குழப்பம்! எனவே ஒரு விதி வைக்கிறாய்: ஒரு நேரத்தில் ஒரே ஒருவர் மட்டுமே பத்திரத்தை கையில் வைக்கலாம் (இதுதான் **state locking**). பத்திரத்தை bank locker-ல் வைக்கிறாய் (இதுதான் **remote backend**), தலையணைக்கு அடியில் அல்ல.

---

## 📊 Concepts: What is Terraform State?

**English:**

State = Terraform's memory of what exists in the cloud. It maps your code to real resources. Without state, Terraform doesn't know what it created.

Three-way comparison:
- `.tf` code = What you WANT (desired state)
- `terraform.tfstate` = What Terraform KNOWS exists (recorded state)
- Cloud reality = What ACTUALLY exists (real state)

`terraform plan` compares all three to figure out what actions to take.

**தமிழ்:**

State = Cloud-ல் என்ன இருக்கிறது என்ற Terraform-ன் நினைவகம். உன் code-ஐ நிஜ resources-உடன் map செய்கிறது. State இல்லாமல், Terraform-க்கு தான் என்ன உருவாக்கியது என்று தெரியாது.

மூன்று வழி ஒப்பீடு:
- `.tf` code = நீ என்ன வேண்டும் (desired state / விரும்பிய நிலை)
- `terraform.tfstate` = Terraform-க்கு என்ன இருக்கு என்று தெரியும் (பதிவு செய்யப்பட்ட நிலை)
- Cloud நிஜம் = உண்மையில் என்ன இருக்கிறது (நிஜ நிலை)

`terraform plan` மூன்றையும் ஒப்பிட்டு, என்ன action எடுக்கணும் என்று கண்டுபிடிக்கும்.

```
┌──────────────────────────────────────────────────────────────────┐
│                THREE-WAY COMPARISON                               │
│                                                                  │
│   .tf Code          terraform.tfstate        Cloud Reality       │
│   (Desired)    ←→   (Known/Recorded)    ←→   (Actual)           │
│                                                                  │
│   "I want 3         "Last time I saw        "Right now there    │
│    VMs with          3 VMs with              are 3 VMs,         │
│    4 CPU"            4 CPU"                  4 CPU each"        │
│                                                                  │
│   terraform plan = compare all three → decide actions            │
└──────────────────────────────────────────────────────────────────┘
```

### What's Inside a State File?

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "resources": [
    {
      "mode": "managed",
      "type": "azurerm_resource_group",
      "name": "main",
      "instances": [
        {
          "attributes": {
            "id": "/subscriptions/xxx/resourceGroups/rg-prod",
            "name": "rg-prod",
            "location": "eastus"
          },
          "dependencies": []
        }
      ]
    }
  ]
}
```

---

## 🧠 Byheart for Interview — State Basics

```
English:
1. State = mapping between code and real cloud resources
2. State file format: JSON (terraform.tfstate)
3. Three-way comparison: Code vs State vs Cloud Reality
4. Without state, Terraform would CREATE everything fresh every time
5. State contains: resource IDs, attributes, metadata, dependencies
6. State can contain SECRETS (DB passwords, keys) — never commit to Git!
7. terraform refresh = sync state with cloud (deprecated → use plan -refresh-only)
8. State serial number increments on every modification

தமிழ்:
1. State = code-க்கும் நிஜ cloud resources-க்கும் இடையே mapping
2. State file format: JSON (terraform.tfstate)
3. மூன்று வழி ஒப்பீடு: Code vs State vs Cloud நிஜம்
4. State இல்லாமல், Terraform ஒவ்வொரு தடவையும் புதிதாக CREATE செய்ய முயலும்
5. State-ல்: resource IDs, attributes, metadata, dependencies
6. State-ல் SECRETS இருக்கலாம் (DB passwords, keys) — Git-க்கு commit செய்யாதே!
7. terraform refresh = state-ஐ cloud-உடன் sync செய் (deprecated → plan -refresh-only)
8. ஒவ்வொரு மாற்றத்திலும் State serial number அதிகரிக்கும்
```

---

## ⚡ Quick Hands-on — State Basics

```bash
ssh root@203.57.85.108

mkdir -p ~/tf-lab/02-state && cd ~/tf-lab/02-state

cat > main.tf << 'EOF'
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "demo" {
  content  = "State demo file - understanding state"
  filename = "${path.module}/demo.txt"
}

resource "local_file" "config" {
  content  = "Config file for state lab"
  filename = "${path.module}/config.txt"
}
EOF

terraform init
terraform apply -auto-approve

# Examine the state
terraform state list
terraform state show local_file.demo
cat terraform.tfstate | python3 -m json.tool | head -50
# Notice: serial number, resource IDs, all attributes stored
```

---

## 📊 Remote Backends

**English:**

Why Remote? NEVER use local state in teams! Local state on your laptop means:
- No locking → two people corrupt state simultaneously
- Lost if laptop dies
- No sharing with team members
- No audit trail

Remote backend stores state in a durable, shared, lockable, versioned location.

**தமிழ்:**

ஏன் Remote? Team-ல் local state ஒருபோதும் பயன்படுத்தாதே! உன் laptop-ல் local state என்றால்:
- Locking இல்லை → இரண்டு பேர் ஒரே நேரத்தில் state-ஐ கெடுக்கலாம்
- Laptop போனால் state போச்சு
- Team members-உடன் share முடியாது
- Audit trail இல்லை

Remote backend state-ஐ நீடித்த, பகிரக்கூடிய, lock செய்யக்கூடிய, version-controlled இடத்தில் சேமிக்கும்.

| Feature | Azure Blob | GCS | S3 |
|---------|-----------|-----|-----|
| Storage | Blob Storage | Cloud Storage | S3 Bucket |
| Locking | Blob Lease | Built-in | DynamoDB table |
| Versioning | Blob versioning | Object versioning | S3 versioning |
| Encryption | SSE + CMK | CMEK | SSE-KMS |
| Auth | SPN/MSI/SAS | SA/WIF | IAM Role |

### Azure Backend (Production Pattern)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstateproduction"
    container_name       = "tfstate"
    key                  = "production/aks-cluster.tfstate"
    
    # Authentication: use ARM_* env vars or Azure CLI
    # Never hardcode credentials in backend block!
  }
}

# Bootstrap (create backend storage BEFORE using it):
# az group create -n rg-terraform-state -l eastus
# az storage account create -n sttfstateproduction -g rg-terraform-state \
#   -l eastus --sku Standard_GRS --min-tls-version TLS1_2
# az storage container create -n tfstate --account-name sttfstateproduction
# az storage account blob-service-properties update \
#   --account-name sttfstateproduction --enable-versioning true
```

### GCP Backend

```hcl
terraform {
  backend "gcs" {
    bucket = "myproject-terraform-state"
    prefix = "production/gke"
  }
}

# Bootstrap:
# gsutil mb -l us-central1 -b on gs://myproject-terraform-state
# gsutil versioning set on gs://myproject-terraform-state
```

### AWS Backend

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/eks.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # Required for locking!
    encrypt        = true
  }
}
```

### Partial Backend Config (CI/CD Pattern)

```hcl
# backend.tf — no secrets in code
terraform {
  backend "azurerm" {}
}

# Pass config via CLI or file:
# terraform init -backend-config=backend.hcl
# terraform init -backend-config="storage_account_name=sttfstate" \
#                -backend-config="container_name=tfstate" ...
```

```hcl
# backend.hcl (gitignored or from CI secrets)
resource_group_name  = "rg-terraform-state"
storage_account_name = "sttfstateproduction"
container_name       = "tfstate"
key                  = "production/aks.tfstate"
```

### 🧠 Byheart — Remote Backends

```
English:
1. Remote backends: Azure Blob, GCS, S3, Terraform Cloud, Consul, HTTP
2. Azure: Storage Account + Container + Key (blob path) + Blob Lease locking
3. GCP: GCS Bucket + Prefix + built-in locking
4. AWS: S3 Bucket + DynamoDB table (separate resource for locking!)
5. Always enable VERSIONING (rollback on corruption!)
6. Backend config can't use variables — must be literal or -backend-config
7. Migrate local→remote: add backend block → terraform init (auto-migrates)
8. Backend bootstrap = chicken-and-egg — create storage manually/separately first

தமிழ்:
1. Remote backends: Azure Blob, GCS, S3, Terraform Cloud, Consul, HTTP
2. Azure: Storage Account + Container + Key + Blob Lease locking
3. GCP: GCS Bucket + Prefix + built-in locking
4. AWS: S3 Bucket + DynamoDB table (locking-க்கு தனி resource!)
5. எப்போதும் VERSIONING enable செய் (corruption-ல் rollback!)
6. Backend config-ல் variables வேலை செய்யாது — literal அல்லது -backend-config
7. Local→remote: backend block சேர் → terraform init (auto-migrate)
8. Backend bootstrap = கோழி-முட்டை பிரச்சனை — storage-ஐ முதலில் manually உருவாக்கு
```

---

## 🔒 State Locking — Deep Dive

**English:**

State locking prevents concurrent modification. When `terraform apply` runs, it acquires a lock. If another person/pipeline tries to apply, they get "state locked" error.

Think of it like a checkout system in a library — only one person borrows the book at a time. Others must wait until it's returned.

**தமிழ்:**

State locking ஒரே நேரத்தில் state மாற்றத்தை தடுக்கும். `terraform apply` ஓடும்போது lock acquire செய்யும். வேறொரு நபர்/pipeline apply செய்ய முயன்றால், "state locked" error வரும்.

Library-ல் புத்தகம் எடுப்பது போல — ஒரு நேரத்தில் ஒருவர் மட்டும் எடுக்கலாம். மற்றவர்கள் திருப்பும் வரை காத்திருக்க வேண்டும்.

### How Locking Works Per Backend

```
┌─────────────────────────────────────────────────────────────────┐
│ AZURE BLOB STORAGE LOCKING                                      │
│                                                                 │
│ 1. terraform apply starts                                       │
│ 2. Acquires 60-second BLOB LEASE on state file                 │
│ 3. Lease auto-renews every 15 seconds during operation         │
│ 4. If apply crashes → lease expires after 60s (auto-unlock)    │
│ 5. Force unlock: break lease via Azure CLI                     │
│    az storage blob lease break --blob-name prod.tfstate ...    │
├─────────────────────────────────────────────────────────────────┤
│ AWS S3 + DYNAMODB LOCKING                                       │
│                                                                 │
│ 1. terraform apply starts                                       │
│ 2. Writes lock record to DynamoDB (LockID = state file path)   │
│ 3. If another apply attempts → ConditionalCheckFailed          │
│ 4. If apply crashes → lock stays FOREVER (must force-unlock)   │
│ 5. Force unlock: terraform force-unlock <LOCK_ID>              │
├─────────────────────────────────────────────────────────────────┤
│ GCS LOCKING                                                     │
│                                                                 │
│ 1. terraform apply starts                                       │
│ 2. Creates .tflock file in same bucket                         │
│ 3. Uses generation numbers for consistency                     │
│ 4. If apply crashes → lock file remains (force-unlock needed)  │
└─────────────────────────────────────────────────────────────────┘
```

### Lock Troubleshooting

```bash
# Scenario: "Error: Error locking state: Error acquiring the state lock"
# This means another process holds the lock

# Step 1: Check WHO has the lock (info in error message)
# Error shows: Lock Info: ID, Who, Operation, Created

# Step 2: Verify no one is actually running
# (Check CI/CD pipelines, ask team members)

# Step 3: Force unlock ONLY if confirmed nobody is running
terraform force-unlock <LOCK_ID_FROM_ERROR>

# Azure specific — break lease directly:
az storage blob lease break \
  --blob-name "production/aks.tfstate" \
  --container-name tfstate \
  --account-name sttfstateproduction
```

### 🧠 Byheart — Locking

```
English:
1. Locking prevents concurrent state modification
2. Auto-acquired on: plan, apply, destroy, import, state mv/rm
3. Azure: Blob lease (60s, auto-renew, auto-expire on crash)
4. AWS: DynamoDB record (NEVER auto-expires — stuck locks possible!)
5. GCS: .tflock file with generation numbers
6. force-unlock = break stuck lock (LAST RESORT — confirm nobody running!)
7. Azure lease is SELF-HEALING (expires in 60s), AWS/GCS locks are NOT
8. Terraform Cloud: queue-based locking (safest — applies run sequentially)

தமிழ்:
1. Locking = ஒரே நேரத்தில் state மாற்றத்தை தடுக்கும்
2. Auto-acquire: plan, apply, destroy, import, state mv/rm
3. Azure: Blob lease (60 வினாடி, auto-renew, crash-ல் auto-expire)
4. AWS: DynamoDB record (ஒருபோதும் auto-expire ஆகாது — stuck locks!)
5. GCS: .tflock file + generation numbers
6. force-unlock = stuck lock உடை (கடைசி வழி — யாரும் run செய்யவில்லை என்று உறுதி!)
7. Azure lease SELF-HEALING (60s-ல் expire), AWS/GCS அப்படி அல்ல
8. Terraform Cloud: queue-based locking (மிக பாதுகாப்பு — sequential apply)
```

---

## 🛠️ State Commands — Operations

**English:**

State commands let you inspect, move, import, and remove resources from state. These are your daily operations tools for managing Terraform at scale.

**தமிழ்:**

State commands state-ல் resources-ஐ inspect, move, import, remove செய்ய உதவும். இவை scale-ல் Terraform manage செய்வதற்கான அன்றாட கருவிகள்.

### Command Reference

| Command | Purpose | Dangerous? |
|---------|---------|-----------|
| `state list` | List all managed resources | No |
| `state show <addr>` | Show attributes of one resource | No |
| `state pull` | Download state JSON | No |
| `state mv` | Rename/move resource in state | ⚠️ Medium |
| `state rm` | Remove resource from state (keeps in cloud) | ⚠️ Medium |
| `state push` | Upload state (overwrite remote!) | 🔴 HIGH |
| `import` | Adopt existing resource | ⚠️ Medium |
| `force-unlock` | Break stuck lock | 🔴 HIGH |

### terraform state mv (Refactoring)

```bash
# Renamed resource in code:
# Old: resource "azurerm_resource_group" "rg" { ... }
# New: resource "azurerm_resource_group" "main" { ... }
terraform state mv azurerm_resource_group.rg azurerm_resource_group.main

# Moved resource into a module:
terraform state mv azurerm_virtual_network.vnet module.network.azurerm_virtual_network.vnet

# Moved resource to different module instance:
terraform state mv 'module.app[0]' 'module.app["api"]'
```

### Terraform 1.5+ `moved` Block (BETTER than state mv)

```hcl
# Declarative, version-controlled, works for whole team automatically
moved {
  from = azurerm_resource_group.rg
  to   = azurerm_resource_group.main
}

moved {
  from = azurerm_virtual_network.vnet
  to   = module.network.azurerm_virtual_network.vnet
}
# After successful apply, remove the moved blocks (they're one-time)
```

### terraform import

```bash
# CLI import (legacy — imperative)
terraform import azurerm_resource_group.main \
  "/subscriptions/xxx/resourceGroups/rg-production"

terraform import azurerm_kubernetes_cluster.aks \
  "/subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod"

# GCP resource import:
terraform import google_container_cluster.primary \
  "projects/myproject/locations/us-central1/clusters/gke-prod"
```

### Terraform 1.5+ `import` Block (BETTER — Declarative)

```hcl
# import.tf
import {
  to = azurerm_resource_group.main
  id = "/subscriptions/xxx/resourceGroups/rg-production"
}

import {
  to = azurerm_kubernetes_cluster.aks
  id = "/subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-prod"
}

# Then run: terraform plan -generate-config-out=generated.tf
# This generates the HCL code for you! Review, adjust, apply.
```

### terraform state rm

```bash
# Stop managing a resource WITHOUT destroying it in cloud
terraform state rm azurerm_virtual_machine.legacy

# Use case: Resource is now managed by another team/state
# Use case: Resource should exist but not be Terraform-managed anymore
# The resource STAYS in cloud — Terraform just forgets about it
```

### ⚡ Quick Hands-on — State Commands

```bash
ssh root@203.57.85.108
cd ~/tf-lab/02-state

# List all resources
terraform state list

# Show one resource's details
terraform state show local_file.demo

# Pull state to file for inspection
terraform state pull > state_backup.json
cat state_backup.json | python3 -m json.tool | head -30

# Move/rename a resource
terraform state mv local_file.config local_file.app_config
terraform state list   # Notice: config → app_config

# Import an external file
echo "I exist outside terraform" > /tmp/external.txt
cat >> main.tf << 'EOF'

resource "local_file" "imported" {
  content  = "I exist outside terraform"
  filename = "/tmp/external.txt"
}
EOF
terraform import local_file.imported /tmp/external.txt
terraform state list   # Now shows imported resource

# Remove from state (don't destroy)
terraform state rm local_file.imported
terraform state list   # Gone from state
ls /tmp/external.txt   # Still exists in filesystem!
```

---

## 📋 Workspaces

**English:**

Workspaces = same code, different state files. Like a restaurant with multiple branches — same menu (code), different inventory (state) per branch.

Good for: same infrastructure pattern with different sizes (dev=2 nodes, prod=10 nodes).
Bad for: fundamentally different architectures (prod has WAF, dev doesn't).

**தமிழ்:**

Workspaces = ஒரே code, வெவ்வேறு state files. பல கிளைகள் உள்ள உணவகம் போல — ஒரே menu (code), ஒவ்வொரு கிளைக்கும் வெவ்வேறு inventory (state).

நல்லது: ஒரே infrastructure pattern வெவ்வேறு அளவுகளில் (dev=2 nodes, prod=10 nodes).
கெடுதல்: அடிப்படையில் வேறுபட்ட architectures (prod-ல் WAF உண்டு, dev-ல் இல்லை).

```hcl
# Using workspace in code:
resource "azurerm_kubernetes_cluster" "aks" {
  name       = "aks-${terraform.workspace}"
  node_count = terraform.workspace == "prod" ? 5 : 2
  vm_size    = terraform.workspace == "prod" ? "Standard_D4s_v3" : "Standard_D2s_v3"
}

locals {
  env_config = {
    dev     = { node_count = 2, vm_size = "Standard_D2s_v3" }
    staging = { node_count = 3, vm_size = "Standard_D2s_v3" }
    prod    = { node_count = 5, vm_size = "Standard_D4s_v3" }
  }
  current = local.env_config[terraform.workspace]
}
```

### Workspaces vs Directory Structure

| Approach | When to Use | Example |
|----------|-------------|---------|
| Workspaces | Same arch, different sizes | dev/staging/prod with same resources |
| Directories | Different arch per env | prod has DR, WAF; dev doesn't |
| Terragrunt | Both + DRY code | Large teams, many environments |

### ⚡ Quick Hands-on — Workspaces

```bash
cd ~/tf-lab/02-state

terraform workspace list
terraform workspace new dev
terraform workspace new prod
terraform workspace list           # * shows current

terraform workspace select dev
terraform apply -auto-approve      # Creates in dev state

terraform workspace select prod
terraform apply -auto-approve      # Separate state!

ls terraform.tfstate.d/            # One dir per workspace
terraform workspace select default
```

---

## 🔐 State File Security

**English:**

STATE FILES CONTAIN SECRETS! When you create a database with a password, that password is stored in plaintext in state. When you create a service principal, the client_secret is in state. This is the #1 security concern with Terraform.

Think of state as a file containing all your house keys, safe combinations, and ATM PINs — written in plain text. You MUST protect it.

**தமிழ்:**

STATE FILES-ல் SECRETS இருக்கும்! Database password-உடன் உருவாக்கும்போது, அந்த password state-ல் plaintext-ல் சேமிக்கப்படும். Service principal உருவாக்கினால், client_secret state-ல் இருக்கும். இதுதான் Terraform-ன் #1 security concern.

State-ஐ உன் வீட்டு சாவிகள், safe combination, ATM PIN எல்லாம் plain text-ல் எழுதிய ஒரு file என்று நினை. நீ அதை கட்டாயம் பாதுகாக்க வேண்டும்.

### What Secrets End Up in State?

```
┌──────────────────────────────────────────────────────────┐
│ SECRETS THAT LEAK INTO STATE:                            │
│                                                          │
│ • Database passwords (azurerm_mysql_server.password)     │
│ • Storage account access keys                           │
│ • Service principal client secrets                      │
│ • TLS private keys (tls_private_key resource)           │
│ • Random passwords (random_password resource)           │
│ • API keys in resource attributes                       │
│ • Connection strings with embedded credentials          │
│ • SSH private keys                                      │
└──────────────────────────────────────────────────────────┘
```

### Protection Strategies

```hcl
# 1. Encrypt state at rest (backend-level)
# Azure: Storage Account encryption (enabled by default) + Customer Managed Key
# GCS: CMEK encryption
# S3: SSE-KMS with dedicated key

# 2. Encrypt state in transit (HTTPS — automatic with cloud backends)

# 3. Restrict access to state (IAM)
# Only CI/CD service principal should have write access
# Developers: read-only or no direct access

# 4. Use sensitive = true (hides from CLI output, NOT from state file!)
variable "db_password" {
  type      = string
  sensitive = true   # Hides in plan/apply output, but still stored in state!
}

# 5. Terraform 1.7+ — ephemeral resources (NOT stored in state!)
# ephemeral "aws_secretsmanager_secret_version" "db_pass" { ... }

# 6. External secret management pattern
# Instead of storing password in Terraform:
# - Create KeyVault/Secret Manager via Terraform
# - Generate password OUTSIDE Terraform
# - Reference via data source at runtime
```

### 🧠 Byheart — State Security

```
English:
1. State contains ALL attributes including secrets in PLAINTEXT
2. sensitive = true hides from output, NOT from state file!
3. Never commit state to Git (add *.tfstate to .gitignore)
4. Encrypt at rest: CMK on Azure, CMEK on GCP, SSE-KMS on AWS
5. Restrict IAM: only CI/CD writes state, devs get read-only or none
6. Enable audit logging on state storage (who accessed when)
7. Terraform 1.7+: ephemeral resources = not stored in state
8. State is the crown jewel — protect like a database backup

தமிழ்:
1. State-ல் secrets உட்பட எல்லா attributes-ம் PLAINTEXT-ல் இருக்கும்
2. sensitive = true output-ல் மறைக்கும், state file-ல் இல்லை!
3. State-ஐ Git-க்கு ஒருபோதும் commit செய்யாதே (*.tfstate → .gitignore)
4. Rest-ல் encrypt: Azure-CMK, GCP-CMEK, AWS SSE-KMS
5. IAM restrict: CI/CD மட்டும் state write, devs read-only அல்லது access இல்லை
6. State storage-ல் audit logging enable செய் (யார் எப்போது access செய்தார்கள்)
7. Terraform 1.7+: ephemeral resources = state-ல் store ஆகாது
8. State = crown jewel — database backup போல பாதுகாக்க வேண்டும்
```

---

## 🆘 Disaster Recovery — State Corruption & Loss

**English:**

State corruption or loss is the WORST thing that can happen in Terraform. If you lose state, Terraform doesn't know what it manages — it thinks nothing exists. Next `apply` tries to CREATE everything again, causing conflicts.

This is like losing your property document — you still own the house, but proving it and making legal changes becomes a nightmare.

**தமிழ்:**

State corruption அல்லது loss Terraform-ல் மிக மோசமான விஷயம். State இழந்தால், Terraform-க்கு என்ன manage செய்கிறது என்று தெரியாது — ஒன்றும் இல்லை என்று நினைக்கும். அடுத்த `apply` எல்லாவற்றையும் CREATE செய்ய முயலும், conflicts ஏற்படும்.

இது பத்திரம் இழப்பது போல — வீடு உன்னிடமே இருக்கு, ஆனால் நிரூபிப்பதும் சட்ட மாற்றங்களும் கொடுமையாகும்.

### Recovery Scenarios

```
┌──────────────────────────────────────────────────────────────────┐
│ SCENARIO 1: State Corrupted (invalid JSON)                       │
│ Solution:                                                        │
│   1. Restore from version history (Azure Blob versioning/GCS)   │
│   2. az storage blob download --version-id <previous_version>   │
│   3. terraform state push recovered_state.tfstate                │
├──────────────────────────────────────────────────────────────────┤
│ SCENARIO 2: State Lost Completely                                │
│ Solution:                                                        │
│   1. Create empty state: terraform init                         │
│   2. Import every resource manually:                            │
│      terraform import azurerm_resource_group.main /subs/xxx/... │
│   3. Run terraform plan — fix until "No changes"               │
│   4. Use terraform plan -generate-config-out for bulk import    │
├──────────────────────────────────────────────────────────────────┤
│ SCENARIO 3: State Drift (cloud modified outside Terraform)      │
│ Solution:                                                        │
│   1. terraform plan -refresh-only (see what drifted)            │
│   2. terraform apply -refresh-only (update state to reality)    │
│   OR: Fix cloud to match code (revert manual changes)           │
├──────────────────────────────────────────────────────────────────┤
│ SCENARIO 4: Accidental terraform destroy                         │
│ Solution:                                                        │
│   1. Resources are GONE — restore from cloud backups            │
│   2. Restore state from version history                         │
│   3. Re-apply to recreate resources                             │
│   4. Prevention: use prevent_destroy lifecycle rule             │
└──────────────────────────────────────────────────────────────────┘
```

### Prevention: Protect Critical Resources

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name = "aks-production"
  # ...

  lifecycle {
    prevent_destroy = true   # terraform destroy will ERROR for this resource
  }
}

# Also: use Azure Resource Locks / GCP Liens
resource "azurerm_management_lock" "aks_lock" {
  name       = "aks-nodelete"
  scope      = azurerm_kubernetes_cluster.aks.id
  lock_level = "CanNotDelete"
}
```

### State Recovery Hands-on

```bash
ssh root@203.57.85.108
cd ~/tf-lab/02-state

# Backup state before experiments
terraform state pull > backup_$(date +%Y%m%d).json

# Simulate state loss
mv terraform.tfstate terraform.tfstate.lost

# Now terraform thinks nothing exists:
terraform plan   # Shows "create" for everything!

# Recovery option 1: Restore backup
mv terraform.tfstate.lost terraform.tfstate

# Recovery option 2: Import resources back
# terraform import local_file.demo $(terraform state show local_file.demo -no-color 2>/dev/null | grep filename | awk '{print $3}')
```

### 🧠 Byheart — Disaster Recovery

```
English:
1. Enable versioning on state bucket = rollback to any previous version
2. State corrupted → restore from blob/object versioning
3. State lost → import all resources (painful but possible)
4. terraform state push = upload state (DANGEROUS — overwrites remote!)
5. prevent_destroy = lifecycle block to stop accidental destroy
6. Azure Resource Lock / GCP Lien = cloud-level protection
7. Always backup state before risky operations (state rm, state mv)
8. terraform plan -refresh-only = detect drift without making changes
9. Serial number in state prevents concurrent writes (optimistic locking)

தமிழ்:
1. State bucket-ல் versioning enable = எந்த முந்தைய version-க்கும் rollback
2. State corrupt → blob/object versioning-லிருந்து restore
3. State lost → எல்லா resources-ஐ import செய் (வலிக்கும் ஆனால் முடியும்)
4. terraform state push = state upload (ஆபத்தானது — remote-ஐ overwrite!)
5. prevent_destroy = accidental destroy தடுக்கும் lifecycle block
6. Azure Resource Lock / GCP Lien = cloud-level பாதுகாப்பு
7. ஆபத்தான operations-க்கு முன் எப்போதும் state backup எடு
8. terraform plan -refresh-only = மாற்றம் செய்யாமல் drift கண்டறி
9. State-ல் serial number = concurrent writes தடுக்கும் (optimistic locking)
```

---

## 📊 State Splitting Strategy

**English:**

Never put all resources in one monolith state! Split by blast radius and team ownership. If one state gets corrupted, only that component is affected.

**தமிழ்:**

எல்லா resources-ஐ ஒரே monolith state-ல் போடாதே! Blast radius மற்றும் team ownership-படி பிரி. ஒரு state corrupt ஆனால், அந்த component மட்டும் affected.

```
PRODUCTION STATE LAYOUT (TVS OTA Pattern):
├── networking/
│   └── terraform.tfstate     → VNets, Subnets, NSGs, UDRs
├── aks-cluster/
│   └── terraform.tfstate     → AKS cluster, node pools
├── aks-workloads/
│   └── terraform.tfstate     → Namespaces, RBAC, Helm releases
├── data/
│   └── terraform.tfstate     → CosmosDB, Storage, Redis
└── iam/
    └── terraform.tfstate     → Managed Identities, Role Assignments

EB CI/CT Pattern:
├── jenkins-infra/
│   └── terraform.tfstate     → Jenkins master VM, disks, networking
├── gerrit-infra/
│   └── terraform.tfstate     → Gerrit server, replication, networking
├── build-agents/
│   └── terraform.tfstate     → Agent VMs, autoscaling, images
└── shared-networking/
    └── terraform.tfstate     → VPC, subnets, firewall rules
```

### Cross-State Data Sharing

```hcl
# In aks-cluster/main.tf — read networking outputs:
data "terraform_remote_state" "networking" {
  backend = "azurerm"
  config = {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstateproduction"
    container_name       = "tfstate"
    key                  = "production/networking.tfstate"
  }
}

resource "azurerm_kubernetes_cluster" "aks" {
  default_node_pool {
    vnet_subnet_id = data.terraform_remote_state.networking.outputs.aks_subnet_id
  }
}

# ALTERNATIVE (less coupling): Use data sources
data "azurerm_subnet" "aks" {
  name                 = "snet-aks"
  virtual_network_name = "vnet-production"
  resource_group_name  = "rg-networking"
}
# This queries cloud directly — no dependency on another state
```

---

## 🔥 Scenario Challenge

### Scenario 1: The Monday Morning Disaster

**English:**

You come in Monday morning. A junior developer ran `terraform destroy` on the production networking state over the weekend (thought they were in dev workspace). All VNets, subnets, NSGs are gone. AKS cluster lost connectivity.

**What do you do?**

```
Answer Framework:
1. IMMEDIATE: Check if AKS pods are still running (networking gone ≠ pods dead immediately)
2. RESTORE NETWORKING:
   - Restore state from blob versioning (get last good state)
   - terraform state push <restored_state>
   - terraform apply (recreates networking with SAME config)
3. VERIFY: terraform plan shows "No changes" = fully recovered
4. POST-MORTEM:
   - Why did junior have destroy access to prod? → Fix IAM
   - Add prevent_destroy on networking resources
   - Add Azure Resource Locks
   - Separate CI/CD permissions: human can plan, only pipeline can apply/destroy
5. PREVENTION:
   - Use workspace-aware prompts or directory separation
   - CI/CD approval gates for destroy operations
   - terraform destroy should require -target or manual approval
```

**தமிழ்:**

திங்கள் காலை வருகிறாய். ஒரு junior developer weekend-ல் production networking state-ல் `terraform destroy` ஓட்டிவிட்டார் (dev workspace-ல் இருப்பதாக நினைத்தார்). எல்லா VNets, subnets, NSGs போய்விட்டன. AKS connectivity இழந்தது.

**என்ன செய்வாய்?** — மேலே உள்ள Answer Framework-ஐ பயன்படுத்து.

### Scenario 2: State Lock Stuck for 4 Hours

**English:**

CI/CD pipeline started terraform apply but the pipeline agent crashed mid-way. State lock has been held for 4 hours. Multiple developers are blocked.

```
Answer Framework:
1. VERIFY: Is something actually still running? Check CI/CD pipeline status
2. IDENTIFY: Get lock ID from error message
3. AZURE: Blob lease will auto-expire in 60 seconds... if it's been 4 hours,
   something else is wrong. Check if another pipeline picked up the same job.
4. AWS/GCS: Locks DON'T auto-expire! Must force-unlock.
5. FORCE UNLOCK:
   - terraform force-unlock <LOCK_ID>
   - OR: Azure CLI: az storage blob lease break ...
6. AFTER UNLOCK:
   - terraform plan -refresh-only (verify state consistency)
   - If state is half-applied: review plan carefully, may need manual fixes
7. PREVENTION:
   - Pipeline timeouts with proper cleanup
   - Alert on locks held > 30 minutes
```

### Scenario 3: Import 200 Existing Resources

**English:**

Your company has 200 manually-created Azure resources. Management wants them under Terraform. How do you approach this?

```
Answer Framework:
1. DISCOVERY: az resource list --output table (inventory everything)
2. PRIORITIZE: Start with stable resources (networking, RGs) not volatile ones
3. APPROACH:
   - Write resource blocks matching existing config
   - Use import blocks (Terraform 1.5+) for declarative import
   - terraform plan -generate-config-out=generated.tf (auto-generate HCL!)
4. BATCHING: Import 10-20 resources at a time (not all 200 at once)
5. VERIFICATION: terraform plan must show "No changes" after import
6. TOOL: aztfexport (Microsoft's tool) — auto-generates tf + imports!
7. GOTCHA: Some attributes are read-only, some have defaults — plan will show
   diffs even after import. Adjust code to match reality.
```

**தமிழ்:**

உன் company-ல் 200 manually-created Azure resources இருக்கின்றன. Management Terraform-ல் கொண்டு வர விரும்புகிறது.

Approach: Discovery → Prioritize → Import in batches → Verify "No changes" → Repeat

---

## 🏗️ Real Project Reference

### TVS OTA Platform — State Architecture

```
TVS OTA STATE STRATEGY:
═══════════════════════

Backend: Azure Blob Storage
- Storage Account: sttfstatetvsprod (Standard_GRS — geo-redundant!)
- Container: tfstate
- Versioning: ENABLED (can rollback any state)
- Locking: Azure Blob Lease (auto-healing)
- Encryption: Microsoft-managed key + soft delete

State Split (per cluster, per component):
├── shared/
│   ├── networking.tfstate          → Hub VNet, DNS zones, Firewall
│   └── iam.tfstate                 → Managed Identities, RBAC
├── cluster-india/
│   ├── aks.tfstate                 → AKS cluster + node pools
│   ├── data.tfstate                → CosmosDB, Blob Storage
│   └── observability.tfstate       → Log Analytics, App Insights
├── cluster-europe/
│   ├── aks.tfstate                 → AKS cluster + node pools
│   ├── data.tfstate                → CosmosDB (multi-region write)
│   └── observability.tfstate       → Log Analytics workspace
└── dr/
    └── passive-cluster.tfstate     → DR AKS (standby)

KEY DECISIONS:
- Separate state per cluster → blast radius limited to one region
- Shared networking in its own state → stable, rarely changes
- Data layer separate → different lifecycle than compute
- prevent_destroy on all production AKS clusters and databases
- State access: only CI/CD SPN (no human write access to prod state)
```

### EB CI/CT Platform — State Architecture

```
EB CI/CT STATE STRATEGY:
═══════════════════════

Backend: GCS Bucket
- Bucket: eb-terraform-state-prod (multi-region)
- Versioning: ENABLED
- Locking: GCS built-in
- Access: CI service account only

State Split (per component team):
├── jenkins-infra/
│   └── jenkins.tfstate             → Master VM, persistent disks, LB
├── gerrit-infra/
│   └── gerrit.tfstate              → Gerrit server, replication config
├── build-agents/
│   └── agents.tfstate              → Agent pool VMs, autoscaler, images
├── networking/
│   └── network.tfstate             → VPC, subnets, Cloud NAT, firewall
└── monitoring/
    └── monitoring.tfstate          → Prometheus VM, Grafana, alerting

WHY SEPARATE:
- Jenkins infra changes weekly (plugin updates, scaling)
- Gerrit infra changes monthly (stable, critical)
- Build agents change daily (scaling up/down, new images)
- Networking almost never changes (most stable)
- If agent state corrupts → only agents affected, Jenkins/Gerrit safe
```

---

## 📋 Best Practices

```
┌──────────────────────────────────────────────────────────────────┐
│           STATE MANAGEMENT BEST PRACTICES                        │
├──────────────────────────────────────────────────────────────────┤
│ BACKEND:                                                         │
│  ✓ Always remote backend (never local for teams)                │
│  ✓ Enable versioning (state rollback = life saver)              │
│  ✓ Geo-redundant storage (GRS/multi-region)                     │
│  ✓ Encrypt at rest with customer-managed keys                   │
│  ✓ Restrict access: CI/CD writes, humans read-only              │
│                                                                  │
│ SPLITTING:                                                       │
│  ✓ One state per component (networking, compute, data)          │
│  ✓ Max 50-100 resources per state (performance + blast radius)  │
│  ✓ Split by lifecycle (stable infra vs frequently-changing)     │
│  ✓ Split by team ownership                                      │
│                                                                  │
│ OPERATIONS:                                                      │
│  ✓ Use moved blocks (not state mv) for refactoring              │
│  ✓ Use import blocks (not CLI import) for adoption              │
│  ✓ Always backup state before risky operations                  │
│  ✓ terraform plan -refresh-only before big changes              │
│  ✓ Never edit terraform.tfstate JSON manually                   │
│                                                                  │
│ SECURITY:                                                        │
│  ✓ Never commit state to Git (.gitignore *.tfstate*)            │
│  ✓ Audit logging on state storage account                       │
│  ✓ Rotate state storage access keys regularly                   │
│  ✓ Use ephemeral resources (1.7+) for secrets                   │
│                                                                  │
│ DR/SAFETY:                                                       │
│  ✓ prevent_destroy on critical resources (AKS, databases)       │
│  ✓ Cloud resource locks as second layer                         │
│  ✓ Test state recovery quarterly                                │
│  ✓ Document import procedures for each resource type            │
│  ✓ CI/CD: require approval for destroy operations               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A

### Q1: What happens if two people run terraform apply simultaneously?

**English:**
Remote backend has locking. Second person gets "Error acquiring the state lock" with lock details (who, when, operation). Must wait until first completes. If lock stuck (crash) → verify nobody running → `terraform force-unlock`. Azure Blob Lease auto-expires in 60s. AWS DynamoDB lock does NOT auto-expire.

**தமிழ்:**
Remote backend-ல் locking இருக்கும். இரண்டாவது நபருக்கு "state lock" error வரும். முதல் apply முடியும் வரை காத்திரு. Lock stuck ஆனால் → யாரும் run செய்யவில்லை என்று verify → `terraform force-unlock`. Azure Blob Lease 60 வினாடியில் auto-expire. AWS DynamoDB lock auto-expire ஆகாது.

### Q2: How do you handle state for 500+ resources across 5 teams?

**English:**
Split into 15-20 independent states by component and team. Each team owns their states. CI/CD applies (no human direct access). Use terraform_remote_state or data sources for cross-references. Enforce through CI: PR → plan → peer review plan output → approve → apply.

**தமிழ்:**
Component மற்றும் team-படி 15-20 independent states-ஆக பிரி. ஒவ்வொரு team-ம் தங்கள் states own செய்யும். CI/CD apply செய்யும் (human direct access இல்லை). Cross-references-க்கு terraform_remote_state அல்லது data sources. CI enforce: PR → plan → peer review → approve → apply.

### Q3: State file contains database passwords. How do you secure it?

**English:**
1. Backend encryption (SSE + CMK), 2. IAM restriction (only CI SPN writes), 3. No Git commits (*.tfstate in .gitignore), 4. Audit logging, 5. For new code: use ephemeral resources (Terraform 1.7+) or external secret injection. 6. sensitive = true hides from output but NOT from state file.

**தமிழ்:**
1. Backend encryption (SSE + CMK), 2. IAM restriction (CI SPN மட்டும் write), 3. Git-க்கு commit வேண்டாம், 4. Audit logging, 5. புதிய code-க்கு: ephemeral resources அல்லது external secret injection, 6. sensitive = true output-ல் மறைக்கும் ஆனால் state file-ல் இல்லை.

### Q4: Someone manually deleted a resource. What happens on next plan?

**English:**
terraform plan detects drift: "resource in state but not in cloud." It proposes recreating it. Options: 1. Apply to recreate (match desired state), 2. `terraform state rm` if you don't want it anymore, 3. `terraform plan -refresh-only` + apply to update state to match reality.

**தமிழ்:**
terraform plan drift detect செய்யும்: "state-ல் resource இருக்கு ஆனால் cloud-ல் இல்லை." Recreate செய்ய propose செய்யும். Options: 1. Apply → recreate, 2. `terraform state rm` → இனி வேண்டாம் என்றால், 3. `plan -refresh-only` → state-ஐ reality-க்கு match செய்.

### Q5: You need to refactor — move resources into modules. How without destroying?

**English:**
Two approaches: 1. `moved` blocks (preferred, Terraform 1.5+) — declare in code, version-controlled, team-wide. 2. `terraform state mv` — imperative, one-time, must run manually. Always verify with `terraform plan` showing "No changes" after move.

**தமிழ்:**
இரண்டு approaches: 1. `moved` blocks (preferred, 1.5+) — code-ல் declare, version-controlled, team-wide. 2. `terraform state mv` — imperative, one-time, manually run. Move-க்கு பிறகு `terraform plan` "No changes" காட்ட வேண்டும் என்று verify செய்.

### Q6: How do you recover from a corrupted state file?

**English:**
1. Restore from backend versioning (Azure Blob versions / GCS object versions). 2. Download previous version → `terraform state push`. 3. Run `terraform plan -refresh-only` to verify. 4. If no versioning (disaster!): import every resource manually. 5. Prevention: ALWAYS enable versioning, geo-redundant storage, regular backups.

**தமிழ்:**
1. Backend versioning-லிருந்து restore (Azure Blob / GCS versions). 2. முந்தைய version download → `terraform state push`. 3. `terraform plan -refresh-only` ஓட்டி verify. 4. Versioning இல்லை என்றால்: ஒவ்வொரு resource-ஐ manually import. 5. Prevention: எப்போதும் versioning enable, geo-redundant storage, regular backups.

### Q7: Workspaces vs separate directories for multi-environment?

**English:**
Workspaces: same architecture, different scale (dev=2 nodes, prod=10). Directories: different architectures per environment (prod has WAF, DR; dev doesn't). My preference for enterprise: directories + Terragrunt for DRY code. Workspaces have a hidden gotcha — easy to apply to wrong workspace.

**தமிழ்:**
Workspaces: ஒரே architecture, வெவ்வேறு scale. Directories: ஒவ்வொரு env-க்கும் வெவ்வேறு architecture. Enterprise preference: directories + Terragrunt. Workspaces gotcha: தவறான workspace-ல் apply செய்ய எளிது.

### Q8: How does the backend bootstrap problem work?

**English:**
Chicken-and-egg: you need storage to store state, but you want to create storage with Terraform. Solution: 1. Create state storage manually (CLI/portal) or with a bootstrap Terraform that uses local state. 2. Then configure backend in main Terraform. 3. Some teams have a "layer-0" Terraform with local state that only creates state storage + IAM.

**தமிழ்:**
கோழி-முட்டை problem: state store செய்ய storage வேண்டும், ஆனால் storage-ஐ Terraform-ல் create செய்ய வேண்டும். Solution: 1. State storage-ஐ manually create (CLI/portal) அல்லது local state-உடன் bootstrap Terraform. 2. பிறகு main Terraform-ல் backend configure. 3. சில teams "layer-0" Terraform வைத்திருப்பார்கள் — local state, state storage + IAM மட்டும் create.

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Three-way comparison explain முடியும் (code vs state vs cloud)
- [ ] Remote backend configure முடியும் (Azure, GCP, AWS differences)
- [ ] Locking mechanism per backend explain முடியும் (Blob lease vs DynamoDB vs GCS)
- [ ] state mv, rm, import — when to use each explain முடியும்
- [ ] moved block vs state mv — why moved is better
- [ ] State file security risks explain முடியும் (what secrets leak)
- [ ] State corruption recovery steps explain முடியும்
- [ ] State splitting strategy design முடியும் (by component/team/lifecycle)
- [ ] Workspace vs directory — trade-offs explain முடியும்
- [ ] terraform_remote_state vs data source — when each is appropriate
- [ ] Import 200 resources — approach and tooling explain முடியும்
- [ ] Backend bootstrap problem — solution explain முடியும்
- [ ] Real project state layout design முடியும் (TVS OTA / EB CI/CT pattern)
