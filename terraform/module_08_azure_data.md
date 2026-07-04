# Module 08: Azure Storage & Data
# மாடுல் 08: Azure Storage & Data Services

---

## 🎯 What? | என்ன?

**English:** Provision Azure data services with Terraform — Blob Storage (firmware packages), PostgreSQL (metadata), Key Vault (secrets/certs), all with Private Endpoints.

**தமிழ்:** Azure data services-ஐ Terraform-ல் provision — Blob Storage (firmware), PostgreSQL (metadata), Key Vault (secrets), Private Endpoints-உடன்.

---

## 🛠️ Blob Storage (OTA Firmware Packages)

```hcl
resource "azurerm_storage_account" "firmware" {
  name                     = "stfirmware${var.environment}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"    # Geo-redundant for DR
  
  # Security
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false    # No public blob access!
  public_network_access_enabled   = false    # Private Endpoint only!

  blob_properties {
    versioning_enabled = true    # Keep old firmware versions
    delete_retention_policy {
      days = 30                  # Soft delete protection
    }
  }

  tags = local.common_tags
}

resource "azurerm_storage_container" "packages" {
  name                  = "firmware-packages"
  storage_account_name  = azurerm_storage_account.firmware.name
  container_access_type = "private"
}

# Private Endpoint for Storage
resource "azurerm_private_endpoint" "storage" {
  name                = "pe-storage-${var.environment}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "psc-storage"
    private_connection_resource_id = azurerm_storage_account.firmware.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}
```

---

## 🛠️ PostgreSQL Flexible Server

```hcl
resource "azurerm_postgresql_flexible_server" "main" {
  name                = "psql-${var.project}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  version             = "14"
  
  # Compute
  sku_name = var.environment == "prod" ? "GP_Standard_D4s_v3" : "B_Standard_B2s"
  
  # Storage
  storage_mb = 65536    # 64 GB
  
  # Network (private access via delegated subnet)
  delegated_subnet_id = azurerm_subnet.postgres.id
  private_dns_zone_id = azurerm_private_dns_zone.postgres.id
  
  # Auth
  administrator_login    = "pgadmin"
  administrator_password = var.pg_admin_password    # From Key Vault!

  # HA
  high_availability {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }

  # Backup
  backup_retention_days        = 35
  geo_redundant_backup_enabled = var.environment == "prod"

  tags = local.common_tags
}

# Database
resource "azurerm_postgresql_flexible_server_database" "ota" {
  name      = "ota_db"
  server_id = azurerm_postgresql_flexible_server.main.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}

# Firewall (allow AKS subnet)
resource "azurerm_postgresql_flexible_server_firewall_rule" "aks" {
  name             = "allow-aks-subnet"
  server_id        = azurerm_postgresql_flexible_server.main.id
  start_ip_address = "10.1.1.0"
  end_ip_address   = "10.1.1.255"
}
```

---

## 🛠️ Key Vault (Secrets, Certificates, Keys)

```hcl
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.project}-${var.environment}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "premium"    # HSM-backed for PKI root keys!

  # Security
  purge_protection_enabled   = true    # Cannot permanently delete
  soft_delete_retention_days = 90
  
  # Network
  public_network_access_enabled = false

  # RBAC (not access policy!)
  enable_rbac_authorization = true

  tags = local.common_tags
}

# Grant AKS access to secrets
resource "azurerm_role_assignment" "aks_kv" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_kubernetes_cluster.ota.key_vault_secrets_provider[0].secret_identity[0].object_id
}

# Store database password in Key Vault
resource "azurerm_key_vault_secret" "pg_password" {
  name         = "pg-admin-password"
  value        = var.pg_admin_password
  key_vault_id = azurerm_key_vault.main.id

  depends_on = [azurerm_role_assignment.kv_admin]
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│       AZURE DATA SERVICES CHEAT SHEET            │
├──────────────────────────────────────────────────┤
│ BLOB STORAGE:                                    │
│   Firmware packages, build artifacts, backups    │
│   GRS = geo-redundant (DR across regions)        │
│   Versioning = keep old firmware versions        │
│   Private Endpoint = no public access            │
│                                                  │
│ POSTGRESQL:                                      │
│   Flexible Server (newest, recommended)          │
│   ZoneRedundant HA for production                │
│   Delegated subnet (VNet integration)            │
│   35-day backup retention                        │
│                                                  │
│ KEY VAULT:                                       │
│   Premium SKU = HSM-backed (for PKI!)            │
│   RBAC mode (not access policies)                │
│   Purge protection = can't permanently delete    │
│   AKS CSI driver = mount secrets as volumes      │
│                                                  │
│ SECURITY PATTERN:                                │
│   All data services: Private Endpoint            │
│   Credentials: Key Vault (never in .tfvars!)     │
│   Encryption: Customer-managed keys (CMK)        │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: How do you manage database credentials in Terraform?**
- Generate password externally (or random_password resource)
- Store in Key Vault immediately
- Reference via data source in other configs
- Never commit to Git, never in terraform.tfvars
- AKS pods access via Workload Identity + CSI driver

**Q: Why Premium Key Vault for PKI?**
- HSM-backed keys (hardware security module)
- Root CA private key never leaves HSM
- Required for automotive security compliance (ISO 21434)
- Certificate operations happen inside HSM boundary

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Storage Account with Private Endpoint provision முடியும்
- [ ] PostgreSQL Flexible Server with HA configure முடியும்
- [ ] Key Vault with RBAC setup முடியும்
- [ ] Secrets management strategy explain முடியும்
