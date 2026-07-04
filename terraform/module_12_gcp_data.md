# Module 12: GCP Storage & Data with Terraform

## 📖 Story: The Office Filing System

**English:**
Think of GCP's data services as different parts of an office filing system:
- **Cloud Storage (GCS)** = The document warehouse — infinite shelves, organize by folders (prefixes), different sections for hot documents (Standard), cold archives (Coldline/Archive)
- **Cloud SQL** = The structured filing cabinet — organized drawers (tables), indexed for fast lookup, replicated copies in another building (HA)
- **Secret Manager** = The vault — combination-locked, audit trail for every access, versioned documents
- **Memorystore** = The whiteboard — fast, temporary, everyone can see it quickly, but it's erased when you're done

In EB's pipeline, GCS stored build artifacts, Cloud SQL held CI metadata, Secret Manager managed deploy credentials, and Memorystore cached frequently-accessed configs. This is the standard "data tier" pattern for any GCP infrastructure.

**தமிழ்:**
GCP data services-ஐ office filing system-ன் வெவ்வேறு பகுதிகளாக நினைக்கவும்:
- **Cloud Storage (GCS)** = Document warehouse — infinite shelves, folders (prefixes) மூலம் organize, hot documents-க்கு Standard, cold archives-க்கு Coldline/Archive
- **Cloud SQL** = Structured filing cabinet — organized drawers (tables), fast lookup-க்கு indexed, HA-க்கு replicated copies
- **Secret Manager** = Vault — combination-locked, ஒவ்வொரு access-க்கும் audit trail, versioned
- **Memorystore** = Whiteboard — fast, temporary, quickly access, done ஆனால் erase

EB pipeline-ல், GCS build artifacts store, Cloud SQL CI metadata hold, Secret Manager deploy credentials manage, Memorystore frequently-accessed configs cache. இது standard "data tier" pattern.

---

## 📊 Architecture & Concepts

### GCP Data Services Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    GCP DATA TIER                                  │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Cloud Storage │  │  Cloud SQL   │  │   Secret Manager     │  │
│  │    (GCS)      │  │ (PostgreSQL) │  │                      │  │
│  │              │  │              │  │  ┌────┐ ┌────┐       │  │
│  │ ┌──────────┐│  │ Primary      │  │  │v1  │ │v2  │       │  │
│  │ │artifacts/││  │   ↓ (HA)     │  │  └────┘ └────┘       │  │
│  │ │builds/   ││  │ Standby      │  │  deploy-key           │  │
│  │ │backups/  ││  │   ↓ (read)   │  │  db-password          │  │
│  │ └──────────┘│  │ Read Replica │  │  api-token            │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                   │
│  ┌──────────────┐                                                │
│  │ Memorystore  │                                                │
│  │   (Redis)    │                                                │
│  │  Cache layer │                                                │
│  └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

### Azure vs GCP Data Services Comparison

| Feature | Azure | GCP | Key Difference |
|---------|-------|-----|----------------|
| **Object Storage** | Blob Storage | Cloud Storage (GCS) | GCS uses flat namespace with prefixes (no true folders) |
| **Storage tiers** | Hot/Cool/Cold/Archive | Standard/Nearline/Coldline/Archive | Similar pricing, GCP min retention penalties differ |
| **Managed DB** | Azure PostgreSQL Flexible Server | Cloud SQL | Cloud SQL supports MySQL/PostgreSQL/SQL Server |
| **DB HA** | Zone-redundant HA | Regional HA (auto-failover) | Both use synchronous replication |
| **Read replicas** | Read replicas (same region/cross) | Read replicas (same/cross region) | Similar capability |
| **Secrets** | Key Vault | Secret Manager | Key Vault = keys+certs+secrets; GCP splits them |
| **Key management** | Key Vault (HSM keys) | Cloud KMS | GCP separates KMS from secrets |
| **Certificates** | Key Vault Certificates | Certificate Manager | Different services in GCP |
| **Cache** | Azure Cache for Redis | Memorystore (Redis/Memcached) | Similar, GCP also offers Memcached |
| **IAM access** | RBAC + Access Policies | IAM roles + conditions | GCP uses project-level IAM |
| **Encryption** | Service-managed/CMK | Google-managed/CMEK | Same concept, different names |
| **Private access** | Private Endpoint | Private Service Connect / VPC | GCS: Private Google Access (no PE needed) |

### Cloud Storage (GCS) — Key Architecture Points

