# Module 06: Azure Compute with Terraform

## 📖 Story: The Factory Floor

**English:**
Think of Azure compute like a factory. A single VM is one worker at a workstation — reliable but limited. A VMSS (Virtual Machine Scale Set) is a production line that can add or remove workers based on demand. Availability Zones are separate buildings — if one catches fire, the others keep running. Managed Disks are the tool cabinets bolted to each workstation.

In our EB (Embedded Build) project, we used VMSS for CI build agents. When developers push code, the queue grows, VMSS scales out new agents, builds complete, agents scale back to zero. No idle machines burning money. This is the interview gold — cost-efficient, auto-scaling CI infrastructure.

**தமிழ்:**
Azure compute-ஐ ஒரு தொழிற்சாலை போல நினைக்கவும். ஒரு VM என்பது ஒரு workstation-ல் ஒரு worker — நம்பகமானது ஆனால் limited. VMSS ஒரு production line — demand-க்கு ஏற்ப workers-ஐ add/remove செய்யலாம். Availability Zones தனி buildings — ஒன்று தீப்பிடித்தால் மற்றவை தொடரும். Managed Disks ஒவ்வொரு workstation-க்கும் bolted tool cabinets.

EB project-ல், CI build agents-க்கு VMSS பயன்படுத்தினோம். Developers code push செய்யும்போது queue grow ஆகும், VMSS புதிய agents-ஐ scale out செய்யும், builds முடிந்ததும் zero-க்கு scale back. Idle machines இல்லை — cost-efficient, auto-scaling CI infrastructure.

---

## 📊 Architecture & Concepts

### Compute Decision Matrix

| Need | Solution | When to Use |
|------|----------|-------------|
| Single persistent server | VM | Jump boxes, license servers |
| Auto-scaling identical servers | VMSS | CI agents, web tier, workers |
| Containers without infra mgmt | AKS | Microservices (next module) |
| Event-driven short tasks | Functions | Webhooks, triggers |
| Batch processing | Batch Service | Large parallel compute jobs |

### VMSS Architecture for CI Agents

```
┌──────────────────────────────────────────────────┐
│              Azure DevOps / Jenkins               │
│                  Build Queue                      │
└────────────────────┬─────────────────────────────┘
                     │ Queue depth metric
                     ▼
┌──────────────────────────────────────────────────┐
│              Autoscale Rules                      │
│  IF queue > 5 pending → scale out (+2 instances) │
│  IF queue = 0 for 10min → scale in (min: 0)     │
└────────────────────┬─────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────┐
│           VMSS: CI Build Agents                   │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐           │
│  │ VM0 │  │ VM1 │  │ VM2 │  │ VM3 │  → max:20 │
│  │Agent│  │Agent│  │Agent│  │Agent│            │
│  └─────┘  └─────┘  └─────┘  └─────┘           │
│                                                  │
│  OS: Ubuntu 22.04 (Custom Image with tools)      │
│  Size: Standard_D4s_v3 (4 vCPU, 16GB)          │
│  Data Disk: 128GB Premium SSD (build cache)      │
└──────────────────────────────────────────────────┘
```

### Availability Zones Strategy

**English:**
Azure has 3 availability zones per region (physically separate data centers):
- **Zone-redundant**: Resources spread across all 3 zones (best HA)
- **Zonal**: Pinned to specific zone (for low latency / compliance)
- **Non-zonal**: No zone preference (Azure decides placement)

For VMSS: use zone-redundant (`zones = [1, 2, 3]`) for production workloads.

**தமிழ்:**
Azure-ல் ஒரு region-க்கு 3 availability zones (physical-ஆக தனி data centers):
- **Zone-redundant**: அனைத்து 3 zones-லும் பரவியது (சிறந்த HA)
- **Zonal**: குறிப்பிட்ட zone-ல் pin செய்யப்பட்டது
- **Non-zonal**: Azure placement decide செய்யும்

VMSS-க்கு: production-ல் zone-redundant (`zones = [1, 2, 3]`) பயன்படுத்துங்கள்.

### Managed Disk Types

| Type | IOPS | Use Case | Cost |
|------|------|----------|------|
| Standard HDD | 500 | Backups, non-critical | Lowest |
| Standard SSD | 6000 | Dev/Test, light workloads | Low |
| Premium SSD | 20000 | Production, databases | Medium |
| Ultra Disk | 160000 | SAP HANA, heavy IO | Highest |

---

## 🧠 Byheart for Interview

### VM/VMSS Key Points
```
1. VMSS for stateless workloads (CI agents, web servers, workers)
2. Custom Image (Packer-built) > cloud-init for large tool installations
3. Separate OS disk (small, ephemeral) from Data disk (persistent, cache)
4. Spot instances for fault-tolerant workloads = 60-90% cost savings
5. VMSS + Autoscale rules = queue-based scaling for CI agents
6. Availability Zones for HA, Availability Sets for fault domains (legacy)
7. VM Extensions for post-deploy config (but prefer custom images)
8. Ephemeral OS disk for VMSS = faster scaling + no disk cost
```

