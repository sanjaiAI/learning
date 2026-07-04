# Module 13: Security & Compliance for Terraform

## 📖 Story: The Airport Security Checkpoint

**English:**
Think of Terraform security like airport security — you don't check passengers AFTER they're on the plane. You check them BEFORE boarding. That's "shift-left" security.

In a traditional workflow:
1. Developer writes Terraform → deploys → security team finds problems WEEKS later → expensive fix
2. **Shift-left approach:** Developer writes Terraform → automated scan catches issues in the PR → fix immediately → deploy safely

At EB (Embedded CI/CT project), we had 50+ developers writing Terraform. Without automated scanning, someone WILL accidentally:
- Create a storage bucket with public access
- Deploy a VM without disk encryption
- Forget mandatory cost-allocation tags
- Open SSH (port 22) to 0.0.0.0/0

Our solution: **Checkov runs in Jenkins pipeline as a mandatory gate.** No green CI = no merge. Period.

This is what separates a senior architect from a developer — you don't just write correct code, you **build systems that prevent incorrect code from ever reaching production.**

**தமிழ்:**
Terraform security-ஐ airport security போல நினைக்கவும் — passengers plane-ல் ஏறிய பிறகு check செய்வதில்லை. ஏறுவதற்கு முன்பே check செய்கிறோம். இதுதான் "shift-left" security.

Traditional workflow-ல்:
1. Developer Terraform எழுதுகிறார் → deploy → security team WEEKS-க்கு பிறகு problems கண்டுபிடிக்கிறது → expensive fix
2. **Shift-left approach:** Developer Terraform எழுதுகிறார் → automated scan PR-ல் issues catch → உடனே fix → safely deploy

EB project-ல், 50+ developers Terraform எழுதினார்கள். Automated scanning இல்லாமல், யாராவது accidentally:
- Public access-உடன் storage bucket create செய்வார்
- Disk encryption இல்லாமல் VM deploy செய்வார்
- Mandatory cost-allocation tags மறப்பார்
- SSH (port 22) 0.0.0.0/0-க்கு open செய்வார்

எங்கள் solution: **Checkov Jenkins pipeline-ல் mandatory gate-ஆக run ஆகும்.** Green CI இல்லை = merge இல்லை.

Senior architect-ஐ developer-இடமிருந்து வேறுபடுத்துவது இதுதான் — நீங்கள் correct code மட்டும் எழுதுவதில்லை, **incorrect code production-ஐ reach செய்வதை தடுக்கும் systems build செய்கிறீர்கள்.**

---

## 📊 Architecture & Concepts

### Shift-Left Security Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                   IaC SECURITY PIPELINE                               │
│                                                                       │
│  Developer      PR Created       CI Pipeline        Merge/Deploy     │
│     │               │                │                   │           │
│     ▼               ▼                ▼                   ▼           │
│  ┌──────┐     ┌──────────┐    ┌────────────┐     ┌───────────┐     │
│  │ Write │────▶│ PR Gate  │───▶│ Scan Tools │────▶│  Deploy   │     │
│  │  HCL  │     │          │    │            │     │           │     │
│  └──────┘     └──────────┘    │ • Checkov  │     │ terraform │     │
│                                │ • tfsec    │     │   apply   │     │
│                                │ • OPA      │     └───────────┘     │
│                                │ • Sentinel │                        │
│                                └────────────┘                        │
│                                      │                               │
│                               PASS ──┤── FAIL                        │
│                                │           │                         │
│                           Allow PR    Block PR                       │
│                                      + Comment                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Security Scanning Tools Comparison

| Tool | Type | Speed | Coverage | Cost | Best For |
|------|------|-------|----------|------|----------|
| **Checkov** | Static analysis | Fast | Broad (1000+ rules) | Free/OSS | General scanning |
| **tfsec** | Static analysis | Very fast | Terraform-focused | Free/OSS | Quick PR checks |
| **OPA/Rego** | Policy engine | Medium | Custom policies | Free/OSS | Custom org rules |
| **Sentinel** | Policy-as-code | Fast | HCP Terraform | Paid (HCP) | Enterprise governance |
| **Trivy** | Multi-scanner | Fast | IaC + containers | Free/OSS | All-in-one scanning |