**English:**
- **Global namespace**: Bucket names are globally unique (like Azure storage account names)
- **Flat structure**: No real folders — "folder/file.txt" is just an object name with "/" in it
- **Storage classes**: Applied per-object, not per-container (Azure: per container tier)
- **Lifecycle rules**: Automatically transition objects between tiers or delete
- **Uniform access**: Recommended — IAM only, no per-object ACLs (simpler, auditable)
- **Versioning**: Keep all versions of an object (for backup/compliance)

**தமிழ்:**
- **Global namespace**: Bucket names globally unique (Azure storage account names போல)
- **Flat structure**: Real folders இல்லை — "folder/file.txt" என்பது "/" உள்ள object name
- **Storage classes**: Per-object apply (Azure: per container tier)
- **Lifecycle rules**: Objects-ஐ automatically tiers இடையே transition அல்லது delete
- **Uniform access**: Recommended — IAM only, per-object ACLs இல்லை (simpler, auditable)

### Secret Manager vs Azure Key Vault

**English:**
GCP splits what Azure combines in Key Vault:
- **Secret Manager**: Stores secret values (passwords, API keys, config)
- **Cloud KMS**: Manages encryption keys (CMEK)
- **Certificate Manager**: Handles TLS certificates

Interview insight: "GCP follows separation of concerns — secrets, keys, and certs are separate services. Azure bundles them in Key Vault. GCP's approach gives finer-grained IAM control."

**தமிழ்:**
Azure Key Vault-ல் combine செய்வதை GCP split செய்கிறது:
- **Secret Manager**: Secret values store (passwords, API keys)
- **Cloud KMS**: Encryption keys manage
- **Certificate Manager**: TLS certificates handle

Interview insight: "GCP separation of concerns follow செய்கிறது — secrets, keys, certs separate services. Azure Key Vault-ல் bundle. GCP approach finer-grained IAM control கொடுக்கும்."

---

## 🧠 Byheart for Interview

### GCP Data Services Key Facts
```
1. GCS bucket names are GLOBALLY unique (across all GCP projects)
2. GCS has NO folder concept — "/" in object name creates virtual folder
3. Uniform bucket-level access: IAM ONLY (recommended over legacy ACLs)
4. Cloud SQL HA: automatic failover, 99.95% SLA, synchronous replication
5. Cloud SQL: private IP (VPC-native) recommended over public IP
6. Secret Manager: automatic versioning, IAM per-secret, audit logging
7. Secret Manager ≠ Cloud KMS (secrets vs encryption keys — split in GCP)
8. Memorystore Redis: VPC-native only (no public access), managed HA
9. GCS lifecycle: Standard → Nearline (30d) → Coldline (90d) → Archive (365d)
10. CMEK: use Cloud KMS key to encrypt GCS/Cloud SQL/Memorystore
```

### Cost Optimization Patterns
```
GCS Storage Classes (per GB/month):
  Standard:  $0.020  (frequent access)
  Nearline:  $0.010  (once/month, 30-day min)
  Coldline:  $0.004  (once/quarter, 90-day min)
  Archive:   $0.0012 (once/year, 365-day min)

Pattern: Lifecycle rule to auto-transition build artifacts:
  0-30 days:  Standard (recent builds, frequent access)
  30-90 days: Nearline (occasionally referenced)
  90+ days:   Coldline (compliance retention)
  365+ days:  Delete (unless audit requirement)
```

### Interview Golden Answers
```
Q: "How do you manage secrets across GCP services?"
A: "Secret Manager with Workload Identity integration:
    1. Secrets stored in Secret Manager (versioned, audited)
    2. Workload Identity on GKE pods → access secrets without keys
    3. IAM per-secret: each service only accesses its own secrets
    4. Secret rotation: new version in Secret Manager → apps read latest
    5. No keys in env vars, no keys in repos — zero static credentials
    
    vs Azure: Key Vault + Pod Identity — same concept, different service boundary."

Q: "GCS vs Azure Blob — architectural differences?"
A: "Three key differences:
    1. Namespace: GCS = globally unique bucket; Azure = account + container
    2. Tiers: GCS per-object; Azure Blob per-account (set default tier)
    3. Access: GCS uniform bucket access (IAM); Azure = RBAC + SAS tokens
    
    Similarity: both support lifecycle policies, versioning, replication."

Q: "Cloud SQL HA — how does failover work?"
A: "Regional HA with synchronous replication:
    - Primary in zone A, standby in zone B (same region)
    - Synchronous disk replication (RPO = 0, no data loss)
    - Automatic failover on primary failure (~60-120 seconds)
    - Same connection IP (no app changes needed)
    - Read replicas for read scaling (async, separate from HA)"
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/gcp-data && cd ~/tf-lab/gcp-data
```