### Autoscale Rule Patterns
```
CI Agent Pattern:
  Scale OUT: queue_depth > threshold → add instances
  Scale IN:  queue_depth = 0 for cooldown → remove to min(0)
  
Web Tier Pattern:
  Scale OUT: CPU > 70% for 5 min → add 2 instances
  Scale IN:  CPU < 30% for 10 min → remove 1 instance
  Min: 2, Max: 20
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/azure-compute && cd ~/tf-lab/azure-compute
```

### Exercise 1: VMSS for CI Agents

```hcl
cat > main.tf << 'EOF'
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

# ─── VMSS for CI Build Agents ────────────────────
resource "azurerm_linux_virtual_machine_scale_set" "ci_agents" {
  name                = "vmss-ci-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "Standard_D4s_v3"  # 4 vCPU, 16GB RAM
  instances           = 0                   # Start with zero — scale on demand

  zones = [1, 2, 3]  # Zone-redundant

  admin_username = "azureuser"
  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  # Custom image with all build tools pre-installed
  source_image_id = var.custom_image_id

  # OR use marketplace image:
  # source_image_reference {
  #   publisher = "Canonical"
  #   offer     = "0001-com-ubuntu-server-jammy"
  #   sku       = "22_04-lts-gen2"
  #   version   = "latest"
  # }

  os_disk {
    caching              = "ReadOnly"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = 30

    diff_disk_settings {
      option    = "Local"      # Ephemeral OS disk — fast, no cost
      placement = "CacheDisk"
    }
  }

  # Separate data disk for build cache
  data_disk {
    lun                  = 0
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 128
  }

  network_interface {
    name    = "nic-ci"
    primary = true

    ip_configuration {
      name      = "ipconfig"
      primary   = true
      subnet_id = var.subnet_id
    }
  }

  # Cloud-init to register as build agent
  custom_data = base64encode(templatefile("${path.module}/cloud-init.yaml", {
    agent_pool = var.agent_pool_name
    org_url    = var.devops_org_url
    pat_token  = var.agent_pat_token
  }))

  identity {
    type = "SystemAssigned"  # For Azure resource access
  }

  tags = var.tags
}

# ─── AUTOSCALE: Queue-based scaling ─────────────
resource "azurerm_monitor_autoscale_setting" "ci_scale" {
  name                = "autoscale-ci-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.ci_agents.id

  profile {
    name = "queue-based"

    capacity {
      minimum = 0   # Scale to zero when idle
      default = 0
      maximum = 20  # Cap costs
    }

    # Scale OUT when queue has pending jobs
    rule {
      metric_trigger {
        metric_name        = "ApproximateMessageCount"
        metric_resource_id = var.queue_id
        operator           = "GreaterThan"
        threshold          = 0
        statistic          = "Average"
        time_grain         = "PT1M"
        time_window        = "PT5M"
        time_aggregation   = "Average"
      }
      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "2"
        cooldown  = "PT5M"
      }
    }

    # Scale IN when queue empty
    rule {
      metric_trigger {
        metric_name        = "ApproximateMessageCount"
        metric_resource_id = var.queue_id
        operator           = "LessThanOrEqual"
        threshold          = 0
        statistic          = "Average"
        time_grain         = "PT1M"
        time_window        = "PT10M"
        time_aggregation   = "Average"
      }
      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }
  }
}
EOF
```

### Exercise 2: VM with Managed Disk