### What Each Tool Catches

```
┌────────────────────────────────────────────────────────────────┐
│                     SCANNING LAYERS                              │
│                                                                  │
│  Layer 1: STATIC ANALYSIS (Checkov/tfsec)                       │
│  ├── Public access on storage/databases                         │
│  ├── Missing encryption (at-rest, in-transit)                   │
│  ├── Overly permissive IAM                                      │
│  ├── Missing logging/monitoring                                 │
│  ├── Network security (open ports, missing firewalls)           │
│  └── Missing tags/labels                                        │
│                                                                  │
│  Layer 2: POLICY ENGINE (OPA/Sentinel)                          │
│  ├── Custom organizational rules                                │
│  ├── Naming conventions                                         │
│  ├── Allowed instance types/regions                             │
│  ├── Cost controls (max instance size)                          │
│  └── Compliance frameworks (SOC2, HIPAA, PCI)                   │
│                                                                  │
│  Layer 3: PLAN ANALYSIS (OPA on terraform plan)                 │
│  ├── Destructive changes (prevent accidental deletes)           │
│  ├── Resource count limits                                      │
│  └── Cross-resource validation                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Byheart for Interview

### The Security Triad for IaC

```
1. PREVENT  → Scan in CI (Checkov/tfsec) before deploy
2. DETECT   → Drift detection (scheduled terraform plan)
3. RESPOND  → Auto-remediation or alert + manual fix

Key Phrase: "We shift security left — scan at PR time,
            enforce via CI gates, detect drift continuously"
```

### Top 10 Findings (memorize these!)

| # | Finding | Risk | Fix |
|---|---------|------|-----|
| 1 | S3/GCS bucket public | Data breach | `public_access_prevention = "enforced"` |
| 2 | No encryption at rest | Compliance fail | `encryption_key_name = ...` |
| 3 | SSH open to 0.0.0.0/0 | Unauthorized access | Restrict to bastion CIDR |
| 4 | No VPC flow logs | No audit trail | Enable flow logs |
| 5 | Missing tags | Cost allocation fail | Enforce via policy |
| 6 | Overly permissive IAM | Privilege escalation | Least-privilege roles |
| 7 | No deletion protection | Accidental data loss | `deletion_protection = true` |
| 8 | Default network used | Shared attack surface | Create custom VPC |
| 9 | No backup configured | Data loss risk | Enable automated backups |
| 10 | HTTP instead of HTTPS | Data interception | Force SSL/TLS |

### Interview Phrases (Tamil Memory Aid)

```
"Shift-left" = கதவிலேயே தடுப்பது (stopping at the door itself)
"Policy-as-code" = விதிகளை code-ல் எழுதுவது (writing rules as code)
"Guardrails" = guard rails — தடுப்பு வேலி (protective fence)
"Blast radius" = damage range — சேதம் பரவும் அளவு
"Compliance drift" = விதிமீறல் நகர்வு (rule violation movement)
```

---

## ⚡ Quick Hands-on

### Lab Setup

```bash
# SSH to lab server
ssh root@203.57.85.108

# Create lab directory
mkdir -p ~/tf-lab/security && cd ~/tf-lab/security

# Install Checkov
pip3 install checkov 2>/dev/null || pip install checkov

# Install tfsec
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash 2>/dev/null

# Verify installations
checkov --version
tfsec --version
```

### Exercise 1: Create Intentionally Insecure Terraform

```bash
cat > main.tf << 'EOF'
# INTENTIONALLY INSECURE — for scanning demo
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

# BAD: Public storage bucket
resource "google_storage_bucket" "data" {
  name     = "my-insecure-bucket-demo"
  location = "US"
  
  # MISSING: public_access_prevention
  # MISSING: uniform_bucket_level_access
  # MISSING: versioning
  # MISSING: encryption
}

