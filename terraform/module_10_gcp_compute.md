# Module 10: GCP Compute with Terraform

## 📖 Story: The Factory Assembly Line

**English:**
Think of GCP Compute like running a factory. A **Compute Engine instance** is one machine on the assembly line. An **Instance Template** is the blueprint for that machine — specs, OS, startup script. A **Managed Instance Group (MIG)** is the assembly line itself — it creates identical machines from the template, replaces broken ones, and scales up/down based on demand. The **Autoscaler** is the factory manager watching the workload meters and adding/removing machines.

In EB's CI pipeline, we used MIGs with **preemptible VMs** (now called Spot VMs). These are 60-91% cheaper but can be reclaimed with 30-second notice. For CI builds that take 10-20 minutes, this is perfect — if a VM dies, the MIG recreates it and the build retries. We saved ~70% on compute costs.

**தமிழ்:**
GCP Compute-ஐ ஒரு factory நடத்துவது போல நினைக்கவும். **Compute Engine instance** assembly line-ல் ஒரு machine. **Instance Template** அந்த machine-ன் blueprint — specs, OS, startup script. **Managed Instance Group (MIG)** assembly line — template-லிருந்து identical machines உருவாக்கும், broken ones-ஐ replace செய்யும், demand-க்கு ஏற்ப scale up/down செய்யும். **Autoscaler** workload meters பார்த்து machines add/remove செய்யும் factory manager.

EB CI pipeline-ல், **preemptible VMs** (இப்போது Spot VMs) உடன் MIGs பயன்படுத்தினோம். இவை 60-91% cheaper ஆனால் 30-second notice-ல் reclaim ஆகலாம். 10-20 நிமிட CI builds-க்கு இது perfect — VM die ஆனால், MIG recreate செய்யும், build retry ஆகும். Compute costs-ல் ~70% save செய்தோம்.

---

## 📊 Architecture & Concepts

### GCP Compute Hierarchy

```
┌────────────────────────────────────────────────────────────┐
│                    AUTOSCALER                                │
│         (watches metrics, adjusts MIG size)                 │
└────────────────────────┬───────────────────────────────────┘
                         │ controls
┌────────────────────────▼───────────────────────────────────┐
│              MANAGED INSTANCE GROUP (MIG)                    │
│         (manages lifecycle of identical VMs)                 │
│                                                             │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐       │
│  │ VM-1 │  │ VM-2 │  │ VM-3 │  │ VM-4 │  │ VM-5 │       │
│  │(spot)│  │(spot)│  │(spot)│  │(spot)│  │(spot)│       │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘       │
└────────────────────────┬───────────────────────────────────┘
                         │ created from
┌────────────────────────▼───────────────────────────────────┐
│              INSTANCE TEMPLATE                               │
│  machine_type: e2-standard-4                                │
│  disk: 100GB SSD                                            │
│  image: ubuntu-2204-lts                                     │
│  startup_script: install-ci-tools.sh                        │
│  tags: ["ci-agent", "allow-ssh"]                           │
│  scheduling: preemptible = true                             │
└────────────────────────────────────────────────────────────┘
```

### Azure VMSS vs GCP MIG — Key Comparison

| Feature | Azure VMSS | GCP MIG |
|---------|-----------|---------|
| **Concept** | Virtual Machine Scale Set | Managed Instance Group |
| **Template** | VMSS model (inline) | Instance Template (separate resource) |
| **Template update** | Rolling upgrade policy | Rolling update policy / Canary |
| **Scope** | Regional (zone-spread) | Zonal OR Regional |
| **Autoscaling** | VMSS Autoscale rules | Autoscaler (separate resource) |
| **Health check** | Extension or LB probe | Autohealing (health check) |
| **Spot/Low priority** | Spot VMSS | Preemptible/Spot MIG |
| **Eviction** | Deallocate or Delete | Always Delete (30s warning) |
| **Max instances** | 1000 per VMSS | 2000 per MIG |
| **Stateful** | Supported | Stateful MIG (persistent disks) |
| **Image update** | Requires reimage | Create new template → rolling update |
| **Load Balancer** | Azure LB / App Gateway | Backend Service + Health Check |

### Machine Type Families (Interview Reference)

