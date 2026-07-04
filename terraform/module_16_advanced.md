# Module 16: Advanced Patterns
# மாடுல் 16: Advanced Terraform Patterns

---

## 🎯 What? | என்ன?

**English:** Advanced Terraform operations — import existing resources, moved blocks for refactoring, drift detection, Crossplane comparison, and large-scale patterns.

**தமிழ்:** Advanced operations — existing resources import, refactoring, drift detection, large-scale patterns.

---

## 🛠️ Import (Adopt existing resources)

```hcl
# Terraform 1.5+ — declarative import blocks (preferred!)
import {
  to = azurerm_resource_group.legacy
  id = "/subscriptions/xxx/resourceGroups/rg-legacy-app"
}

import {
  to = azurerm_kubernetes_cluster.existing
  id = "/subscriptions/xxx/resourceGroups/rg-prod/providers/Microsoft.ContainerService/managedClusters/aks-existing"
}

# Generate config from imported resource:
terraform plan -generate-config-out=generated.tf
# Review generated.tf → clean up → commit
```

---

## 🛠️ Moved Blocks (Refactoring)

```hcl
# Renamed a resource? Tell Terraform (avoid destroy+recreate!)
moved {
  from = azurerm_resource_group.main
  to   = azurerm_resource_group.platform
}

# Moved into a module?
moved {
  from = azurerm_virtual_network.vnet
  to   = module.networking.azurerm_virtual_network.this
}

# Moved from count to for_each?
moved {
  from = azurerm_subnet.subnets[0]
  to   = azurerm_subnet.subnets["aks"]
}
```

---

## 🛠️ Drift Detection & Remediation

```bash
# Detect drift (plan in CI, no apply)
terraform plan -detailed-exitcode
# Exit 0 = no changes
# Exit 1 = error
# Exit 2 = changes detected (DRIFT!)

# Refresh state (update state to match reality, don't change infra)
terraform apply -refresh-only

# Full drift detection script
#!/bin/bash
terraform init -backend-config=backend.hcl
terraform plan -detailed-exitcode -out=drift.tfplan 2>&1
EXIT_CODE=$?
if [ $EXIT_CODE -eq 2 ]; then
  echo "DRIFT DETECTED!"
  terraform show drift.tfplan
  # Send alert to Slack/Teams
fi
```

---

## 🛠️ State Splitting (Monolith → Components)

```bash
# Problem: one giant state file with 200+ resources (risky, slow)
# Solution: split into logical components

# 1. Extract networking resources to separate state
terraform state mv -state-out=networking.tfstate \
  azurerm_virtual_network.main \
  azurerm_subnet.aks \
  azurerm_subnet.data

# 2. Move to new backend
# In new networking/ directory, configure backend and import

# Better: use terraform_remote_state data source to reference across states
data "terraform_remote_state" "networking" {
  backend = "azurerm"
  config = {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstate"
    container_name       = "tfstate"
    key                  = "networking/terraform.tfstate"
  }
}

# Use output from networking state
resource "azurerm_kubernetes_cluster" "aks" {
  # ...
  default_node_pool {
    vnet_subnet_id = data.terraform_remote_state.networking.outputs.subnet_ids["aks"]
  }
}
```

---

## 🛠️ Crossplane vs Terraform

| Feature | Terraform | Crossplane |
|---------|-----------|------------|
| Language | HCL | YAML (K8s CRDs) |
| State | External (Blob/GCS) | In-cluster (etcd) |
| Reconciliation | Manual (apply) | Continuous (controller loop) |
| Drift handling | Detect (plan) | Auto-fix! |
| Best for | Infra provisioning | Platform self-service |
| Learning curve | HCL + providers | K8s + CRDs |

```yaml
# Crossplane: same Cloud SQL, but as K8s YAML
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-db
spec:
  forProvider:
    databaseVersion: POSTGRES_14
    region: us-central1
    settings:
    - tier: db-f1-micro
```

> **When Terraform, when Crossplane?**
> - Terraform: initial infrastructure provisioning (networks, clusters)
> - Crossplane: developer self-service (databases, buckets on-demand via K8s)
> - Both can coexist!

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│      ADVANCED TERRAFORM CHEAT SHEET              │
├──────────────────────────────────────────────────┤
│ IMPORT:                                          │
│   import block (1.5+) > terraform import CLI     │
│   -generate-config-out to bootstrap code         │
│                                                  │
│ REFACTORING:                                     │
│   moved block > terraform state mv               │
│   Code-based = reviewable, documented            │
│                                                  │
│ DRIFT:                                           │
│   plan -detailed-exitcode (exit 2 = drift)       │
│   apply -refresh-only (update state, no changes) │
│   Weekly CI job for detection                    │
│                                                  │
│ LARGE-SCALE:                                     │
│   Split state by component (network, compute)    │
│   terraform_remote_state for cross-references    │
│   Parallel applies (independent components)      │
│                                                  │
│ LIFECYCLE:                                       │
│   prevent_destroy = true (critical resources)    │
│   create_before_destroy (zero-downtime replace)  │
│   ignore_changes (managed outside Terraform)     │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: You have 200 manually-created Azure resources. How to bring under Terraform?**
1. Inventory: list all resources (`az resource list`)
2. Prioritize: start with networking, then compute, then data
3. Write Terraform code for each resource
4. Import: use `import` blocks + `-generate-config-out`
5. Plan: verify plan shows no changes (state matches reality)
6. Iterate: module-by-module, not all at once

**Q: How to refactor Terraform without destroying resources?**
- `moved` blocks: rename resources, move to modules
- `terraform state mv`: manual state surgery (last resort)
- Plan after refactor: must show NO changes (same resources, new addresses)
- Always backup state before refactoring

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Import existing resource முடியும்
- [ ] Moved blocks for refactoring use முடியும்
- [ ] Drift detection implement முடியும்
- [ ] State splitting execute முடியும்
- [ ] Terraform vs Crossplane tradeoff explain முடியும்