# BAD: Firewall open to world
resource "google_compute_firewall" "allow_ssh" {
  name    = "allow-ssh-world"
  network = "default"  # BAD: using default network

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]  # BAD: open to entire internet
}

# BAD: VM without encryption, no service account restriction
resource "google_compute_instance" "web" {
  name         = "insecure-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      # MISSING: disk encryption key
    }
  }

  network_interface {
    network = "default"
    access_config {}  # BAD: public IP without restriction
  }

  # MISSING: service_account block
  # MISSING: shielded_instance_config
  # MISSING: labels/tags
}

# BAD: SQL instance without SSL, no private IP
resource "google_sql_database_instance" "db" {
  name             = "insecure-db"
  database_version = "POSTGRES_14"
  region           = "us-central1"

  settings {
    tier = "db-f1-micro"
    # MISSING: backup_configuration
    # MISSING: ip_configuration with private_network
    # MISSING: database_flags for SSL
  }

  # MISSING: deletion_protection = true
}
EOF
```

### Exercise 2: Run Checkov Scan

```bash
# Run Checkov against our insecure code
checkov -d . --output cli

# Expected output: MANY failures!
# You'll see findings like:
#   CKV_GCP_62: "Ensure Cloud storage has versioning enabled"
#   CKV_GCP_28: "Ensure Firewall rule does not allow SSH from 0.0.0.0/0"
#   CKV_GCP_32: "Ensure VM disk encryption"

# Generate JSON report
checkov -d . --output json > checkov_report.json

# Check specific framework only
checkov -d . --framework terraform --check CKV_GCP_28,CKV_GCP_62
```

### Exercise 3: Run tfsec Scan

```bash
# Run tfsec
tfsec .

# tfsec output is more concise, color-coded
# Shows: severity, description, resource, and fix suggestion

# Generate SARIF output (for CI integration)
tfsec . --format sarif > tfsec_results.sarif

# Exclude specific checks if needed (document WHY)
tfsec . --exclude google-compute-no-public-ingress
```

### Exercise 4: Fix the Violations

```bash
cat > main_secure.tf << 'EOF'
# SECURE VERSION — all findings fixed
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

variable "project_id" {
  type = string
}

variable "kms_key" {
  type        = string
  description = "KMS key for encryption"
}

# GOOD: Secure storage bucket
resource "google_storage_bucket" "data" {
  name     = "${var.project_id}-secure-data"
  location = "US"
  project  = var.project_id

  uniform_bucket_level_access = true        # FIXED
  public_access_prevention    = "enforced"  # FIXED

  versioning {                              # FIXED
    enabled = true
  }

  encryption {                              # FIXED
    default_kms_key_name = var.kms_key
  }

  labels = {                                # FIXED
    environment = "production"
    team        = "platform"
    cost-center = "engineering"
  }
}

# GOOD: Restricted firewall
resource "google_compute_firewall" "allow_ssh" {
  name    = "allow-ssh-bastion"
  network = google_compute_network.main.name  # FIXED: custom network

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["10.0.0.0/24"]  # FIXED: bastion subnet only
  target_tags   = ["ssh-allowed"]   # FIXED: only tagged instances
}

# Custom VPC instead of default
resource "google_compute_network" "main" {
  name                    = "secure-vpc"
  auto_create_subnetworks = false
  project                 = var.project_id
}

# GOOD: Secure VM
resource "google_compute_instance" "web" {
  name         = "secure-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
  project      = var.project_id

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
    kms_key_self_link = var.kms_key  # FIXED
  }

  network_interface {
    network    = google_compute_network.main.name
    subnetwork = "secure-subnet"
    # NO access_config = no public IP  # FIXED
  }

  service_account {                    # FIXED
    email  = "minimal-sa@${var.project_id}.iam.gserviceaccount.com"
    scopes = ["cloud-platform"]
  }

  shielded_instance_config {           # FIXED
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }

  labels = {                           # FIXED
    environment = "production"
    team        = "platform"
  }
}