| Family | Use Case | Example | Azure Equivalent |
|--------|----------|---------|-----------------|
| **E2** | General purpose, cost-optimized | e2-standard-4 | B-series |
| **N2** | General purpose, balanced | n2-standard-8 | D-series |
| **C2/C3** | Compute-optimized | c2-standard-16 | F-series |
| **M2** | Memory-optimized | m2-ultramem-208 | M-series |
| **A2** | GPU (ML/AI) | a2-highgpu-1g | NC-series |
| **T2D** | Scale-out (ARM-like perf) | t2d-standard-4 | Dps-series |

### Preemptible vs Spot VMs

**English:**
- **Preemptible** (legacy): Max 24-hour lifetime, 60-91% discount, 30-second shutdown notice
- **Spot** (new): Same discount, NO 24-hour limit, but dynamic pricing
- **Key design**: Workloads must be fault-tolerant (CI builds, batch jobs, data processing)
- **MIG handles recovery**: If VM is preempted, MIG auto-creates a replacement

**தமிழ்:**
- **Preemptible** (legacy): Max 24 மணி lifetime, 60-91% discount, 30-second shutdown notice
- **Spot** (new): Same discount, 24-hour limit இல்லை, ஆனால் dynamic pricing
- **Key design**: Workloads fault-tolerant ஆக இருக்க வேண்டும் (CI builds, batch jobs)
- **MIG recovery handle செய்யும்**: VM preempt ஆனால், MIG auto-create replacement

---

## 🧠 Byheart for Interview

### GCP Compute Key Facts
```
1. Instance Template is IMMUTABLE — any change = new template + rolling update
2. MIG can be ZONAL (one zone) or REGIONAL (multi-zone for HA)
3. Preemptible VMs: 60-91% cheaper, 30s warning, max 24h, auto-restart = false
4. Spot VMs: same discount as preemptible, no 24h limit, dynamic pricing
5. MIG Autoscaler metrics: CPU, LB utilization, Pub/Sub queue, custom metrics
6. MIG Autohealing: health check fails → VM auto-recreated
7. Rolling Update: maxSurge + maxUnavailable control update speed
8. Instance Template versioning: use name suffix (v1, v2) or date stamps
9. Metadata startup-script runs on EVERY boot (vs Azure custom-script = once)
10. Network tags on template → firewall rules apply to all MIG instances
```

### Cost Optimization Formula (CI Workloads)
```
Standard e2-standard-4:  $0.134/hr × 10 VMs × 24h × 30d = $965/month
Preemptible e2-standard-4: $0.040/hr × 10 VMs × 24h × 30d = $288/month
                                                     Savings: ~70%

Design: MIG with preemptible + autoscaler (scale to 0 at night)
Night savings: 12h off → additional 50% reduction → $144/month total!
```

### Interview Golden Answers
```
Q: "How would you design cost-optimized CI on GCP?"
A: "MIG with preemptible/spot VMs, autoscaler based on Pub/Sub queue depth 
    (pending builds), scale-to-zero during off-hours, instance template with 
    startup script that auto-registers as CI agent. MIG autohealing handles 
    preemption gracefully. This gives ~70-85% savings vs on-demand."

Q: "Instance Template is immutable — how do you handle updates?"
A: "Create new instance template with updated config → update MIG to use new 
    template → choose update strategy: PROACTIVE (immediately roll) or 
    OPPORTUNISTIC (update on next recreation). For CI agents, PROACTIVE with 
    maxSurge=3 gives fast rollout without downtime."

Q: "Azure VMSS vs GCP MIG — key architectural differences?"
A: "Three main differences:
    1. Template: VMSS has inline model; GCP MIG references separate immutable template
    2. Autoscaler: Azure built-in to VMSS; GCP is separate resource (more flexible)
    3. Update: Azure does in-place reimage; GCP replaces VMs entirely (cleaner state)"
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/gcp-compute && cd ~/tf-lab/gcp-compute
```

### Exercise 1: Instance Template + MIG

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

