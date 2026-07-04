# Module 08: Azure Storage & Data with Terraform

## 📖 Story: The Secure Vault System

**English:**
Think of Azure data services as a bank's vault system. Blob Storage is the document vault — stores millions of files (firmware packages) with different access tiers (hot vault for daily access, cold vault for archives). PostgreSQL is the ledger book — structured records of every transaction (OTA campaign metadata). Key Vault is the master safe — holds the keys that unlock everything else (PKI certificates, signing keys). Private Endpoints are the underground tunnels connecting vaults — no one can intercept from outside.

In TVS OTA, firmware packages (up to 2GB each) go to Blob Storage, OTA campaign tracking to PostgreSQL, and signing keys + TLS certificates to Key Vault. ALL accessed via Private Endpoints — zero public internet exposure. This is defense-in-depth for data services.

**தமிழ்:**
Azure data services-ஐ ஒரு bank-ன் vault system போல நினைக்கவும். Blob Storage document vault — millions files (firmware packages) store செய்கிறது, different access tiers-உடன். PostgreSQL ledger book — structured records (OTA campaign metadata). Key Vault master safe — மற்ற எல்லாவற்றையும் unlock செய்யும் keys வைக்கிறது (PKI certificates, signing keys). Private Endpoints underground tunnels — வெளியிலிருந்து யாரும் intercept செய்ய முடியாது.

TVS OTA-ல், firmware packages (2GB வரை) Blob Storage-க்கு, OTA campaign tracking PostgreSQL-க்கு, signing keys + TLS certificates Key Vault-க்கு. அனைத்தும் Private Endpoints வழியாக access — zero public internet exposure. இது data services-க்கான defense-in-depth.

---

## 📊 Architecture & Concepts

### Data Service Architecture (TVS OTA)

```
┌──────────────────────────────────────────────────────────────┐
│                    AKS Cluster (Private)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐     │
│  │ OTA Service │  │ PKI Service │  │ Campaign Service │     │
│  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘     │
└─────────┼────────────────┼───────────────────┼───────────────┘
          │                │                   │
          │ Private        │ Private           │ Private
          │ Endpoint       │ Endpoint          │ Endpoint
          ▼                ▼                   ▼
┌─────────────────┐ ┌────────────┐  ┌──────────────────────┐
│  Blob Storage   │ │  Key Vault │  │ PostgreSQL Flexible  │
│                 │ │            │  │                      │
│ firmware/       │ │ PKI certs  │  │ ota_campaigns DB     │
│   v2.1.0.bin   │ │ TLS certs  │  │ device_registry DB   │
│   v2.0.9.bin   │ │ Signing    │  │ rollout_history DB   │
│ configs/       │ │ keys       │  │                      │
│   device.json  │ │            │  │                      │
│                 │ │ RBAC:      │  │ VNet Integration     │
│ Lifecycle:     │ │ Workload   │  │ (delegated subnet)   │
│  Hot→Cool→Arch │ │ Identity   │  │                      │
└─────────────────┘ └────────────┘  └──────────────────────┘
     No public          No public          No public
     access             access             access
```

### Blob Storage Tiers & Lifecycle

| Tier | Access Cost | Storage Cost | Use Case |
|------|-------------|--------------|----------|
| Hot | Low | High | Active firmware, configs |
| Cool | Medium | Medium | Previous versions (30+ days) |
| Archive | High | Lowest | Compliance retention (1+ year) |

**English:**
Lifecycle management automates tier transitions:
- Firmware active → 30 days → move to Cool (60% cheaper storage)
- 90 days → move to Archive (90% cheaper storage)
- 365 days → delete (unless compliance requires longer)

**தமிழ்:**
Lifecycle management tier transitions-ஐ automate செய்கிறது:
- Firmware active → 30 நாட்கள் → Cool-க்கு move (60% cheaper)
- 90 நாட்கள் → Archive-க்கு move (90% cheaper)
- 365 நாட்கள் → delete (compliance தேவைப்படாவிட்டால்)

### PostgreSQL Flexible Server vs Single Server

