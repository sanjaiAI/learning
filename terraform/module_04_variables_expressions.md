# Module 04: Variables & Expressions
# மாடுல் 04: Variables & Expressions

---

## 🎯 What? | என்ன?

**English:** Variables make Terraform code dynamic. Expressions (conditionals, loops, functions) let you write flexible, DRY infrastructure code.

**தமிழ்:** Variables Terraform code-ஐ dynamic ஆக்குகிறது. Expressions (conditionals, loops, functions) flexible, DRY code எழுத உதவுகிறது.

---

## 🔑 Variable Types | Variable வகைகள்

```hcl
# String
variable "location" {
  type    = string
  default = "eastus"
}

# Number
variable "node_count" {
  type    = number
  default = 3
}

# Bool
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List
variable "allowed_ips" {
  type    = list(string)
  default = ["10.0.0.0/8", "172.16.0.0/12"]
}

# Map
variable "tags" {
  type = map(string)
  default = {
    environment = "production"
    team        = "platform"
  }
}

# Object (structured)
variable "node_pool" {
  type = object({
    name       = string
    vm_size    = string
    count      = number
    max_pods   = optional(number, 30)  # Optional with default!
  })
}

# Validation
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

---

## 🛠️ Locals | Local Values

```hcl
# Locals = computed values (DRY, no repetition)
locals {
  env_prefix   = "${var.project}-${var.environment}"
  is_prod      = var.environment == "prod"
  
  common_tags = {
    environment = var.environment
    project     = var.project
    managed_by  = "terraform"
    cost_center = var.cost_center
  }
  
  # Merge maps
  all_tags = merge(local.common_tags, var.extra_tags)
}

# Use: local.env_prefix, local.common_tags
resource "azurerm_resource_group" "main" {
  name     = "rg-${local.env_prefix}"
  location = var.location
  tags     = local.all_tags
}
```

---

## 🛠️ Conditionals | நிபந்தனைகள்

```hcl
# Ternary: condition ? true_value : false_value
resource "azurerm_kubernetes_cluster" "aks" {
  name       = "aks-${var.environment}"
  node_count = var.environment == "prod" ? 5 : 2
  vm_size    = var.environment == "prod" ? "Standard_D4s_v3" : "Standard_D2s_v3"
}

# Conditional resource (count trick)
resource "azurerm_monitor_diagnostic_setting" "logs" {
  count = var.enable_monitoring ? 1 : 0   # 0 = don't create!
  name  = "diag-${var.environment}"
  # ...
}

# Reference conditional resource:
# azurerm_monitor_diagnostic_setting.logs[0].id  (must use [0])
```

---

## 🛠️ Loops | Loops (for_each, count, for)

```hcl
# --- for_each (MAP — preferred!) ---
variable "subnets" {
  type = map(string)
  default = {
    aks  = "10.0.1.0/24"
    data = "10.0.2.0/24"
    ci   = "10.0.3.0/24"
  }
}

resource "azurerm_subnet" "this" {
  for_each             = var.subnets
  name                 = "snet-${each.key}"         # "snet-aks", "snet-data"
  address_prefixes     = [each.value]               # "10.0.1.0/24"
  virtual_network_name = azurerm_virtual_network.main.name
  resource_group_name  = azurerm_resource_group.main.name
}
# Reference: azurerm_subnet.this["aks"].id