# GOOD: Secure SQL instance
resource "google_sql_database_instance" "db" {
  name             = "secure-db"
  database_version = "POSTGRES_14"
  region           = "us-central1"
  project          = var.project_id

  deletion_protection = true            # FIXED

  settings {
    tier = "db-custom-2-4096"

    backup_configuration {              # FIXED
      enabled                        = true
      point_in_time_recovery_enabled = true
      backup_retention_settings {
        retained_backups = 30
      }
    }

    ip_configuration {                  # FIXED
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
      require_ssl     = true
    }

    database_flags {                    # FIXED
      name  = "log_checkpoints"
      value = "on"
    }
  }
}
EOF

# Re-scan the secure version
checkov -f main_secure.tf --output cli
# Expected: All checks PASSED!
```

### Exercise 5: OPA Policy (Custom Rules)

```bash
mkdir -p policies

# Create OPA policy: enforce mandatory tags
cat > policies/mandatory_tags.rego << 'EOF'
package terraform.mandatory_tags

# Rule: All resources must have 'environment' and 'team' labels
deny[msg] {
  resource := input.resource_changes[_]
  resource.change.after.labels == null
  msg := sprintf("Resource '%s' is missing labels", [resource.address])
}

deny[msg] {
  resource := input.resource_changes[_]
  labels := resource.change.after.labels
  not labels.environment
  msg := sprintf("Resource '%s' missing 'environment' label", [resource.address])
}

deny[msg] {
  resource := input.resource_changes[_]
  labels := resource.change.after.labels
  not labels.team
  msg := sprintf("Resource '%s' missing 'team' label", [resource.address])
}
EOF

# Create OPA policy: restrict instance types
cat > policies/instance_restrictions.rego << 'EOF'
package terraform.instance_restrictions

# Rule: No instances larger than n2-standard-8 in non-prod
allowed_types := {
  "e2-micro", "e2-small", "e2-medium",
  "e2-standard-2", "e2-standard-4",
  "n2-standard-2", "n2-standard-4", "n2-standard-8"
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "google_compute_instance"
  machine_type := resource.change.after.machine_type
  not allowed_types[machine_type]
  msg := sprintf(
    "Instance '%s' uses disallowed type '%s'. Max: n2-standard-8",
    [resource.address, machine_type]
  )
}
EOF

# Create OPA policy: no public IPs
cat > policies/no_public_ip.rego << 'EOF'
package terraform.no_public_ip

# Rule: Compute instances must not have public IPs
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "google_compute_instance"
  resource.change.after.network_interface[_].access_config
  msg := sprintf(
    "Instance '%s' has public IP (access_config). Use internal IPs + IAP.",
    [resource.address]
  )
}
EOF

echo "OPA policies created. In CI, you'd run:"
echo "  terraform plan -out=plan.tfplan"
echo "  terraform show -json plan.tfplan > plan.json"
echo "  opa eval --data policies/ --input plan.json 'data.terraform.mandatory_tags.deny'"
```

---

## 🔥 Scenario Challenge

### Scenario: "Security Audit Found 47 Violations Across 12 Terraform Repos"

**Situation:** You're the platform architect. Security audit ran Checkov across all repos and found 47 violations. CTO wants zero violations in 2 weeks.

**Your approach (think like an architect):**

```
Week 1: TRIAGE + QUICK WINS
├── Categorize by severity (Critical/High/Medium/Low)
├── Critical (public access, no encryption): Fix immediately
├── High (missing logging, overly permissive IAM): Fix in 3 days
├── Medium (missing tags, default networks): Fix in 5 days
└── Low (naming conventions): Track, fix in sprint