| Feature | Flexible Server | Single Server (Legacy) |
|---------|----------------|----------------------|
| HA | Zone-redundant | Limited |
| VNet Integration | Native (delegated subnet) | VNet rules only |
| Maintenance | Custom window | Fixed |
| Scaling | Online vertical | Requires restart |
| Cost | Pay per use | Always running |
| Status | **Current** | Retirement path |

**Always use Flexible Server** — Single Server is being retired.

### Key Vault Access Models

```
RBAC (Recommended)              vs       Access Policies (Legacy)
─────────────────                        ─────────────────────────
Azure RBAC roles:                        Vault-level permissions:
- Key Vault Secrets User                 - GET, LIST, SET per principal
- Key Vault Crypto User                  - No granular scope
- Key Vault Certificates Officer         - Max 1024 policies
                                         
✓ Granular (per-secret scope)            ✗ Vault-level only
✓ Conditional access                     ✗ No conditions
✓ PIM eligible assignments               ✗ No PIM
✓ Audit via Azure RBAC logs              ✗ Separate audit
```

---

## 🧠 Byheart for Interview

### Security Checklist for Data Services
```
1. Private Endpoints for ALL PaaS (Blob, PostgreSQL, Key Vault)
2. Disable public network access on every resource
3. Key Vault RBAC mode (not access policies)
4. Workload Identity for application access (no connection strings in code)
5. Customer-managed keys (CMK) for encryption at rest
6. Soft delete + purge protection on Key Vault
7. PostgreSQL: SSL enforce, AAD-only auth, no password auth
8. Blob: immutable storage for compliance, versioning enabled
9. Network: Only allow from specific subnets via Private Endpoints
10. Monitoring: Diagnostic logs to Log Analytics for all services
```

### Key Terraform Patterns
```
Storage Account:
  - public_network_access_enabled = false
  - min_tls_version = "TLS1_2"
  - shared_access_key_enabled = false (use RBAC)
  - blob_properties.versioning_enabled = true

PostgreSQL:
  - public_network_access_enabled = false
  - delegated_subnet_id (VNet integration)
  - authentication.active_directory_auth_enabled = true
  - authentication.password_auth_enabled = false
  - high_availability.mode = "ZoneRedundant"

Key Vault:
  - enable_rbac_authorization = true
  - purge_protection_enabled = true
  - soft_delete_retention_days = 90
  - public_network_access_enabled = false
  - network_acls.default_action = "Deny"
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/azure-data && cd ~/tf-lab/azure-data
```

### Exercise 1: Secure Blob Storage

```hcl
cat > storage.tf << 'EOF'
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

# ─── STORAGE ACCOUNT ─────────────────────────────
resource "azurerm_storage_account" "firmware" {
  name                          = "st${var.project}fw${var.environment}"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  account_tier                  = "Standard"
  account_replication_type      = "GRS"  # Geo-redundant for firmware
  min_tls_version               = "TLS1_2"
  public_network_access_enabled = false  # Only via Private Endpoint
  shared_access_key_enabled     = false  # Force RBAC, no keys

  blob_properties {
    versioning_enabled       = true  # Rollback capability
    change_feed_enabled      = true  # Track all changes
    last_access_time_enabled = true  # For lifecycle management

    delete_retention_policy {
      days = 30  # Soft delete
    }

    container_delete_retention_policy {
      days = 7
    }
  }

  identity {
    type = "SystemAssigned"
  }

  tags = var.tags
}

# ─── CONTAINERS ──────────────────────────────────
resource "azurerm_storage_container" "firmware" {
  name                  = "firmware"
  storage_account_name  = azurerm_storage_account.firmware.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "configs" {
  name                  = "configs"
  storage_account_name  = azurerm_storage_account.firmware.name
  container_access_type = "private"
}

# ─── LIFECYCLE MANAGEMENT ────────────────────────
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.firmware.id

  rule {
    name    = "firmware-lifecycle"
    enabled = true

    filters {
      prefix_match = ["firmware/"]
      blob_types   = ["blockBlob"]
    }

    actions {
      base_blob {
        tier_to_cool_after_days_since_last_access_time_greater_than    = 30
        tier_to_archive_after_days_since_last_access_time_greater_than = 90
        delete_after_days_since_last_access_time_greater_than          = 365
      }
    }
  }
}

# ─── RBAC: Grant AKS Workload Identity access ───
resource "azurerm_role_assignment" "ota_blob_reader" {
  scope                = azurerm_storage_account.firmware.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = var.ota_identity_principal_id
}

resource "azurerm_role_assignment" "ota_blob_writer" {
  scope                = azurerm_storage_container.firmware.resource_manager_id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = var.upload_identity_principal_id
}
EOF
```