```hcl
cat > vm.tf << 'EOF'
# ─── Single VM (Jump Box / License Server) ───────
resource "azurerm_linux_virtual_machine" "jumpbox" {
  name                = "vm-jumpbox-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  size                = "Standard_B2s"  # Burstable — cheap for intermittent use
  zone                = "1"

  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  network_interface_ids = [azurerm_network_interface.jumpbox.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_SSD_LRS"
    disk_size_gb         = 30
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = var.tags
}

# ─── Managed Data Disk ───────────────────────────
resource "azurerm_managed_disk" "data" {
  name                 = "disk-jumpbox-data"
  location             = var.location
  resource_group_name  = var.resource_group_name
  storage_account_type = "Premium_LRS"
  create_option        = "Empty"
  disk_size_gb         = 64
  zone                 = "1"  # Must match VM zone
}

resource "azurerm_virtual_machine_data_disk_attachment" "data" {
  managed_disk_id    = azurerm_managed_disk.data.id
  virtual_machine_id = azurerm_linux_virtual_machine.jumpbox.id
  lun                = 0
  caching            = "ReadWrite"
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

### Scenario: "Design auto-scaling CI agents for 50-developer team"

**Requirements:**
- Developers push ~200 builds/day (peaks at 50/hour during morning)
- Each build needs: 4 vCPU, 16GB RAM, 100GB disk
- Builds take 15-30 minutes average
- Must scale to zero at night (cost saving)
- Agents need Docker, Go, Node.js, Java pre-installed
- Build artifacts cached across builds

**Your Design Answer:**

```
Architecture:
├── Custom Image (Packer)
│   ├── Ubuntu 22.04 base
│   ├── Docker, Go 1.21, Node 18, Java 17
│   ├── Common dependencies pre-pulled
│   └── Agent registration script
│
├── VMSS Configuration
│   ├── SKU: Standard_D4s_v3 (4 vCPU, 16GB)
│   ├── Ephemeral OS disk (fast boot, no cost)
│   ├── Premium SSD 128GB data disk (build cache)
│   ├── Zones: [1, 2, 3]
│   ├── Min: 0, Max: 15
│   └── Spot instances for non-critical builds (60% savings)
│
├── Autoscale Rules
│   ├── Scale OUT: queue > 0 → +3 instances (5 min cooldown)
│   ├── Scale IN: queue = 0 for 10 min → -1 instance
│   └── Schedule: Max=0 from 10PM-6AM (force zero)
│
└── Cost Optimization
    ├── Ephemeral OS disk: ~₹2000/month saved per instance
    ├── Spot for 70% of builds: ~60% savings
    ├── Scale-to-zero: zero cost outside work hours
    └── Estimated: ₹45K/month vs ₹2L always-on