Week 2: PREVENT RECURRENCE
├── Add Checkov to ALL CI pipelines (mandatory gate)
├── Create .checkov.yaml baseline (suppress known false positives)
├── Write custom OPA policies for org-specific rules
├── Create secure Terraform module library (pre-approved patterns)
└── Train developers (30-min session: "Top 10 Terraform security mistakes")
```

**The architect answer includes GOVERNANCE:**
```
Ongoing Process:
├── Weekly: Review new Checkov rules in updates
├── Monthly: Compliance dashboard review
├── Quarterly: Policy review with security team
└── Annually: Full re-assessment
```

---

## 🏗️ Real Project Reference: EB CI/CT Pipeline

### How We Enforced Security at EB

```
┌────────────────────────────────────────────────────────────────┐
│              EB PROJECT: Security Pipeline                       │
│                                                                  │
│  Jenkins Pipeline Stages:                                       │
│                                                                  │
│  1. checkout ─────────────────────────────────────────────┐     │
│  2. terraform init                                        │     │
│  3. terraform validate                                    │     │
│  4. ┌─────────────────────────────────────────────┐      │     │
│     │ SECURITY GATE (parallel)                     │      │     │
│     │  ├── checkov -d . --soft-fail-on LOW        │      │     │
│     │  ├── tfsec . --minimum-severity HIGH        │      │     │
│     │  └── custom OPA policies                     │      │     │
│     └─────────────────────────────────────────────┘      │     │
│     │                                                     │     │
│     ├── ALL PASS → continue                              │     │
│     └── ANY FAIL → block + notify + PR comment           │     │
│                                                           │     │
│  5. terraform plan                                        │     │
│  6. Manual approval (for prod)                           │     │
│  7. terraform apply                                       │     │
└────────────────────────────────────────────────────────────────┘

Checkov Config (.checkov.yaml):
  soft-fail-on: [LOW]          # Don't block on low severity
  skip-check:                   # Documented exceptions
    - CKV_GCP_25              # Reason: using org policy instead
  framework: terraform
```

### Jenkins Pipeline Snippet (EB Pattern)

```groovy
// Jenkinsfile — Security scanning stage
stage('Security Scan') {
    parallel {
        stage('Checkov') {
            steps {
                sh '''
                    checkov -d . \
                        --config-file .checkov.yaml \
                        --output cli \
                        --output junitxml \
                        --output-file-path console,results/
                '''
            }
            post {
                always {
                    junit 'results/results_junitxml.xml'
                }
            }
        }
        stage('tfsec') {
            steps {
                sh 'tfsec . --format sarif --out tfsec.sarif --minimum-severity HIGH'
            }
        }
        stage('OPA Policy') {
            steps {
                sh '''
                    terraform plan -out=plan.tfplan
                    terraform show -json plan.tfplan > plan.json
                    opa eval \
                        --data policies/ \
                        --input plan.json \
                        --fail-defined \
                        'data.terraform.mandatory_tags.deny[x]'
                '''
            }
        }
    }
}
```

### Suppression File (.checkov.yaml)

```yaml
# .checkov.yaml — EB project configuration
framework:
  - terraform
output:
  - cli
  - junitxml
directory:
  - .
soft-fail-on:
  - LOW
skip-check:
  # Documented exceptions with JIRA tickets
  - CKV_GCP_25  # JIRA-1234: Org policy handles this
  - CKV_GCP_78  # JIRA-1235: False positive for our pattern
check:
  # Run only these high-value checks for fast feedback
  - CKV_GCP_28  # No SSH from 0.0.0.0/0
  - CKV_GCP_62  # Storage versioning
  - CKV_GCP_32  # VM disk encryption