### Exercise 1: Cloud Storage with Lifecycle

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

# ─── GCS BUCKET (Build Artifacts) ─────────────────────────────────
resource "google_storage_bucket" "artifacts" {
  name     = "${var.project_id}-build-artifacts"  # Must be globally unique
  location = var.region

  # Uniform bucket access — IAM only, no ACLs
  uniform_bucket_level_access = true

  # Versioning for rollback capability
  versioning {
    enabled = true
  }

  # Lifecycle rules — auto-transition and cleanup
  lifecycle_rule {
    condition {
      age = 30  # After 30 days
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }

  lifecycle_rule {
    condition {
      age = 90  # After 90 days
    }
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
  }

  lifecycle_rule {
    condition {
      age = 365  # After 1 year — delete
    }
    action {
      type = "Delete"
    }
  }

  # Delete old versions after 30 days
  lifecycle_rule {
    condition {
      num_newer_versions = 3
      with_state         = "ARCHIVED"
    }
    action {
      type = "Delete"
    }
  }

  # Encryption with CMEK (Customer-Managed Encryption Key)
  encryption {
    default_kms_key_name = var.kms_key_id
  }

  # Prevent accidental deletion
  force_destroy = false

  labels = {
    environment = var.environment
    team        = "devops"
  }
}

# ─── IAM: CI agents can read/write artifacts ──────────────────────
resource "google_storage_bucket_iam_member" "ci_writer" {
  bucket = google_storage_bucket.artifacts.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${var.ci_service_account}"
}

# ─── IAM: Developers can read only ────────────────────────────────
resource "google_storage_bucket_iam_member" "dev_reader" {
  bucket = google_storage_bucket.artifacts.name
  role   = "roles/storage.objectViewer"
  member = "group:developers@company.com"
}
EOF
```

### Exercise 2: Cloud SQL (PostgreSQL) with HA

```hcl
cat > cloudsql.tf << 'EOF'
# ─── CLOUD SQL INSTANCE (PostgreSQL, HA) ──────────────────────────
resource "google_sql_database_instance" "main" {
  name             = "sql-${var.environment}"
  database_version = "POSTGRES_15"
  region           = var.region

  # Prevent accidental deletion
  deletion_protection = true

  settings {
    tier              = "db-custom-4-16384"  # 4 vCPU, 16GB RAM
    availability_type = "REGIONAL"           # HA (zone failover)
    disk_type         = "PD_SSD"
    disk_size         = 100
    disk_autoresize   = true

    # Private IP only (VPC-native, no public access)
    ip_configuration {
      ipv4_enabled                                  = false  # No public IP!
      private_network                               = var.vpc_id
      enable_private_path_for_google_cloud_services = true
    }

    # Backup configuration
    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"  # 2 AM
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7

      backup_retention_settings {
        retained_backups = 30
        retention_unit   = "COUNT"
      }
    }

    # Maintenance window
    maintenance_window {
      day          = 7  # Sunday
      hour         = 3  # 3 AM
      update_track = "stable"
    }

    # Insights (query performance)
    insights_config {
      query_insights_enabled  = true
      record_application_tags = true
      record_client_address   = true
    }

    database_flags {
      name  = "max_connections"
      value = "200"
    }

    database_flags {
      name  = "log_min_duration_statement"
      value = "1000"  # Log queries > 1 second
    }
  }
}

# ─── DATABASE ─────────────────────────────────────────────────────
resource "google_sql_database" "ci_metadata" {
  name     = "ci_metadata"
  instance = google_sql_database_instance.main.name
}

# ─── USER (password from Secret Manager) ──────────────────────────
resource "google_sql_user" "app" {
  name     = "app_user"
  instance = google_sql_database_instance.main.name
  password = google_secret_manager_secret_version.db_password.secret_data
}