```

---

## 🏗️ Real Project Reference: EB Embedded CI/CT

**English:**
In the EB (Elektrobit) Embedded Build project:
- **Problem**: 80+ developers, C/C++ builds taking 45 minutes, fixed 10-agent pool always at 100%
- **Solution**: VMSS with queue-based autoscaling
  - Custom image with ARM cross-compilers, AUTOSAR tools, build cache
  - Ephemeral OS disk for 30-second boot time
  - Separate 256GB Premium SSD for ccache (persisted across builds)
  - Scale: 0→20 agents based on Bitbucket pipeline queue
- **Result**: 
  - Build wait time: 30 min → 2 min
  - Cost: ₹3L/month fixed → ₹80K/month (73% reduction)
  - Night/weekend: zero running agents

**தமிழ்:**
EB Embedded Build project-ல்:
- **பிரச்சனை**: 80+ developers, C/C++ builds 45 minutes, fixed 10-agent pool எப்போதும் 100%
- **தீர்வு**: Queue-based autoscaling-உடன் VMSS
  - Custom image: ARM cross-compilers, AUTOSAR tools, build cache
  - Ephemeral OS disk: 30-second boot time
  - தனி 256GB Premium SSD: ccache (builds-க்கு இடையில் persisted)
  - Scale: Bitbucket pipeline queue அடிப்படையில் 0→20 agents
- **முடிவு**: 
  - Build wait time: 30 min → 2 min
  - Cost: ₹3L/month → ₹80K/month (73% குறைப்பு)
  - இரவு/weekend: zero agents

---

## 🎤 Interview Q&A

### Q1: "How do you design auto-scaling CI agents?"

**English Answer:**
"Queue-based autoscaling with VMSS. I create a custom Packer image with all build tools pre-installed — this gives consistent, fast boot times versus cloud-init which runs on every scale event. VMSS uses ephemeral OS disks for zero-cost fast provisioning and separate Premium SSD data disks for build cache persistence. Autoscale watches the build queue metric — scales out by 2-3 instances when jobs are pending, scales in to zero when queue is empty for 10 minutes. Key decisions: scale to zero for cost savings, use Spot instances for non-critical builds at 60% discount, and schedule-based rules to force zero outside work hours."

**தமிழ் Answer:**
"Queue-based autoscaling VMSS-உடன். Custom Packer image-ல் build tools pre-installed — cloud-init-ஐ விட consistent, fast boot times. VMSS ephemeral OS disks (zero-cost fast provisioning) + தனி Premium SSD data disks (build cache persistence). Autoscale build queue metric-ஐ watch செய்கிறது — jobs pending-ல் 2-3 instances scale out, queue 10 minutes empty-ல் zero-க்கு scale in. Key decisions: zero-க்கு scale (cost savings), Spot instances 60% discount-ல், work hours-க்கு வெளியே force zero."

### Q2: "Ephemeral OS disk vs regular — when and why?"

**English Answer:**
"Ephemeral OS disks use the VM's local SSD cache — zero additional cost, faster boot, and no network dependency for OS operations. Perfect for VMSS where instances are disposable: CI agents, web workers, batch processors. The tradeoff is data loss on deallocation — but for stateless workloads, that's ideal. I keep persistent data on attached data disks. In our EB project, ephemeral OS disks cut our VMSS boot time from 2 minutes to 30 seconds and eliminated OS disk costs entirely."

**தமிழ் Answer:**
"Ephemeral OS disks VM-ன் local SSD cache-ஐ பயன்படுத்துகின்றன — zero additional cost, faster boot, OS operations-க்கு network dependency இல்லை. VMSS-க்கு perfect (instances disposable): CI agents, web workers. Tradeoff: deallocation-ல் data loss — ஆனால் stateless workloads-க்கு ideal. Persistent data attached data disks-ல் வைப்பேன். EB project-ல், ephemeral OS disks boot time 2 minutes → 30 seconds குறைத்தது, OS disk costs eliminate ஆனது."

### Q3: "How do you choose VM sizes for different workloads?"

**English Answer:**
"I categorize by workload pattern: B-series burstable for intermittent workloads like jump boxes — cheap base price, burst when needed. D-series general purpose for CI agents and web servers — balanced CPU/memory. E-series memory-optimized for caches, in-memory databases. F-series compute-optimized for CPU-heavy builds. For CI agents specifically, D4s_v3 (4 vCPU, 16GB) is the sweet spot — enough for Docker builds, parallel compilation, and test execution without overpaying."

**தமிழ் Answer:**
"Workload pattern-படி categorize செய்வேன்: B-series — jump boxes போன்ற intermittent workloads-க்கு cheap. D-series — CI agents, web servers-க்கு balanced CPU/memory. E-series — caches, in-memory databases-க்கு memory-optimized. F-series — CPU-heavy builds-க்கு. CI agents-க்கு specifically D4s_v3 (4 vCPU, 16GB) sweet spot — Docker builds, parallel compilation, tests-க்கு போதுமானது."

### Q4: "VMSS vs AKS for CI agents — how do you decide?"

**English Answer:**
"It depends on build requirements. VMSS for: builds needing privileged Docker (docker-in-docker), large disk I/O (ccache), specific OS-level tools (ARM cross-compilers), or kernel-level operations. AKS for: containerized builds that don't need privileged access, faster scaling (pod startup < VM startup), and when you already have AKS infrastructure. In EB project, we chose VMSS because embedded C/C++ builds needed specific compiler toolchains, large ccache persistence, and Docker-in-Docker for firmware packaging."

**தமிழ் Answer:**
"Build requirements-ஐ பொறுத்தது. VMSS: privileged Docker, large disk I/O (ccache), specific OS-level tools, kernel-level operations தேவைப்படும்போது. AKS: containerized builds, privileged access தேவையில்லை, faster scaling (pod startup < VM startup), ஏற்கனவே AKS infrastructure இருக்கும்போது. EB project-ல் VMSS தேர்ந்தெடுத்தோம் — embedded C/C++ builds-க்கு specific compiler toolchains, large ccache, Docker-in-Docker தேவைப்பட்டதால்."

### Q5: "How do you handle secrets in VMSS instances?"

**English Answer:**
"Never hardcode secrets in images or user data. Three approaches: One, SystemAssigned Managed Identity — VMSS gets an identity, grant it access to Key Vault or other resources via RBAC. No credentials stored anywhere. Two, Key Vault VM extension — pulls certificates directly into the VM's certificate store. Three, for CI agent PAT tokens — store in Key Vault, retrieve at boot via managed identity. In Terraform, I assign identity to VMSS and create role assignments for needed resources."

**தமிழ் Answer:**
"Secrets-ஐ images அல்லது user data-ல் hardcode செய்யவே கூடாது. மூன்று approaches: ஒன்று, SystemAssigned Managed Identity — VMSS-க்கு identity கொடுத்து RBAC வழியாக Key Vault/resources access. Credentials எங்கும் store இல்லை. இரண்டு, Key Vault VM extension — certificates-ஐ நேரடியாக VM-க்கு pull செய்யும். மூன்று, CI agent PAT tokens — Key Vault-ல் store, boot-ல் managed identity வழியாக retrieve."

---

## ✅ Self-Check

- [ ] Can you explain VMSS queue-based autoscaling architecture?
- [ ] Can you justify ephemeral OS disk + separate data disk strategy?
- [ ] Can you design a CI agent VMSS from scratch (image, scaling, disks)?
- [ ] Can you explain VM size selection (B/D/E/F series) for different workloads?
- [ ] Can you compare VMSS vs AKS for CI agents with real criteria?
- [ ] Can you explain Availability Zones vs Availability Sets?
- [ ] Can you describe the EB project's CI scaling solution with numbers?
- [ ] Can you explain managed identity for VMSS (no hardcoded secrets)?
