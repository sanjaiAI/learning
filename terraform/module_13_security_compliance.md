# Module 13: Security & Compliance
# மாடுல் 13: Terraform Security & Compliance

---

## 🎯 What? | என்ன?

**English:** Shift-left security for Terraform — scan IaC code for misconfigurations BEFORE deployment using Checkov, tfsec, and policy-as-code (OPA/Sentinel).

**தமிழ்:** Terraform code-ஐ deploy செய்வதற்கு முன் security scan — Checkov, tfsec, OPA/Sentinel.

---

## 🛠️ Checkov (Most popular IaC scanner)

```bash
# Install
pip install checkov

# Scan Terraform directory
checkov -d . --framework terraform

# Scan plan file (catches runtime values!)
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
checkov -f plan.json --framework terraform_plan

# Skip specific checks (with justification!)
checkov -d . --skip-check CKV_AZURE_35,CKV_AZURE_36

# Custom policy (Python)
# checkov/policies/custom_check.py
```

### Common Checkov Findings

| Check ID | Issue | Fix |
|----------|-------|-----|
| CKV_AZURE_35 | Storage account allows public access | `public_network_access_enabled = false` |
| CKV_AZURE_23 | Key Vault not recoverable | `purge_protection_enabled = true` |
| CKV_AZURE_7 | AKS without network policy | `network_policy = "calico"` |
| CKV_GCP_12 | GKE basic auth enabled | Remove basic auth config |
| CKV_GCP_18 | Cloud SQL public IP | `ipv4_enabled = false` |

---

## 🛠️ tfsec (Terraform-specific scanner)

```bash
# Install
brew install tfsec   # or go install github.com/aquasecurity/tfsec/cmd/tfsec@latest

# Scan
tfsec .
tfsec . --format json --out results.json

# Inline suppression (in .tf file)
resource "azurerm_storage_account" "legacy" {
  #tfsec:ignore:azure-storage-no-public-access
  public_network_access_enabled = true    # Legacy app requires this
}
```

---

## 🛠️ CI/CD Integration

```yaml
# .github/workflows/terraform-security.yml
name: Terraform Security
on: [pull_request]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Checkov Scan
      uses: bridgecrewio/checkov-action@v12
      with:
        directory: terraform/
        framework: terraform
        output_format: sarif
        soft_fail: false    # Block PR on findings!
    
    - name: tfsec
      uses: aquasecurity/tfsec-action@v1.0.0
      with:
        working_directory: terraform/
        
    - name: Terraform Plan (for plan scanning)
      run: |
        cd terraform/
        terraform init
        terraform plan -out=plan.tfplan
        terraform show -json plan.tfplan > plan.json
        checkov -f plan.json --framework terraform_plan
```

---

## 🛠️ OPA (Open Policy Agent) for Terraform

```rego
# policy/require_tags.rego
package terraform

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "azurerm_resource_group"
  not resource.change.after.tags.environment
  msg := sprintf("Resource group '%s' must have 'environment' tag", [resource.name])
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "azurerm_kubernetes_cluster"
  not resource.change.after.network_profile[0].network_policy
  msg := "AKS cluster must have network_policy enabled"
}
```

```bash
# Evaluate
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
opa eval --data policy/ --input plan.json "data.terraform.deny"
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│      TERRAFORM SECURITY CHEAT SHEET              │
├──────────────────────────────────────────────────┤
│ TOOLS:                                           │
│   Checkov    = comprehensive, multi-framework    │
│   tfsec      = Terraform-focused, fast           │
│   OPA/Rego   = custom policies                   │
│   Sentinel   = Terraform Cloud/Enterprise        │
│                                                  │
│ SCAN STRATEGY:                                   │
│   1. Pre-commit: tfsec (fast, local)             │
│   2. PR gate: Checkov (comprehensive)            │
│   3. Plan scan: Checkov on plan.json (runtime!)  │
│   4. Drift detection: periodic plan in CI        │
│                                                  │
│ MUST-HAVE RULES:                                 │
│   ✓ No public IPs on databases                   │
│   ✓ Encryption at rest enabled                   │
│   ✓ Network policies on K8s clusters             │
│   ✓ Required tags (environment, team, project)   │
│   ✓ Private endpoints for PaaS services          │
│   ✓ Key Vault purge protection                   │
│   ✓ Storage account: no public access            │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: How do you enforce Terraform security in your pipeline?**
1. Pre-commit hooks: `tfsec` + `terraform fmt` + `terraform validate`
2. PR gate: Checkov scan blocks merge on critical findings
3. Plan scan: `checkov -f plan.json` catches runtime misconfigs
4. OPA policies: custom org rules (required tags, naming conventions)
5. Drift detection: weekly `terraform plan` in CI to catch manual changes

**Q: Static scan vs plan scan — difference?**
- Static: scans `.tf` files directly. Fast but misses dynamic values (variables, data sources)
- Plan scan: scans `plan.json` with actual resolved values. Catches real misconfigs but needs cloud access

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Checkov scan run முடியும்
- [ ] CI/CD-ல் security gate implement முடியும்
- [ ] OPA policy write முடியும்
- [ ] Common findings fix முடியும்