```

---

## 🎤 Interview Q&A

### Q1: "How do you enforce security in Terraform at scale?"

**English Answer:**
"We use a three-layer approach:

1. **Prevention (PR time):** Checkov and tfsec run as mandatory CI gates. No merge if HIGH/CRITICAL violations exist. This catches 90% of issues before they reach any environment.

2. **Custom policies (OPA/Rego):** Organization-specific rules beyond what Checkov covers — naming conventions, allowed regions, mandatory tags, instance type restrictions. These encode our specific security posture.

3. **Detection (runtime):** Scheduled `terraform plan` detects drift — if someone makes a manual change that violates policy, we catch it within hours.

At EB with 50+ developers, this reduced security findings from 47 to near-zero in production within 3 weeks. The key insight: make the secure path the easy path — provide pre-approved modules that are already compliant."

**தமிழ் Answer:**
"நாங்கள் three-layer approach பயன்படுத்துகிறோம்:

1. **Prevention (PR time):** Checkov, tfsec mandatory CI gates-ஆக run ஆகும். HIGH/CRITICAL violations இருந்தால் merge ஆகாது. இது 90% issues-ஐ environment reach செய்வதற்கு முன்பே catch செய்கிறது.

2. **Custom policies (OPA/Rego):** Organization-specific rules — naming conventions, allowed regions, mandatory tags. இவை எங்கள் specific security posture-ஐ encode செய்கின்றன.

3. **Detection (runtime):** Scheduled `terraform plan` drift detect செய்கிறது — யாராவது manual change செய்தால், hours-க்குள் catch செய்கிறோம்.

EB-ல் 50+ developers-உடன், 3 weeks-ல் security findings 47-லிருந்து near-zero-க்கு reduce ஆனது. Key insight: secure path-ஐ easy path-ஆக மாற்றுங்கள் — pre-approved modules provide செய்யுங்கள்."

---

### Q2: "Checkov found 200 violations in an existing project. What's your approach?"

**English Answer:**
"I wouldn't try to fix all 200 at once — that's a recipe for outages. My approach:

1. **Baseline:** Create `.checkov.yaml` with current state as baseline (skip existing violations)
2. **Prevent new:** CI gate prevents ANY new violations immediately
3. **Triage:** Categorize existing 200 by severity — Critical (fix today), High (this week), Medium (this sprint), Low (next sprint)
4. **Incremental fix:** One PR per resource group, each with proper testing
5. **Remove from baseline:** As fixes merge, remove skip entries

This approach means: no new debt from day 1, existing debt reduced systematically. At EB, we cleared 47 violations in 2 weeks using this exact pattern."

**தமிழ் Answer:**
"200 violations-ஐ ஒரே நேரத்தில் fix செய்ய முயற்சிக்க மாட்டேன் — அது outages-க்கு வழிவகுக்கும். என் approach:

1. **Baseline:** `.checkov.yaml`-ல் current state-ஐ baseline-ஆக create (existing violations skip)
2. **Prevent new:** CI gate உடனடியாக புதிய violations-ஐ prevent
3. **Triage:** 200-ஐ severity-ப்படி categorize — Critical (இன்றே fix), High (இந்த வாரம்), Medium (இந்த sprint), Low (அடுத்த sprint)
4. **Incremental fix:** ஒவ்வொரு resource group-க்கும் ஒரு PR, proper testing-உடன்
5. **Remove from baseline:** Fixes merge ஆகும்போது, skip entries-ஐ remove

Day 1-லிருந்து புதிய debt இல்லை, existing debt systematically reduce ஆகும்."

---

### Q3: "OPA vs Sentinel vs Checkov — when do you use which?"

**English Answer:**
"Each tool has a different sweet spot:

- **Checkov/tfsec:** First line of defense. Pre-built rules for known misconfigurations. Use for: standard security checks that apply to everyone. Fast, no custom logic needed.

- **OPA/Rego:** Custom organizational policies. Use for: company-specific rules that no pre-built tool covers — 'only these regions', 'must have these tags', 'max instance size is X'. Runs against `terraform plan` JSON.

- **Sentinel:** HashiCorp ecosystem native. Use for: HCP Terraform/Enterprise customers who want policy integrated into the Terraform workflow itself. Advantage: runs INSIDE the Terraform run, not as external CI step.

At EB, we used Checkov (standard checks) + OPA (custom rules). We didn't use Sentinel because we weren't on HCP Terraform — we ran open-source Terraform in Jenkins."

**தமிழ் Answer:**
"ஒவ்வொரு tool-க்கும் வெவ்வேறு sweet spot உள்ளது:

- **Checkov/tfsec:** First line of defense. Known misconfigurations-க்கு pre-built rules. Standard security checks-க்கு use.

- **OPA/Rego:** Custom organizational policies. Company-specific rules-க்கு — 'இந்த regions மட்டும்', 'இந்த tags கட்டாயம்', 'max instance size X'. `terraform plan` JSON-க்கு எதிராக run ஆகும்.

- **Sentinel:** HashiCorp ecosystem native. HCP Terraform/Enterprise customers-க்கு. Advantage: Terraform run INSIDE run ஆகும்.

EB-ல், Checkov (standard checks) + OPA (custom rules) use செய்தோம். HCP Terraform-ல் இல்லாததால் Sentinel use செய்யவில்லை."

---

### Q4: "How do you handle false positives in security scanning?"

**English Answer:**
"False positives are inevitable and must be managed, not ignored:

1. **Document every suppression** with a reason and ticket number
2. **Use `.checkov.yaml` or inline comments** (`#checkov:skip=CKV_GCP_25:Reason here`)
3. **Regular review** — quarterly, review all suppressions. Are they still valid?
4. **Alternative controls** — if you skip a check, document what alternative control exists