# ─── READ REPLICA (for reporting/analytics) ───────────────────────
resource "google_sql_database_instance" "read_replica" {
  name                 = "sql-${var.environment}-replica"
  master_instance_name = google_sql_database_instance.main.name
  database_version     = "POSTGRES_15"
  region               = var.region

  replica_configuration {
    failover_target = false
  }

  settings {
    tier            = "db-custom-2-8192"  # Smaller than primary
    disk_type       = "PD_SSD"
    disk_autoresize = true

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_id
    }
  }
}
EOF
```

### Exercise 3: Secret Manager

```hcl
cat > secrets.tf << 'EOF'
# ─── SECRET: Database Password ────────────────────────────────────
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password-${var.environment}"

  replication {
    auto {}  # Google manages replication
  }

  labels = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

# ─── SECRET VERSION (the actual value) ────────────────────────────
resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = var.db_password  # From tfvars or CI variable
}

# ─── SECRET: API Token ────────────────────────────────────────────
resource "google_secret_manager_secret" "api_token" {
  secret_id = "api-token-${var.environment}"

  replication {
    auto {}
  }

  # Rotation reminder (rotation handled externally)
  rotation {
    rotation_period = "7776000s"  # 90 days
  }
}

# ─── IAM: GKE pods can access secrets ─────────────────────────────
resource "google_secret_manager_secret_iam_member" "ci_access" {
  secret_id = google_secret_manager_secret.db_password.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${var.ci_service_account}"
}

# ─── IAM: Only specific service can access API token ──────────────
resource "google_secret_manager_secret_iam_member" "api_access" {
  secret_id = google_secret_manager_secret.api_token.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${var.app_service_account}"
}
EOF
```

### Exercise 4: Memorystore (Redis)

```hcl
cat > memorystore.tf << 'EOF'
# ─── MEMORYSTORE REDIS (cache layer) ──────────────────────────────
resource "google_redis_instance" "cache" {
  name           = "redis-${var.environment}"
  tier           = "STANDARD_HA"  # HA with automatic failover
  memory_size_gb = 4
  region         = var.region

  # Redis version
  redis_version = "REDIS_7_0"

  # VPC-native (private access only)
  authorized_network = var.vpc_id
  connect_mode       = "PRIVATE_SERVICE_ACCESS"

  # Persistence (RDB snapshots)
  persistence_config {
    persistence_mode    = "RDB"
    rdb_snapshot_period = "TWELVE_HOURS"
  }

  # Maintenance window
  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 2
        minutes = 0
      }
    }
  }

  # Auth (require password)
  auth_enabled = true

  # Transit encryption
  transit_encryption_mode = "SERVER_AUTHENTICATION"

  labels = {
    environment = var.environment
  }
}

# Output connection details
output "redis_host" {
  value = google_redis_instance.cache.host
}

output "redis_port" {
  value = google_redis_instance.cache.port
}
EOF
```

### Exercise 5: Variables

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
  description = "VPC network self_link for private access"
}

variable "ci_service_account" {
  type        = string
  description = "CI workload service account email"
}

variable "app_service_account" {
  type        = string
  description = "Application service account email"
}

variable "kms_key_id" {
  type        = string
  description = "Cloud KMS key for CMEK encryption"
  default     = ""
}

variable "db_password" {
  type        = string
  sensitive   = true
  description = "Database password (from CI secrets)"
}
EOF
```

```bash
terraform init
terraform validate
terraform plan -var="project_id=my-project" -var="vpc_id=projects/my-project/global/networks/vpc-dev" -var="ci_service_account=ci@my-project.iam.gserviceaccount.com" -var="app_service_account=app@my-project.iam.gserviceaccount.com" -var="db_password=dummy"
```

---

## 🔥 Scenario Challenge

### Scenario: Secure Data Tier for CI/CD Platform

**English:**
Design a complete data tier for a CI/CD platform:
1. Artifacts stored in GCS with automatic lifecycle (save costs)
2. CI metadata in Cloud SQL (HA, private IP, encrypted)
3. All secrets in Secret Manager (no hardcoded credentials anywhere)
4. Caching layer for frequently-accessed configs
5. All access via Workload Identity (zero static keys)

**தமிழ்:**
CI/CD platform-க்கு complete data tier design:
1. GCS-ல் artifacts, automatic lifecycle (costs save)
2. Cloud SQL-ல் CI metadata (HA, private IP, encrypted)
3. எல்லா secrets-ம் Secret Manager-ல் (hardcoded credentials இல்லை)
4. Frequently-accessed configs-க்கு caching layer
5. எல்லா access-ம் Workload Identity வழியாக (zero static keys)

### Solution: Zero-Trust Data Tier