### Exercise 2: PostgreSQL Flexible Server

```hcl
cat > postgresql.tf << 'EOF'
# ─── POSTGRESQL FLEXIBLE SERVER ──────────────────
resource "azurerm_postgresql_flexible_server" "main" {
  name                          = "psql-${var.project}-${var.environment}"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  version                       = "15"
  sku_name                      = "GP_Standard_D4s_v3"  # General Purpose
  storage_mb                    = 131072                  # 128GB
  zone                          = "1"
  public_network_access_enabled = false

  # VNet Integration (delegated subnet)
  delegated_subnet_id = var.database_subnet_id
  private_dns_zone_id = azurerm_private_dns_zone.postgresql.id

  # AAD-only authentication (no passwords!)
  authentication {
    active_directory_auth_enabled = true
    password_auth_enabled         = false
    tenant_id                     = var.tenant_id
  }

  # HA: Zone-redundant
  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }

  # Maintenance window
  maintenance_window {
    day_of_week  = 0  # Sunday
    start_hour   = 2
    start_minute = 0
  }

  # Backup
  backup_retention_days        = 35
  geo_redundant_backup_enabled = true

  tags = var.tags
}

# ─── DATABASE ────────────────────────────────────
resource "azurerm_postgresql_flexible_server_database" "ota" {
  name      = "ota_campaigns"
  server_id = azurerm_postgresql_flexible_server.main.id
  charset   = "utf8"
  collation = "en_US.utf8"
}

# ─── AAD Admin ───────────────────────────────────
resource "azurerm_postgresql_flexible_server_active_directory_administrator" "admin" {
  server_name         = azurerm_postgresql_flexible_server.main.name
  resource_group_name = var.resource_group_name
  tenant_id           = var.tenant_id
  object_id           = var.dba_group_object_id
  principal_name      = var.dba_group_name
  principal_type      = "Group"
}

# ─── Private DNS Zone ────────────────────────────
resource "azurerm_private_dns_zone" "postgresql" {
  name                = "privatelink.postgres.database.azure.com"
  resource_group_name = var.resource_group_name
}

resource "azurerm_private_dns_zone_virtual_network_link" "postgresql" {
  name                  = "psql-vnet-link"
  resource_group_name   = var.resource_group_name
  private_dns_zone_name = azurerm_private_dns_zone.postgresql.name
  virtual_network_id    = var.vnet_id
}

# ─── Server Configuration ────────────────────────
resource "azurerm_postgresql_flexible_server_configuration" "log_connections" {
  name      = "log_connections"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "ON"
}

resource "azurerm_postgresql_flexible_server_configuration" "connection_throttle" {
  name      = "connection_throttle.enable"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "ON"
}
EOF
```

### Exercise 3: Key Vault with RBAC

