# Module 12: GCP Storage & Data
# மாடுல் 12: GCP Storage & Data Services

---

## 🎯 What? | என்ன?

**English:** Provision GCP data services — Cloud Storage (GCS), Cloud SQL (PostgreSQL/MySQL), Secret Manager, and Memorystore (Redis).

**தமிழ்:** GCP data services provision — Cloud Storage, Cloud SQL, Secret Manager, Redis.

---

## 🛠️ Cloud Storage (GCS)

```hcl
resource "google_storage_bucket" "artifacts" {
  name          = "${var.gcp_project}-artifacts-${var.environment}"
  location      = var.region
  force_destroy = var.environment != "prod"

  uniform_bucket_level_access = true    # Recommended (no per-object ACLs)

  versioning { enabled = true }

  lifecycle_rule {
    condition { age = 90 }
    action { type = "SetStorageClass", storage_class = "NEARLINE" }
  }
  lifecycle_rule {
    condition { age = 365 }
    action { type = "Delete" }
  }

  encryption {
    default_kms_key_name = google_kms_crypto_key.storage.id
  }
}
```

---

## 🛠️ Cloud SQL (PostgreSQL)

```hcl
resource "google_sql_database_instance" "main" {
  name             = "sql-${var.project}-${var.environment}"
  database_version = "POSTGRES_14"
  region           = var.region

  settings {
    tier              = var.environment == "prod" ? "db-custom-4-16384" : "db-f1-micro"
    availability_type = var.environment == "prod" ? "REGIONAL" : "ZONAL"
    disk_size         = 50
    disk_type         = "PD_SSD"

    ip_configuration {
      ipv4_enabled    = false              # No public IP!
      private_network = google_compute_network.main.id
    }

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "02:00"
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }

    maintenance_window {
      day  = 7    # Sunday
      hour = 3
    }
  }

  deletion_protection = var.environment == "prod"
}

resource "google_sql_database" "app" {
  name     = "app_db"
  instance = google_sql_database_instance.main.name
}

resource "google_sql_user" "app" {
  name     = "app_user"
  instance = google_sql_database_instance.main.name
  password = random_password.db.result
}
```

---

## 🛠️ Secret Manager

```hcl
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password-${var.environment}"
  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "db_password" {
  secret      = google_secret_manager_secret.db_password.id
  secret_data = random_password.db.result
}

# Grant GKE workload access
resource "google_secret_manager_secret_iam_member" "app_access" {
  secret_id = google_secret_manager_secret.db_password.id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.app.email}"
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│       GCP DATA SERVICES CHEAT SHEET              │
├──────────────────────────────────────────────────┤
│ CLOUD STORAGE:                                   │
│   uniform_bucket_level_access = true             │
│   Versioning for critical data                   │
│   Lifecycle rules (auto-archive/delete)          │
│   Customer-managed encryption keys (CMEK)        │
│                                                  │
│ CLOUD SQL:                                       │
│   Private IP only (no public exposure)           │
│   REGIONAL availability for prod                 │
│   Point-in-time recovery enabled                 │
│   deletion_protection = true (prod)              │
│                                                  │
│ SECRET MANAGER:                                  │
│   Replaces .tfvars secrets                       │
│   IAM-controlled access                          │
│   Version history & rotation                     │
│   GKE access via Workload Identity               │
│                                                  │
│ AZURE vs GCP MAPPING:                            │
│   Blob Storage → Cloud Storage (GCS)             │
│   PostgreSQL Flex → Cloud SQL                    │
│   Key Vault → Secret Manager + Cloud KMS         │
│   Private Endpoint → Private Service Connect     │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] GCS bucket with lifecycle rules provision முடியும்
- [ ] Cloud SQL (private IP, HA) provision முடியும்
- [ ] Secret Manager with IAM configure முடியும்
- [ ] Azure vs GCP data services map முடியும்