The worst thing is developers disabling the entire scanner because of false positives. Better to suppress specific findings with documentation than to lose all scanning."

**தமிழ் Answer:**
"False positives unavoidable — manage செய்ய வேண்டும், ignore செய்யக்கூடாது:

1. **ஒவ்வொரு suppression-ஐயும் document** — reason + ticket number
2. **`.checkov.yaml` அல்லது inline comments** use
3. **Regular review** — quarterly, எல்லா suppressions-ஐயும் review. இன்னும் valid-ஆ?
4. **Alternative controls** — check skip செய்தால், alternative control என்ன என்று document

மோசமான விஷயம்: false positives-ஆல் developers scanner-ஐயே disable செய்வது. Specific findings-ஐ documentation-உடன் suppress செய்வது better."

---

## ✅ Self-Check

### Knowledge Verification

- [ ] Can you explain shift-left security in one sentence?
- [ ] Can you name 3 tools and when to use each?
- [ ] Can you write a basic OPA policy from memory?
- [ ] Do you know the top 5 Terraform security findings?
- [ ] Can you explain how to handle 200 existing violations?
- [ ] Can you describe the CI pipeline integration pattern?

### Interview Readiness

- [ ] Can you draw the security pipeline on a whiteboard?
- [ ] Can you explain Checkov vs OPA vs Sentinel trade-offs?
- [ ] Can you describe the EB project security implementation?
- [ ] Can you talk about false positive management strategy?

### Command Competence

```bash
# Can you run these from memory?
checkov -d . --output cli
checkov -d . --config-file .checkov.yaml
tfsec . --minimum-severity HIGH
opa eval --data policies/ --input plan.json 'data.terraform[policy].deny'
terraform plan -out=plan.tfplan && terraform show -json plan.tfplan > plan.json
```

---

## 📋 Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│           TERRAFORM SECURITY — CHEAT SHEET                      │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TOOLS:                                                         │
│    checkov -d .                    # Scan current directory     │
│    tfsec .                         # Fast Terraform scan        │
│    opa eval --input plan.json      # Custom policy check        │
│                                                                  │
│  CI PATTERN:                                                    │
│    PR → checkov + tfsec → PASS → plan → approve → apply        │
│         (fail = block merge)                                    │
│                                                                  │
│  SUPPRESS (with documentation!):                                │
│    # Inline: #checkov:skip=CKV_GCP_25:org policy handles       │
│    # File:   .checkov.yaml skip-check list                     │
│                                                                  │
│  TOP FIXES:                                                     │
│    public_access_prevention = "enforced"                        │
│    deletion_protection      = true                              │
│    encryption              = { default_kms_key_name = key }     │
│    source_ranges           = ["10.x.x.x/24"] (not 0.0.0.0/0)  │
│                                                                  │
│  ARCHITECT MINDSET:                                             │
│    "Make secure path = easy path"                               │
│    "Provide pre-approved modules"                               │
│    "Gate in CI, detect drift continuously"                      │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```