```hcl
cat > keyvault.tf << 'EOF'
# ─── KEY VAULT ───────────────────────────────────
resource "azurerm_key_vault" "main" {
  name                          = "kv-${var.project}-${var.environment}"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  tenant_id                     = var.tenant_id
  sku_name                      = "premium"  # HSM-backed keys for PKI
  enable_rbac_authorization     = true       # RBAC mode (not access policies)
  purge_protection_enabled      = true       # Cannot permanently delete
  soft_delete_retention_days    = 90
  public_network_access_enabled = false      # Private Endpoint only

  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }

  tags = var.tags
}

# ─── RBAC: Workload Identity for PKI service ────
resource "azurerm_role_assignment" "pki_crypto" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Crypto User"
  principal_id         = var.pki_identity_principal_id
}

resource "azurerm_role_assignment" "app_secrets" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = var.app_identity_principal_id
}

resource "azurerm_role_assignment" "cert_officer" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Certificates Officer"
  principal_id         = var.cert_manager_principal_id
}

# ─── Private Endpoint for Key Vault ──────────────
resource "azurerm_private_endpoint" "keyvault" {
  name                = "pe-kv-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.private_endpoint_subnet_id

  private_service_connection {
    name                           = "kv-connection"
    private_connection_resource_id = azurerm_key_vault.main.id
    is_manual_connection           = false
    subresource_names              = ["vault"]
  }

  private_dns_zone_group {
    name                 = "kv-dns"
    private_dns_zone_ids = [azurerm_private_dns_zone.keyvault.id]
  }
}

resource "azurerm_private_dns_zone" "keyvault" {
  name                = "privatelink.vaultcore.azure.net"
  resource_group_name = var.resource_group_name
}

resource "azurerm_private_dns_zone_virtual_network_link" "keyvault" {
  name                  = "kv-vnet-link"
  resource_group_name   = var.resource_group_name
  private_dns_zone_name = azurerm_private_dns_zone.keyvault.name
  virtual_network_id    = var.vnet_id
}

# ─── Diagnostic Logging ──────────────────────────
resource "azurerm_monitor_diagnostic_setting" "keyvault" {
  name                       = "kv-diagnostics"
  target_resource_id         = azurerm_key_vault.main.id
  log_analytics_workspace_id = var.log_analytics_workspace_id

  enabled_log {
    category = "AuditEvent"
  }
  enabled_log {
    category = "AzurePolicyEvaluationDetails"
  }

  metric {
    category = "AllMetrics"
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

### Scenario: "How do you secure data services for a production platform?"

**Requirements:**
- Firmware files (up to 2GB) for OTA delivery to 500K devices
- Campaign metadata (PostgreSQL) with ACID transactions
- PKI certificates and signing keys (HSM-grade protection)
- Compliance: no public internet access to any data store
- Audit trail for all secret/key access
- Disaster recovery across regions

**Your Design Answer:**

```
┌─────────────────────────────────────────────────────────────┐
│ Secure Data Architecture                                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Blob Storage (Firmware):                                    │
│ ├── GRS replication (geo-redundant)                         │
│ ├── public_network_access = false                           │
│ ├── shared_access_key_enabled = false (RBAC only)          │
│ ├── Versioning + soft delete (rollback)                     │
│ ├── Lifecycle: Hot→Cool(30d)→Archive(90d)→Delete(365d)     │
│ ├── Immutable storage for audit packages                    │
│ └── Private Endpoint + Private DNS                          │
│                                                             │
│ PostgreSQL Flexible Server:                                  │
│ ├── VNet Integration (delegated subnet)                     │
│ ├── AAD-only auth (no passwords)                            │
│ ├── Zone-redundant HA                                       │
│ ├── Geo-redundant backup (35 days)                          │
│ ├── SSL enforced, TLS 1.2 minimum                          │
│ ├── Connection throttling enabled                           │
│ └── Private DNS zone linked                                 │
│                                                             │
│ Key Vault (Premium — HSM):                                  │
│ ├── RBAC mode (not access policies)                         │
│ ├── Purge protection + 90-day soft delete                   │
│ ├── public_network_access = false                           │
│ ├── Diagnostic logging (all AuditEvents)                    │
│ ├── Workload Identity access (per-service)                  │
│ └── Private Endpoint + Private DNS                          │
│                                                             │
│ Cross-cutting:                                              │
│ ├── All access via Workload Identity (no stored creds)      │
│ ├── Customer-managed keys for encryption                    │
│ ├── Log Analytics for audit trail                           │
│ ├── Azure Policy: enforce private endpoints                 │
│ └── Terraform: separate state per environment               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Real Project Reference: TVS OTA Data Layer

**English:**
In TVS Connected Vehicle OTA platform:

**Blob Storage:**
- Stores firmware packages (500MB-2GB each, 200+ versions)
- GRS replication — firmware accessible even if primary region fails
- Lifecycle management: active firmware in Hot, older in Cool, compliance copies in Archive
- RBAC-only access — upload service gets Contributor, OTA delivery gets Reader
- Immutable containers for signed release packages (compliance audit)