# ─── INSTANCE TEMPLATE (Blueprint for CI agents) ──────────────────
resource "google_compute_instance_template" "ci_agent" {
  name_prefix  = "ci-agent-"
  machine_type = "e2-standard-4"
  region       = var.region

  # Preemptible for cost savings
  scheduling {
    preemptible         = true
    automatic_restart   = false  # Required for preemptible
    on_host_maintenance = "TERMINATE"
  }

  disk {
    source_image = "ubuntu-os-cloud/ubuntu-2204-lts"
    auto_delete  = true
    boot         = true
    disk_size_gb = 100
    disk_type    = "pd-ssd"
  }

  network_interface {
    subnetwork = var.subnet_id
    # No access_config = no external IP (private only)
  }

  # Network tags for firewall rules
  tags = ["ci-agent", "allow-ssh"]

  # Startup script: install CI tools on every boot
  metadata_startup_script = <<-SCRIPT
    #!/bin/bash
    apt-get update && apt-get install -y docker.io git curl
    # Register as CI agent
    curl -sL https://ci-server/register.sh | bash
  SCRIPT

  metadata = {
    enable-oslogin = "TRUE"  # Use IAM for SSH access
  }

  service_account {
    email  = var.ci_service_account
    scopes = ["cloud-platform"]
  }

  # Lifecycle: create new before destroying old (for template updates)
  lifecycle {
    create_before_destroy = true
  }
}
EOF
```

### Exercise 2: Managed Instance Group with Autoscaler

```hcl
cat > mig.tf << 'EOF'
# ─── REGIONAL MIG (multi-zone HA) ─────────────────────────────────
resource "google_compute_region_instance_group_manager" "ci_agents" {
  name               = "mig-ci-agents"
  base_instance_name = "ci-agent"
  region             = var.region

  version {
    instance_template = google_compute_instance_template.ci_agent.id
    name              = "primary"
  }

  target_size = 3  # Desired number (autoscaler overrides this)

  # Named port for load balancer (if needed)
  named_port {
    name = "http"
    port = 8080
  }

  # Rolling update policy
  update_policy {
    type                           = "PROACTIVE"
    minimal_action                 = "REPLACE"
    most_disruptive_allowed_action = "REPLACE"
    max_surge_fixed                = 3
    max_unavailable_fixed          = 0
    replacement_method             = "SUBSTITUTE"
  }

  # Autohealing: recreate unhealthy VMs
  auto_healing_policies {
    health_check      = google_compute_health_check.ci_agent.id
    initial_delay_sec = 300  # 5 min for startup
  }
}

# ─── HEALTH CHECK (for autohealing) ───────────────────────────────
resource "google_compute_health_check" "ci_agent" {
  name                = "hc-ci-agent"
  check_interval_sec  = 30
  timeout_sec         = 10
  healthy_threshold   = 2
  unhealthy_threshold = 3

  http_health_check {
    port         = 8080
    request_path = "/health"
  }
}

# ─── AUTOSCALER ───────────────────────────────────────────────────
resource "google_compute_region_autoscaler" "ci_agents" {
  name   = "autoscaler-ci-agents"
  region = var.region
  target = google_compute_region_instance_group_manager.ci_agents.id

  autoscaling_policy {
    max_replicas    = 20
    min_replicas    = 1
    cooldown_period = 120

    # Scale based on CPU
    cpu_utilization {
      target = 0.7
    }

    # Scale-in controls (prevent aggressive scale-down)
    scale_in_control {
      max_scaled_in_replicas {
        fixed = 2  # Max 2 VMs removed at a time
      }
      time_window_sec = 300  # Over 5 minutes
    }
  }
}
EOF
```

### Exercise 3: Variables and Outputs

```hcl
cat > variables.tf << 'EOF'
variable "project_id" {
  type = string
}

variable "region" {
  type    = string
  default = "us-central1"
}

variable "subnet_id" {
  type        = string
  description = "Subnet for CI agent VMs"
}

variable "ci_service_account" {
  type        = string
  description = "Service account email for CI agents"
}
EOF

cat > outputs.tf << 'EOF'
output "instance_group" {
  value = google_compute_region_instance_group_manager.ci_agents.instance_group
}

