# Module 06: Azure Compute
# மாடுல் 06: Azure Compute (கணினி வளங்கள்)

---

## 🎯 What? | என்ன?

**English:** Provision Azure VMs, VM Scale Sets, and managed disks with Terraform — for CI/CD agents, CT lab servers, and workload hosts.

**தமிழ்:** Azure VMs, VM Scale Sets, managed disks-ஐ Terraform-ல் provision செய்வது — CI/CD agents, CT lab servers, workload hosts-க்கு.

---

## 🛠️ Linux VM (CI/CD Agent / CT Lab)

```hcl
resource "azurerm_linux_virtual_machine" "agent" {
  name                  = "vm-ci-agent-01"
  resource_group_name   = azurerm_resource_group.main.name
  location              = var.location
  size                  = "Standard_D4s_v3"    # 4 vCPU, 16 GB RAM
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.agent.id]

  # SSH key (no password!)
  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"    # SSD
    disk_size_gb         = 128
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  # Cloud-init for bootstrap
  custom_data = base64encode(templatefile("${path.module}/cloud-init.yaml", {
    jenkins_url = var.jenkins_url
    agent_name  = "agent-01"
  }))

  tags = local.common_tags
}

# Data disk for builds (separate from OS)
resource "azurerm_managed_disk" "build_cache" {
  name                 = "disk-build-cache-01"
  location             = var.location
  resource_group_name  = azurerm_resource_group.main.name
  storage_account_type = "Premium_LRS"
  disk_size_gb         = 256
  create_option        = "Empty"
}

resource "azurerm_virtual_machine_data_disk_attachment" "build_cache" {
  managed_disk_id    = azurerm_managed_disk.build_cache.id
  virtual_machine_id = azurerm_linux_virtual_machine.agent.id
  lun                = 0
  caching            = "ReadWrite"
}
```

---

## 🛠️ VM Scale Set (Auto-scaling CI Agents)

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "agents" {
  name                = "vmss-ci-agents"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  sku                 = "Standard_D4s_v3"
  instances           = 2                    # Minimum instances

  admin_username = "azureuser"
  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 128
  }

  network_interface {
    name    = "nic-agents"
    primary = true
    ip_configuration {
      name      = "internal"
      primary   = true
      subnet_id = azurerm_subnet.ci.id
    }
  }

  # Auto-scale based on CPU
  custom_data = base64encode(file("${path.module}/cloud-init-agent.yaml"))

  tags = local.common_tags
}

# Auto-scale rules
resource "azurerm_monitor_autoscale_setting" "agents" {
  name                = "autoscale-ci-agents"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.agents.id

  profile {
    name = "default"
    capacity {
      minimum = 1
      default = 2
      maximum = 10
    }

    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.agents.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        operator           = "GreaterThan"
        threshold          = 70
      }
      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "2"
        cooldown  = "PT5M"
      }
    }

    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.agents.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        operator           = "LessThan"
        threshold          = 20
      }
      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT10M"
      }
    }
  }
}
```

---

## 🛠️ Availability Zones

```hcl
# Spread VMs across zones for HA
resource "azurerm_linux_virtual_machine" "ha_nodes" {
  count = 3

  name     = "vm-node-${count.index + 1}"
  location = var.location
  size     = "Standard_D4s_v3"
  zone     = tostring((count.index % 3) + 1)  # Zone 1, 2, 3

  # ... (same config as above)
}
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         AZURE COMPUTE CHEAT SHEET                │
├──────────────────────────────────────────────────┤
│ VM SIZES (common):                               │
│   Standard_D2s_v3  = 2 vCPU, 8 GB  (dev)       │
│   Standard_D4s_v3  = 4 vCPU, 16 GB (CI agent)  │
│   Standard_D8s_v3  = 8 vCPU, 32 GB (build)     │
│   Standard_D16s_v3 = 16 vCPU, 64 GB (heavy)    │
│                                                  │
│ DISK TYPES:                                      │
│   Standard_LRS = HDD (cheap, slow)               │
│   Premium_LRS  = SSD (fast, for builds!)         │
│   UltraSSD_LRS = Ultra (extreme IOPS)           │
│                                                  │
│ HA STRATEGY:                                     │
│   Availability Zones (cross-DC) — preferred      │
│   Availability Sets (same DC, diff racks)        │
│   VMSS + autoscale (dynamic)                     │
│                                                  │
│ BOOTSTRAP:                                       │
│   custom_data = cloud-init (first boot)          │
│   extensions  = post-deploy scripts              │
│   Ansible     = full configuration (preferred!)  │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: VM vs VMSS — when to use which?**
- VM: fixed workloads (database server, specific hardware CT lab machine)
- VMSS: dynamic workloads (CI agents, web servers, scale based on demand)

**Q: How to bootstrap a VM after creation?**
- cloud-init (custom_data): first-boot scripts, package install, user setup
- VM Extensions: post-deploy (install monitoring agents, join domain)
- Ansible: full configuration management (preferred for complex setups — our next module!)

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Linux VM with SSH key provision முடியும்
- [ ] VMSS with autoscale configure முடியும்
- [ ] Data disk attach முடியும்
- [ ] Availability zones for HA use முடியும்