**PostgreSQL:**
- Campaign management: which devices get which firmware, rollout state
- Zone-redundant HA — zero downtime during failover
- AAD-only auth — microservices use Workload Identity tokens
- 35-day backup retention + geo-redundant for DR
- Connection pooling via PgBouncer sidecar in AKS

**Key Vault (Premium):**
- PKI signing keys (HSM-backed, never exportable)
- TLS certificates for all internal services (auto-rotated via cert-manager)
- Key Vault CSI driver mounts secrets directly into pods
- Separate Key Vaults per environment (no cross-env access)
- Full audit logging — every access tracked

**தமிழ்:**
TVS OTA platform data layer:

**Blob Storage:**
- Firmware packages store (500MB-2GB, 200+ versions)
- GRS replication — primary region fail ஆனாலும் firmware accessible
- Lifecycle: active → Hot, older → Cool, compliance → Archive
- RBAC-only: upload service-க்கு Contributor, OTA delivery-க்கு Reader

**PostgreSQL:**
- Campaign management: devices-க்கு firmware assignment, rollout state
- Zone-redundant HA — failover-ல் zero downtime
- AAD-only auth — microservices Workload Identity tokens பயன்படுத்தும்
- 35-day backup + geo-redundant DR-க்கு

**Key Vault (Premium):**
- PKI signing keys (HSM-backed, export முடியாது)
- TLS certificates auto-rotated (cert-manager வழியாக)
- Key Vault CSI driver secrets-ஐ pods-க்கு mount செய்யும்
- Environment-க்கு தனி Key Vault (cross-env access இல்லை)

---

## 🎤 Interview Q&A

### Q1: "How do you secure Azure Storage for production?"

**English Answer:**
"Five layers of security: First, disable public network access entirely — all access through Private Endpoints only. Second, disable shared access keys — force RBAC-based access using Workload Identity, eliminating connection strings. Third, enable versioning and soft delete — protection against accidental deletion. Fourth, lifecycle management for cost optimization — move to Cool tier after 30 days, Archive after 90. Fifth, immutable storage for compliance packages — write-once, cannot be modified or deleted until retention expires. In Terraform, key attributes are `public_network_access_enabled = false` and `shared_access_key_enabled = false`."

**தமிழ் Answer:**
"ஐந்து security layers: ஒன்று, public network access disable — Private Endpoints வழியாக மட்டும் access. இரண்டு, shared access keys disable — RBAC + Workload Identity, connection strings eliminate. மூன்று, versioning + soft delete — accidental deletion protection. நான்கு, lifecycle management — 30 days Cool, 90 days Archive. ஐந்து, immutable storage compliance packages-க்கு — retention expire ஆகும் வரை modify/delete முடியாது. Terraform: `public_network_access_enabled = false`, `shared_access_key_enabled = false`."

### Q2: "PostgreSQL Flexible Server — why and how do you secure it?"

**English Answer:**
"Flexible Server over Single Server — it's the current generation with native VNet integration, zone-redundant HA, and custom maintenance windows. Security: First, VNet integration via delegated subnet — server lives in your VNet, no public IP at all. Second, AAD-only authentication — disable password auth completely, services use managed identity tokens. Third, SSL enforced with TLS 1.2 minimum. Fourth, connection throttling to prevent brute force. Fifth, diagnostic logging for all connections and queries. The delegated subnet approach is stronger than Private Endpoints for PostgreSQL because the server itself is IN your network, not just accessible from it."

**தமிழ் Answer:**
"Flexible Server — current generation, native VNet integration, zone-redundant HA. Security: ஒன்று, VNet integration delegated subnet வழியாக — server உங்கள் VNet-ல் இருக்கும், public IP இல்லை. இரண்டு, AAD-only authentication — password auth disable, services managed identity tokens பயன்படுத்தும். மூன்று, SSL enforced TLS 1.2 minimum. நான்கு, connection throttling brute force தடுக்க. ஐந்து, diagnostic logging. Delegated subnet approach Private Endpoints-ஐ விட strong — server உங்கள் network-ல் இருக்கும்."

### Q3: "Key Vault — RBAC vs Access Policies?"

