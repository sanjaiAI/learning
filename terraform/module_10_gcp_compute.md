# Module 10: GCP Compute
# மாடுல் 10: GCP Compute (GCE Instances)

---

## 🎯 What? | என்ன?

**English:** Provision GCP Compute Engine instances, instance groups, and templates — for CI agents, build servers, and workload hosts.

**தமிழ்:** GCP Compute Engine instances, instance groups provision — CI agents, build servers-க்கு.

---

## 🛠️ GCE Instance

```hcl
resource "google_compute_instance" "ci_agent" {
  name         = "vm-ci-agent-01"
  machine_type = "e2-standard-4"    # 4 vCPU, 16 GB
  zone         = "${var.region}-a"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts"
      size  = 100    # GB
      type  = "pd-ssd"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.ci.id
    # No external IP (private only, use Cloud NAT for outbound)
  }

  metadata_startup_script = templatefile("${path.module}/startup.sh", {
    jenkins_url = var.jenkins_url
  })

  service_account {
    email  = google_service_account.ci_agent.email
    scopes = ["cloud-platform"]
  }

  tags = ["ci-agent", "allow-ssh"]

  labels = {
    environment = var.environment
    team        = "platform"
  }
}

# Preemptible (spot) for cost savings — non-critical workloads
resource "google_compute_instance" "spot_agent" {
  name         = "vm-spot-agent-01"
  machine_type = "e2-standard-8"
  zone         = "${var.region}-a"

  scheduling {
    preemptible                 = true
    automatic_restart           = false
    on_host_maintenance         = "TERMINATE"
    provisioning_model          = "SPOT"
    instance_termination_action = "STOP"
  }

  # ... (boot_disk, network same as above)
}
```

---

## 🛠️ Instance Template + Managed Instance Group (Auto-scaling)

```hcl
resource "google_compute_instance_template" "ci_agent" {
  name_prefix  = "tpl-ci-agent-"
  machine_type = "e2-standard-4"

  disk {
    source_image = "ubuntu-os-cloud/ubuntu-2204-lts"
    disk_size_gb = 128
    disk_type    = "pd-ssd"
    auto_delete  = true
    boot         = true
  }

  network_interface {
    subnetwork = google_compute_subnetwork.ci.id
  }

  metadata_startup_script = file("${path.module}/startup-agent.sh")

  service_account {
    email  = google_service_account.ci_agent.email
    scopes = ["cloud-platform"]
  }

  tags = ["ci-agent"]

  lifecycle {
    create_before_destroy = true    # Rolling update!
  }
}

resource "google_compute_instance_group_manager" "ci_agents" {
  name               = "mig-ci-agents"
  base_instance_name = "ci-agent"
  zone               = "${var.region}-a"
  target_size        = 2

  version {
    instance_template = google_compute_instance_template.ci_agent.id
  }

  update_policy {
    type                  = "PROACTIVE"
    minimal_action        = "REPLACE"
    max_surge_fixed       = 2
    max_unavailable_fixed = 0
  }
}

# Autoscaler
resource "google_compute_autoscaler" "ci_agents" {
  name   = "autoscaler-ci-agents"
  zone   = "${var.region}-a"
  target = google_compute_instance_group_manager.ci_agents.id

  autoscaling_policy {
    min_replicas    = 1
    max_replicas    = 10
    cooldown_period = 300

    cpu_utilization {
      target = 0.7    # Scale up at 70% CPU
    }
  }
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         GCP COMPUTE CHEAT SHEET                  │
├──────────────────────────────────────────────────┤
│ MACHINE TYPES:                                   │
│   e2-standard-4  = 4 vCPU, 16 GB (general)     │
│   e2-standard-8  = 8 vCPU, 32 GB (builds)      │
│   n2-highmem-8   = 8 vCPU, 64 GB (memory)      │
│   c2-standard-16 = 16 vCPU (compute-heavy)      │
│                                                  │
│ COST OPTIMIZATION:                               │
│   Spot/Preemptible = 60-91% cheaper!            │
│   Committed use = 1-3 year discount              │
│   Right-sizing recommendations                   │
│                                                  │
│ SCALING:                                         │
│   Instance Template → MIG → Autoscaler           │
│   create_before_destroy for rolling updates      │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] GCE instance provision முடியும்
- [ ] Instance template + MIG + autoscaler configure முடியும்
- [ ] Spot instances for cost savings explain முடியும்