```hcl
# Pattern: Every service accesses data via its own identity
# No shared credentials, no static keys

# 1. Service accounts per workload
resource "google_service_account" "ci_builder" {
  account_id = "ci-builder"
}
resource "google_service_account" "ci_deployer" {
  account_id = "ci-deployer"
}

# 2. GCS — builder writes, deployer reads
resource "google_storage_bucket_iam_member" "builder_write" {
  bucket = google_storage_bucket.artifacts.name
  role   = "roles/storage.objectCreator"
  member = "serviceAccount:${google_service_account.ci_builder.email}"
}
resource "google_storage_bucket_iam_member" "deployer_read" {
  bucket = google_storage_bucket.artifacts.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.ci_deployer.email}"
}

# 3. Secrets — deployer can access deploy keys, builder cannot
resource "google_secret_manager_secret_iam_member" "deployer_secrets" {
  secret_id = google_secret_manager_secret.deploy_key.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.ci_deployer.email}"
}

# 4. Workload Identity bindings (GKE pods → GCP SAs)
resource "google_service_account_iam_binding" "builder_wi" {
  service_account_id = google_service_account.ci_builder.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[ci/builder-sa]"
  ]
}
resource "google_service_account_iam_binding" "deployer_wi" {
  service_account_id = google_service_account.ci_deployer.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[ci/deployer-sa]"
  ]
}
```

---

## 🏗️ Real Project Reference (EB)

**English:**
EB's data tier on GCP:
- **GCS**: Build artifacts bucket with lifecycle (Standard 30d → Nearline 90d → delete 365d), CMEK encryption
- **Cloud SQL PostgreSQL**: CI pipeline metadata, regional HA, private IP only, automated backups with PITR
- **Secret Manager**: All deploy credentials, registry tokens, SSH keys — accessed via Workload Identity
- **Memorystore Redis**: Cache for pipeline configs, test result aggregation
- **Zero static keys**: Every pod accesses data through Workload Identity → Service Account → IAM binding

Security audit result: Zero secrets in code, zero long-lived credentials, full audit trail via Cloud Audit Logs.

**தமிழ்:**
EB data tier GCP-ல்:
- **GCS**: Build artifacts, lifecycle (Standard→Nearline→delete), CMEK encryption
- **Cloud SQL**: CI metadata, regional HA, private IP, automated backups + PITR
- **Secret Manager**: Deploy credentials, registry tokens — Workload Identity வழியாக access
- **Memorystore Redis**: Pipeline configs cache, test results aggregation
- **Zero static keys**: ஒவ்வொரு pod-ம் Workload Identity→Service Account→IAM binding வழியாக data access

Security audit result: Code-ல் zero secrets, zero long-lived credentials, Cloud Audit Logs-ல் full audit trail.

---

## 🎤 Interview Q&A

### Q1: "How do you manage secrets across GCP services?"

**English:**
"I use a layered approach with Secret Manager at the center:

1. **Storage**: All secrets in Secret Manager (versioned, replicated)
2. **Access**: IAM per-secret — each service account accesses only its secrets
3. **Authentication**: Workload Identity for GKE pods, attached SA for GCE VMs
4. **Rotation**: Secret Manager rotation with Cloud Functions trigger
5. **Audit**: Cloud Audit Logs for every secretmanager.versions.access call
6. **Zero static keys**: No JSON key files anywhere — Workload Identity eliminates them

Comparison with Azure: Secret Manager ≈ Key Vault secrets. But GCP separates keys (Cloud KMS) from secrets (Secret Manager), giving finer IAM control. In Azure, Key Vault handles all three (secrets, keys, certs)."

**தமிழ்:**
"Secret Manager-ஐ center-ல் வைத்து layered approach:
1. **Storage**: Secret Manager-ல் எல்லா secrets (versioned, replicated)
2. **Access**: Per-secret IAM — ஒவ்வொரு service account-க்கும் அதன் secrets மட்டும்
3. **Authentication**: GKE pods-க்கு Workload Identity, GCE VMs-க்கு attached SA
4. **Rotation**: Cloud Functions trigger உடன் Secret Manager rotation
5. **Audit**: ஒவ்வொரு access-க்கும் Cloud Audit Logs
6. **Zero static keys**: JSON key files எங்கும் இல்லை"

### Q2: "GCS vs Azure Blob Storage — key architecture differences"

**English:**
"Three architectural differences:

1. **Naming model**: Azure = Storage Account → Container → Blob (3-level). GCS = Bucket → Object (2-level, flat namespace). Bucket name must be globally unique.

2. **Access control**: Azure has RBAC + SAS tokens + Access Keys. GCS recommends uniform bucket-level access (IAM only). SAS equivalent in GCS is signed URLs (time-limited).

3. **Tiering**: Azure sets tier at blob level but default at account level. GCS sets class per-object. Both support lifecycle rules. GCS lifecycle is cleaner — single rule set per bucket.

For enterprise: both are mature. GCS's uniform access + IAM is simpler to audit. Azure's SAS tokens give more flexible sharing but harder to revoke."

**தமிழ்:**
"மூன்று architectural differences:
1. **Naming**: Azure = Account→Container→Blob (3-level). GCS = Bucket→Object (2-level, flat). Bucket name globally unique.
2. **Access**: Azure = RBAC + SAS + Access Keys. GCS = uniform bucket access (IAM only). Signed URLs = SAS equivalent.
3. **Tiering**: Azure per-blob with account default. GCS per-object. GCS lifecycle simpler — single rule set per bucket."

### Q3: "Cloud SQL HA vs Azure PostgreSQL Flexible Server HA"

**English:**
"Both provide zone-level HA with automatic failover:

**Cloud SQL HA:**
- Regional HA instance (primary + standby in different zones)
- Synchronous replication at disk level
- Automatic failover ~60-120 seconds
- Same IP address after failover (no connection string change)
- Separate read replicas (async) for read scaling

**Azure PostgreSQL Flexible HA:**
- Zone-redundant HA (similar concept)
- Synchronous replication (streaming)
- Automatic failover ~60-120 seconds
- Same FQDN after failover
- Read replicas available separately

Key difference: Cloud SQL uses persistent disk replication (block-level). Azure uses PostgreSQL streaming replication (WAL-based). Result is similar — RPO=0, RTO ~2 minutes."

**தமிழ்:**
"இரண்டும் zone-level HA, automatic failover:
- Cloud SQL: Disk-level synchronous replication, ~60-120s failover, same IP
- Azure Flexible: WAL-based streaming replication, ~60-120s failover, same FQDN
- Key difference: Cloud SQL block-level replication, Azure WAL-based
- Result similar: RPO=0, RTO ~2 minutes"

### Q4: "How do you encrypt data at rest and in transit in GCP?"

**English:**
"GCP has three encryption layers:

**At rest:**
1. **Default**: Google-managed encryption (automatic, no config needed)
2. **CMEK**: Customer-Managed Encryption Key via Cloud KMS (you control the key)
3. **CSEK**: Customer-Supplied Encryption Key (you provide the key — rare, compliance-driven)

**In transit:**
- GCS: HTTPS enforced (can require via bucket policy)
- Cloud SQL: SSL/TLS (can be enforced via `require_ssl = true`)
- Memorystore: `transit_encryption_mode = SERVER_AUTHENTICATION`
- Inter-service: Google's ALTS (Application Layer Transport Security) automatic

Azure equivalent: Service-managed = Google-managed, CMK = CMEK, BYO key = CSEK"

**தமிழ்:**
"GCP-ல் மூன்று encryption layers:
**At rest**: 1. Google-managed (automatic) 2. CMEK (Cloud KMS key) 3. CSEK (you provide key)
**In transit**: GCS=HTTPS, Cloud SQL=SSL/TLS, Memorystore=TLS, inter-service=ALTS automatic

Azure equivalent: Service-managed = Google-managed, CMK = CMEK, BYO key = CSEK"

---

## ✅ Self-Check

| # | Question | Can You Answer? |
|---|----------|-----------------|
| 1 | GCS vs Azure Blob — naming model difference? | ☐ |
| 2 | GCS uniform bucket access — what and why? | ☐ |
| 3 | GCS lifecycle rules — cost optimization pattern? | ☐ |
| 4 | Cloud SQL HA — how failover works? | ☐ |
| 5 | Cloud SQL private IP — why not public? | ☐ |
| 6 | Secret Manager vs Key Vault — split vs combined? | ☐ |
| 7 | Secret access pattern with Workload Identity? | ☐ |
| 8 | CMEK encryption — when and how? | ☐ |
| 9 | Memorystore — use cases and access pattern? | ☐ |
| 10 | Zero-trust data tier — design principles? | ☐ |

---

*Next: [Module 13 — Security & Compliance](module_13_security_compliance.md)*