**English Answer:**
"Always RBAC for new deployments. Three reasons: First, granular scope — you can grant access to individual secrets, not the entire vault. Second, conditional access and PIM eligible assignments — time-bound access for operators. Third, consistent audit trail through Azure RBAC logs — same as all other Azure resources. Access policies are legacy — limited to 1024 entries, vault-level only (all or nothing per user), no conditions. In Terraform: `enable_rbac_authorization = true` on the vault, then `azurerm_role_assignment` for each identity with specific roles like `Key Vault Secrets User` or `Key Vault Crypto User`."

**தமிழ் Answer:**
"புதிய deployments-க்கு எப்போதும் RBAC. மூன்று காரணங்கள்: ஒன்று, granular scope — individual secrets-க்கு access grant செய்யலாம். இரண்டு, conditional access + PIM — operators-க்கு time-bound access. மூன்று, Azure RBAC logs வழியாக consistent audit trail. Access policies legacy — 1024 entries limit, vault-level only, no conditions. Terraform: vault-ல் `enable_rbac_authorization = true`, பின் `azurerm_role_assignment` ஒவ்வொரு identity-க்கும்."

### Q4: "How do you handle disaster recovery for data services?"

**English Answer:**
"Different strategies per service: Blob Storage — GRS replication provides automatic geo-redundant copies, failover is automatic for reads. PostgreSQL — geo-redundant backups with 35-day retention, can restore to secondary region, plus read replicas for active-passive DR. Key Vault — soft delete plus purge protection prevents accidental loss, but for true DR I maintain parallel Key Vaults in secondary region with same secrets synced via CI/CD pipeline (never copy keys — regenerate in DR vault). Cross-cutting: all Terraform state in geo-redundant storage account, and I test DR runbook quarterly."

**தமிழ் Answer:**
"ஒவ்வொரு service-க்கும் different strategies: Blob Storage — GRS replication automatic geo-redundant copies. PostgreSQL — geo-redundant backups 35-day retention, secondary region-ல் restore, read replicas active-passive DR-க்கு. Key Vault — soft delete + purge protection, true DR-க்கு secondary region-ல் parallel Key Vaults (keys copy செய்யாதீர்கள் — DR vault-ல் regenerate). Terraform state geo-redundant storage-ல், DR runbook quarterly test."

### Q5: "How does Workload Identity replace connection strings?"

**English Answer:**
"Traditional approach: connection string with password in K8s secret or environment variable — terrible security. With Workload Identity: Pod has a Kubernetes ServiceAccount annotated with Azure Managed Identity client ID. When pod starts, it gets an Azure AD token via OIDC federation — no secrets anywhere. For PostgreSQL, the app uses `DefaultAzureCredential` which gets a token and authenticates via AAD. For Blob Storage, same — token-based RBAC access. For Key Vault, the CSI driver handles token acquisition. Result: zero connection strings in code, Git, or K8s secrets. If a pod is compromised, the token expires in hours and is scoped to that specific identity's permissions."

**தமிழ் Answer:**
"Traditional: connection string with password K8s secret-ல் — terrible security. Workload Identity-உடன்: Pod-க்கு annotated ServiceAccount, OIDC federation வழியாக Azure AD token கிடைக்கும் — secrets எங்கும் இல்லை. PostgreSQL-க்கு `DefaultAzureCredential` token வழியாக AAD authenticate. Blob Storage-க்கும் token-based RBAC. Key Vault CSI driver token acquisition handle செய்யும். Result: code, Git, K8s secrets-ல் zero connection strings. Pod compromise ஆனாலும், token hours-ல் expire, specific permissions-க்கு scoped."

---

## ✅ Self-Check

- [ ] Can you design secure Blob Storage (no public access, RBAC-only, lifecycle)?
- [ ] Can you explain PostgreSQL Flexible Server VNet integration vs Private Endpoint?
- [ ] Can you configure Key Vault with RBAC (not access policies)?
- [ ] Can you explain the complete Private Endpoint + Private DNS flow?
- [ ] Can you describe Workload Identity flow for database access?
- [ ] Can you design lifecycle management for firmware storage?
- [ ] Can you explain DR strategies for each data service?
- [ ] Can you justify the TVS data architecture decisions?
- [ ] Can you explain Key Vault purge protection and soft delete?
- [ ] Can you describe the security audit trail (diagnostic settings)?
