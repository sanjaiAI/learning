# Module 03: Modules & Reusability
# மாடுல் 03: Modules & Reusability (மறுபயன்பாடு)

---

## 🎯 What? | என்ன?

**English:** Modules = reusable Terraform packages. Write once, use everywhere. Like functions in programming — define inputs, get outputs, hide complexity.

**தமிழ்:** Modules = reusable Terraform packages. ஒரு தடவை எழுது, எல்லா இடத்திலும் பயன்படுத்து. Programming functions மாதிரி — inputs define, outputs receive, complexity hide.

### Analogy | உதாரணம்
> LEGO blocks: Each module = a pre-built block (wheel, door, window). Combine blocks to build different houses without rebuilding each piece.

> LEGO: Module = pre-built block. Combine to build different structures without redoing.

---

## 📊 Module Architecture | Module Structure

```mermaid
graph TB
    ROOT[Root Module<br/>main.tf] --> NET[module "networking"<br/>VNet, Subnets, NSG]
    ROOT --> AKS[module "aks"<br/>Cluster, Node Pools]
    ROOT --> DATA[module "database"<br/>PostgreSQL, Private Endpoint]
    
    NET -->|output: subnet_ids| AKS
    NET -->|output: vnet_id| DATA
    AKS -->|output: kubelet_identity| DATA
```

### Module File Structure

```
modules/
├── networking/
│   ├── main.tf          # Resources (VNet, Subnets, NSG)
│   ├── variables.tf     # Inputs (address_space, subnet_names)
│   ├── outputs.tf       # Outputs (vnet_id, subnet_ids)
│   └── README.md        # Usage documentation
├── aks/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf
└── database/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

---

## 🛠️ Creating a Module | Module Create செய்வது

### Example: AKS Module

```hcl
# modules/aks/variables.tf
variable "cluster_name" {
  type        = string
  description = "Name of the AKS cluster"
}

variable "resource_group_name" {
  type = string
}

variable "location" {
  type    = string
  default = "eastus"
}

variable "node_count" {
  type    = number
  default = 3
}

variable "vm_size" {
  type    = string
  default = "Standard_D4s_v3"
}

variable "subnet_id" {
  type        = string
  description = "Subnet to deploy AKS nodes into"
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/aks/main.tf
resource "azurerm_kubernetes_cluster" "this" {
  name                = var.cluster_name
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.cluster_name

  default_node_pool {
    name           = "system"
    node_count     = var.node_count
    vm_size        = var.vm_size
    vnet_subnet_id = var.subnet_id
    
    upgrade_settings {
      max_surge = "25%"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
  }

  tags = var.tags
}
```

```hcl
# modules/aks/outputs.tf
output "cluster_id" {
  value = azurerm_kubernetes_cluster.this.id
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.this.kube_config_raw
  sensitive = true
}

output "kubelet_identity" {
  value = azurerm_kubernetes_cluster.this.kubelet_identity[0].object_id
}

output "cluster_name" {
  value = azurerm_kubernetes_cluster.this.name
}
```

---

## 🛠️ Using a Module | Module பயன்படுத்துவது

```hcl
# environments/production/main.tf
module "networking" {
  source = "../../modules/networking"
  
  vnet_name     = "vnet-production"
  address_space = ["10.0.0.0/16"]
  subnets = {
    aks    = "10.0.1.0/24"
    data   = "10.0.2.0/24"
  }
  location            = "eastus"
  resource_group_name = azurerm_resource_group.main.name
}

module "aks_ota" {
  source = "../../modules/aks"
  
  cluster_name        = "aks-ota-production"
  resource_group_name = azurerm_resource_group.main.name
  location            = "eastus"
  node_count          = 5
  vm_size             = "Standard_D4s_v3"
  subnet_id           = module.networking.subnet_ids["aks"]  # Module output!
  
  tags = {
    environment = "production"
    project     = "tvs-ota"
  }
}

module "aks_pki" {
  source = "../../modules/aks"   # Same module, different config!
  
  cluster_name        = "aks-pki-production"
  resource_group_name = azurerm_resource_group.main.name
  node_count          = 3
  vm_size             = "Standard_D2s_v3"
  subnet_id           = module.networking.subnet_ids["aks"]
  
  tags = {
    environment = "production"
    project     = "tvs-pki"
  }
}
```

---

## 📋 Module Sources | Module Sources

```hcl
# Local path
module "vpc" {
  source = "../../modules/networking"
}

# Git repository
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//networking?ref=v2.1.0"
}

# Terraform Registry (public)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# Private registry (Terraform Cloud/Enterprise)
module "aks" {
  source  = "app.terraform.io/my-org/aks/azurerm"
  version = "~> 3.0"
}
```

---

## 📋 Composition Patterns | Design Patterns

### Pattern 1: Flat Modules

```
environments/
├── dev/
│   └── main.tf      → uses modules/aks, modules/networking
├── staging/
│   └── main.tf      → uses same modules, different values
└── production/
    └── main.tf      → uses same modules, production values
```

### Pattern 2: Terragrunt (DRY wrapper)

```hcl
# terragrunt.hcl — DRY configuration
terraform {
  source = "../../modules/aks"
}

inputs = {
  cluster_name = "aks-${local.env}"
  node_count   = local.env == "prod" ? 5 : 2
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         MODULES CHEAT SHEET                      │
├──────────────────────────────────────────────────┤
│ MODULE STRUCTURE:                                │
│   variables.tf = inputs (what caller provides)   │
│   main.tf      = resources (what module creates) │
│   outputs.tf   = outputs (what caller receives)  │
│                                                  │
│ CALLING A MODULE:                                │
│   module "name" {                                │
│     source = "./modules/aks"                     │
│     input1 = value1                              │
│   }                                              │
│   Use: module.name.output_name                   │
│                                                  │
│ BEST PRACTICES:                                  │
│   ✓ One module = one logical component           │
│   ✓ Pin versions for registry modules            │
│   ✓ Use outputs to pass data between modules     │
│   ✓ Don't nest modules more than 2 levels deep   │
│   ✓ README with usage example in every module    │
│                                                  │
│ VERSIONING:                                      │
│   Git tags: git tag v1.0.0 → source ref=v1.0.0  │
│   Registry: version = "~> 3.0" (semver)         │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: When should you create a module vs inline resources?**
- Module when: same pattern used 2+ times, team sharing needed, want to enforce standards
- Inline when: one-off resource, simple setup, no reuse needed
- Rule: If you copy-paste terraform code → make it a module

**Q: How do you version modules for a team?**
- Git tags (v1.0.0, v2.0.0) with semantic versioning
- Breaking changes = major version bump
- Consumers pin version: `ref=v2.1.0`
- CI validates module changes with tests before tagging

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Module create முடியும் (variables + main + outputs)
- [ ] Module call முடியும் (source, inputs, use outputs)
- [ ] Module versioning strategy explain முடியும்
- [ ] Composition pattern choose முடியும்