output "template_id" {
  value = google_compute_instance_template.ci_agent.id
}
EOF
```

```bash
terraform init
terraform validate
terraform plan -var="project_id=my-project" -var="subnet_id=projects/my-project/regions/us-central1/subnetworks/subnet-ci" -var="ci_service_account=ci@my-project.iam.gserviceaccount.com"
```

---

## 🔥 Scenario Challenge

### Scenario: Cost-Optimized CI with Zero-Downtime Updates

**English:**
Design a CI compute infrastructure that:
1. Uses spot VMs for 70%+ cost savings
2. Handles preemption gracefully (builds don't fail permanently)
3. Scales from 2 (idle) to 50 (peak) based on build queue
4. Updates CI agent image weekly without disrupting running builds
5. Has health monitoring and auto-recovery

**தமிழ்:**
CI compute infrastructure design செய்யுங்கள்:
1. 70%+ cost savings-க்கு spot VMs பயன்படுத்தவும்
2. Preemption gracefully handle செய்யவும் (builds permanently fail ஆகாது)
3. Build queue-ஐ வைத்து 2 (idle) to 50 (peak) scale செய்யவும்
4. Running builds-ஐ disrupt செய்யாமல் weekly CI agent image update
5. Health monitoring மற்றும் auto-recovery

### Solution

```hcl
# Spot VMs with MIG — graceful preemption handling
resource "google_compute_instance_template" "ci_spot" {
  name_prefix  = "ci-spot-${formatdate("YYYYMMDD", timestamp())}-"
  machine_type = "e2-standard-8"

  scheduling {
    provisioning_model  = "SPOT"
    preemptible         = true
    automatic_restart   = false
    on_host_maintenance = "TERMINATE"
    # Instance termination action
    instance_termination_action = "DELETE"
  }

  metadata_startup_script = <<-SCRIPT
    #!/bin/bash
    # Graceful shutdown handler for preemption
    shutdown_handler() {
      echo "Preemption detected, draining CI agent..."
      /opt/ci/drain-agent.sh  # Finish current step, mark agent offline
    }
    trap shutdown_handler SIGTERM

    # Install and register
    /opt/ci/setup.sh
    /opt/ci/register-agent.sh
  SCRIPT

  disk {
    source_image = "projects/${var.project_id}/global/images/ci-agent-${var.image_version}"
    disk_size_gb = 200
    disk_type    = "pd-ssd"
  }

  network_interface {
    subnetwork = var.subnet_id
  }

  tags = ["ci-agent"]

  lifecycle { create_before_destroy = true }
}