# --- for_each with OBJECTS ---
variable "node_pools" {
  type = map(object({
    vm_size    = string
    count      = number
    max_pods   = number
  }))
  default = {
    system = { vm_size = "Standard_D4s_v3", count = 3, max_pods = 30 }
    user   = { vm_size = "Standard_D8s_v3", count = 5, max_pods = 50 }
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "pools" {
  for_each              = var.node_pools
  name                  = each.key
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = each.value.vm_size
  node_count            = each.value.count
  max_pods              = each.value.max_pods
}

# --- for expression (transform data) ---
locals {
  subnet_ids = { for k, v in azurerm_subnet.this : k => v.id }
  # Result: { aks = "/sub/.../snet-aks", data = "/sub/.../snet-data" }
  
  # Filter
  prod_pools = { for k, v in var.node_pools : k => v if v.count > 3 }
}

# --- count (simple list iteration) ---
variable "availability_zones" {
  default = ["1", "2", "3"]
}

resource "azurerm_public_ip" "lb" {
  count    = length(var.availability_zones)
  name     = "pip-lb-${count.index}"
  zones    = [var.availability_zones[count.index]]
  # ...
}
```

---

## 🛠️ Dynamic Blocks | Dynamic Blocks

```hcl
# Dynamic = loop INSIDE a resource block
variable "nsg_rules" {
  type = list(object({
    name     = string
    port     = string
    priority = number
  }))
  default = [
    { name = "ssh",   port = "22",   priority = 100 },
    { name = "https", port = "443",  priority = 200 },
    { name = "mqtt",  port = "8883", priority = 300 },
  ]
}

resource "azurerm_network_security_group" "this" {
  name                = "nsg-production"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name

  dynamic "security_rule" {
    for_each = var.nsg_rules
    content {
      name                       = security_rule.value.name
      priority                   = security_rule.value.priority
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = security_rule.value.port
      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }
  }
}
```

---

## 🛠️ Built-in Functions | Functions

```hcl
locals {
  # String
  upper_name = upper(var.name)                    # "PRODUCTION"
  formatted  = format("aks-%s-%s", var.env, var.region)  # "aks-prod-eastus"
  joined     = join("-", ["aks", var.env, var.region])    # "aks-prod-eastus"
  
  # Collection
  flat_list  = flatten([var.list1, var.list2])     # Merge lists
  keys       = keys(var.tags)                      # Map → key list
  lookup_val = lookup(var.tags, "team", "unknown") # Safe map lookup
  
  # File
  config     = file("${path.module}/config.json")  # Read file content
  template   = templatefile("${path.module}/user-data.sh", { hostname = var.name })
  
  # Encoding
  encoded    = base64encode("secret-value")
  decoded    = jsondecode(file("config.json"))
  
  # IP/CIDR
  subnet     = cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
  
  # Type conversion
  to_number  = tonumber("3")
  to_list    = tolist(toset(var.duplicated_list))  # Remove duplicates
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│      VARIABLES & EXPRESSIONS CHEAT SHEET         │
├──────────────────────────────────────────────────┤
│ VARIABLE PRECEDENCE (highest → lowest):          │
│   1. -var flag (CLI)                             │
│   2. *.auto.tfvars (alphabetical)                │
│   3. terraform.tfvars                            │
│   4. Environment: TF_VAR_name                    │
│   5. Default in variable block                   │
│                                                  │
│ LOOPS:                                           │
│   for_each = iterate MAP/SET (preferred!)        │
│   count    = iterate by number (fragile)         │
│   dynamic  = loop INSIDE resource blocks         │
│                                                  │
│ WHY for_each > count:                            │
│   count: removing item [1] shifts all indexes    │
│   for_each: removing "aks" only affects "aks"    │
│                                                  │
│ COMMON FUNCTIONS:                                │
│   merge()       = combine maps                   │
│   lookup()      = safe map access                │
│   flatten()     = merge nested lists             │
│   cidrsubnet()  = calculate subnet CIDRs         │
│   templatefile()= render templates               │
│   try()         = fallback on error              │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: for_each vs count — when to use which?**
- `for_each` with maps: stable (removing one item doesn't affect others). Use for most cases.
- `count`: only for "create N identical things" or conditional (count = 0 or 1).
- Never use count with lists of unique items (reordering causes recreate!).

**Q: How to pass sensitive variables securely?**
- Never in `.tfvars` committed to Git
- Environment variable: `TF_VAR_db_password`
- CI/CD secrets → injected at runtime
- Or: reference Key Vault/Secret Manager via data source

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Variable types (string, list, map, object) use முடியும்
- [ ] for_each with map/object use முடியும்
- [ ] Dynamic blocks write முடியும்
- [ ] Variable precedence explain முடியும்
- [ ] Common functions (merge, lookup, cidrsubnet) use முடியும்