# Autoscaler based on Pub/Sub (build queue depth)
resource "google_compute_region_autoscaler" "ci_queue_based" {
  name   = "autoscaler-ci-queue"
  region = var.region
  target = google_compute_region_instance_group_manager.ci_spot.id

  autoscaling_policy {
    max_replicas    = 50
    min_replicas    = 2
    cooldown_period = 60

    metric {
      name   = "pubsub.googleapis.com/subscription/num_undelivered_messages"
      type   = "GAUGE"
      target = 2  # 2 pending builds per VM
      filter = "resource.type = pubsub_subscription AND resource.labels.subscription_id = \"ci-build-queue\""
    }
  }
}
```

---

## 🏗️ Real Project Reference (EB)

**English:**
EB's CI infrastructure on GCP Compute:
- **MIG with Spot VMs**: e2-standard-8, 200GB SSD, preemptible = true
- **Autoscaler**: Based on Jenkins/Tekton build queue depth via Pub/Sub metrics
- **Template versioning**: New image weekly (packer-built), rolling update with maxSurge=5
- **Cost result**: $288/month vs $965/month (standard) = 70% savings for 10-agent fleet
- **Preemption handling**: Drain script marks agent offline, build retries on new VM
- **Scheduling**: Scale to min=1 on weekends, min=5 weekdays via Cloud Scheduler + Cloud Functions

**தமிழ்:**
EB CI infrastructure GCP Compute-ல்:
- **MIG with Spot VMs**: e2-standard-8, 200GB SSD, preemptible = true
- **Autoscaler**: Jenkins/Tekton build queue depth Pub/Sub metrics வழியாக
- **Template versioning**: வாரம் ஒரு முறை புதிய image (packer-built), maxSurge=5 உடன் rolling update
- **Cost result**: 10-agent fleet-க்கு $288/month vs $965/month = 70% savings
- **Preemption handling**: Drain script agent-ஐ offline mark செய்யும், build புதிய VM-ல் retry
- **Scheduling**: Weekends min=1, weekdays min=5 Cloud Scheduler + Cloud Functions வழியாக

---

## 🎤 Interview Q&A

### Q1: "Design cost-optimized compute for CI workloads on GCP"

**English:**
"I'd design a three-tier approach:
1. **Base capacity** (2-3 on-demand VMs): Always available for urgent builds
2. **Burst capacity** (MIG with Spot VMs): Scale 0-50 based on build queue
3. **Autoscaler trigger**: Pub/Sub subscription monitoring pending builds

Key decisions:
- Spot VMs for 70% savings (workloads are short-lived, retriable)
- MIG autohealing handles preemption (new VM in ~60 seconds)
- Weekly image updates via new instance template + PROACTIVE rolling update
- Cloud Scheduler scales min_replicas based on time-of-day

Cost model: 10 agents, e2-standard-4, spot pricing → $288/month vs $965/month standard"

**தமிழ்:**
"Three-tier approach design செய்வேன்:
1. **Base capacity** (2-3 on-demand VMs): Urgent builds-க்கு எப்போதும் available
2. **Burst capacity** (Spot VMs உடன் MIG): Build queue-ஐ வைத்து 0-50 scale
3. **Autoscaler trigger**: Pending builds monitor செய்ய Pub/Sub subscription

முக்கிய decisions:
- 70% savings-க்கு Spot VMs (workloads short-lived, retriable)
- MIG autohealing preemption handle செய்யும்
- Weekly image updates புதிய template + rolling update வழியாக
- Time-of-day based min_replicas Cloud Scheduler வழியாக"

### Q2: "How do you handle rolling updates in a MIG without downtime?"

**English:**
"Instance templates are immutable, so updates require:
1. Create new instance template (new image, new config)
2. Update MIG to reference new template
3. Set update policy: `type = PROACTIVE`, `maxSurge = 3`, `maxUnavailable = 0`
4. GCP creates 3 new VMs with new template, then terminates 3 old ones
5. Repeat until all instances are updated

For CI agents specifically, I use `OPPORTUNISTIC` — only update when a VM is naturally recreated (after a build completes or preemption). This avoids killing running builds."

**தமிழ்:**
"Instance templates immutable, updates-க்கு:
1. புதிய instance template create
2. MIG-ஐ புதிய template reference செய்ய update
3. Update policy set: `PROACTIVE`, `maxSurge = 3`, `maxUnavailable = 0`
4. GCP 3 புதிய VMs create, பிறகு 3 பழையவற்றை terminate

CI agents-க்கு `OPPORTUNISTIC` — VM naturally recreate ஆகும்போது மட்டும் update (build complete அல்லது preemption). Running builds kill ஆவதை தவிர்க்கலாம்."

### Q3: "Preemptible vs Spot VMs — which would you choose and why?"

**English:**
"I'd choose **Spot VMs** (the newer model) because:
1. No 24-hour maximum lifetime — long-running batch jobs are possible
2. Same 60-91% discount as preemptible
3. Dynamic pricing model aligns with market demand
4. Same 30-second termination notice
5. Preemptible is being deprecated in favor of Spot

The only case for preemptible: legacy Terraform configs not yet migrated. For new infra, always Spot."

**தமிழ்:**
"**Spot VMs** (புதிய model) choose செய்வேன் காரணம்:
1. 24-hour maximum lifetime இல்லை — long-running batch jobs possible
2. Preemptible போலவே 60-91% discount
3. Market demand-க்கு ஏற்ப dynamic pricing
4. அதே 30-second termination notice
5. Preemptible deprecated ஆகிக்கொண்டிருக்கிறது

Preemptible-க்கான ஒரே case: migrate செய்யாத legacy Terraform configs."

---

## ✅ Self-Check

| # | Question | Can You Answer? |
|---|----------|-----------------|
| 1 | Instance Template immutability — how to handle updates? | ☐ |
| 2 | Zonal MIG vs Regional MIG — when to use each? | ☐ |
| 3 | Preemptible vs Spot VMs — key differences? | ☐ |
| 4 | Autoscaler metrics: CPU vs custom (Pub/Sub queue)? | ☐ |
| 5 | MIG update policy: PROACTIVE vs OPPORTUNISTIC? | ☐ |
| 6 | Azure VMSS vs GCP MIG — 3 key differences? | ☐ |
| 7 | Cost calculation: spot savings for CI fleet? | ☐ |
| 8 | Autohealing vs Autoscaling — what's the difference? | ☐ |
| 9 | Graceful preemption handling design? | ☐ |
| 10 | Machine type selection for CI workloads? | ☐ |

---

*Next: [Module 11 — GCP GKE](module_11_gcp_gke.md)*
