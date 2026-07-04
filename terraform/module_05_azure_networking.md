# Module 05: Azure Networking with Terraform

## 📖 Story: The Highway System

**English:**
Think of Azure networking as designing a city's highway system. Your VNets are separate cities, subnets are neighborhoods within cities, NSGs are traffic police at each intersection, VNet Peering is the express highway connecting cities, and Private Endpoints are underground tunnels that let you reach services without going on public roads.

In our TVS OTA project, we designed 3 separate VNets — one for OTA services, one for PKI (certificate management), and one for Edge device management. They talk to each other via VNet Peering, but internet traffic NEVER touches our internal services. This is exactly what interviewers want to hear — segmentation with controlled connectivity.

**தமிழ்:**
Azure networking-ஐ ஒரு நகரத்தின் highway system வடிவமைப்பதாக நினைக்கவும். VNets தனித்தனி நகரங்கள், subnets நகரத்திற்குள் உள்ள பகுதிகள், NSGs ஒவ்வொரு சந்திப்பிலும் உள்ள traffic police, VNet Peering நகரங்களை இணைக்கும் express highway, Private Endpoints பொது சாலைகளில் செல்லாமல் services-ஐ அடைய உதவும் underground tunnels.

TVS OTA project-ல், 3 தனி VNets வடிவமைத்தோம் — OTA services-க்கு ஒன்று, PKI-க்கு ஒன்று, Edge device management-க்கு ஒன்று. VNet Peering மூலம் ஒன்றோடொன்று பேசுகின்றன, ஆனால் internet traffic எப்போதும் internal services-ஐ தொடாது. இது segmentation with controlled connectivity — interview-ல் சொல்ல வேண்டியது இதுதான்.

---

## 📊 Architecture & Concepts

### Hub-Spoke Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    HUB VNet (10.0.0.0/16)                    │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  │
│  │ Firewall │  │   VPN    │  │  Bastion  │  │    DNS    │  │
│  │  Subnet  │  │  Gateway │  │  Subnet   │  │  Resolver │  │
│  └──────────┘  └──────────┘  └───────────┘  └───────────┘  │
└────────┬──────────────┬──────────────┬──────────────────────┘
         │              │              │
    Peering        Peering        Peering
         │              │              │
┌────────▼───┐  ┌──────▼──────┐  ┌───▼──────────┐
│ SPOKE: OTA │  │ SPOKE: PKI  │  │ SPOKE: Edge  │
│ 10.1.0.0/16│  │ 10.2.0.0/16 │  │ 10.3.0.0/16  │
│            │  │             │  │              │
│ AKS Subnet │  │ HSM Subnet  │  │ IoT Subnet   │
│ App Subnet │  │ CA Subnet   │  │ MQTT Subnet  │
│ DB Subnet  │  │             │  │              │
└────────────┘  └─────────────┘  └──────────────┘
```

**English:**
Hub-Spoke is the gold standard for enterprise Azure networking:
- **Hub**: Shared services (Firewall, VPN, DNS, Bastion) — deployed once
- **Spokes**: Workload-specific VNets — isolated, connect to hub via peering
- **Why**: Centralized security policy, reduced cost, clear segmentation

**தமிழ்:**
Hub-Spoke enterprise Azure networking-க்கான gold standard:
- **Hub**: பகிரப்பட்ட services (Firewall, VPN, DNS, Bastion) — ஒரு முறை deploy
- **Spokes**: Workload-specific VNets — தனிமைப்படுத்தப்பட்டவை, peering மூலம் hub-உடன் இணைப்பு
- **ஏன்**: மையப்படுத்தப்பட்ட security policy, குறைந்த செலவு, தெளிவான segmentation

### Subnet Strategy

**English:**
Subnet design is about isolation + service placement:

| Subnet | Purpose | CIDR Example | Key Rule |
|--------|---------|--------------|----------|
| AKS Nodes | Kubernetes nodes | /21 (2048 IPs) | Needs large CIDR for pod IPs |
| App Gateway | WAF/Load balancer | /24 | Dedicated — no other resources |
| Database | PostgreSQL/Redis | /24 | NSG: only allow from AKS subnet |
| Private Endpoints | PaaS connections | /24 | Disable network policies |
| Bastion | Jump host | /26 (min required) | Named `AzureBastionSubnet` |

**தமிழ்:**
Subnet design என்பது isolation + service placement:
- AKS Nodes: pod IPs-க்கு பெரிய CIDR தேவை (/21)
- App Gateway: dedicated subnet — வேறு resources வேண்டாம்
- Database: NSG மூலம் AKS subnet-லிருந்து மட்டும் அனுமதி
- Private Endpoints: network policies disable செய்ய வேண்டும்
- Bastion: `AzureBastionSubnet` என்ற பெயர் கட்டாயம்

---

## 🧠 Byheart for Interview

### VNet Design Principles
```
1. Hub-Spoke topology for enterprise (centralized security)
2. /16 per VNet, /21-/24 per subnet based on workload
3. Plan IP space for AKS: nodes + pods + services (CNI needs large CIDR)
4. Separate subnets for: AKS, AppGW, DB, Private Endpoints, Bastion
5. NSGs at subnet level (not NIC) for manageability
6. Private Endpoints > Service Endpoints for security
7. VNet Peering is non-transitive — need hub for spoke-to-spoke
8. Azure Firewall in hub for egress filtering
```

### Key Differences to Remember

| Feature | Service Endpoint | Private Endpoint |
|---------|-----------------|------------------|
| Traffic path | Microsoft backbone | Private IP in your VNet |
| DNS | Public IP resolved | Private IP resolved |
| On-prem access | No | Yes (via VPN/ER) |
| Cross-region | No | Yes |
| Cost | Free | Per hour + data |
| Security | Source subnet restriction | Full network isolation |

### CIDR Planning Formula (AKS with Azure CNI)
```
Nodes × (max_pods_per_node + 1) + extra nodes for upgrades
Example: 10 nodes × (30+1) + 5 upgrade nodes = 315 IPs → /23 minimum
```

---

## ⚡ Quick Hands-on

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/azure-network && cd ~/tf-lab/azure-network
```

### Exercise 1: Hub-Spoke VNet with Peering

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

# ─── HUB VNET ───────────────────────────────────
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-hub-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "firewall" {
  name                 = "AzureFirewallSubnet"  # Must be this exact name
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = ["10.0.1.0/26"]
}

resource "azurerm_subnet" "bastion" {
  name                 = "AzureBastionSubnet"  # Must be this exact name
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = ["10.0.2.0/26"]
}

# ─── SPOKE VNET (OTA Services) ──────────────────
resource "azurerm_virtual_network" "spoke_ota" {
  name                = "vnet-ota-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = ["10.1.0.0/16"]
}

resource "azurerm_subnet" "aks" {
  name                 = "snet-aks"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.spoke_ota.name
  address_prefixes     = ["10.1.0.0/21"]  # Large for AKS CNI
}

resource "azurerm_subnet" "database" {
  name                 = "snet-database"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.spoke_ota.name
  address_prefixes     = ["10.1.8.0/24"]

  delegation {
    name = "postgresql"
    service_delegation {
      name = "Microsoft.DBforPostgreSQL/flexibleServers"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

resource "azurerm_subnet" "private_endpoints" {
  name                 = "snet-pe"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.spoke_ota.name
  address_prefixes     = ["10.1.9.0/24"]

  private_endpoint_network_policies_enabled = true
}

# ─── VNET PEERING (Bidirectional) ────────────────
resource "azurerm_virtual_network_peering" "hub_to_ota" {
  name                      = "hub-to-ota"
  resource_group_name       = var.resource_group_name
  virtual_network_name      = azurerm_virtual_network.hub.name
  remote_virtual_network_id = azurerm_virtual_network.spoke_ota.id

  allow_forwarded_traffic = true
  allow_gateway_transit   = true  # Hub shares its gateway
}

resource "azurerm_virtual_network_peering" "ota_to_hub" {
  name                      = "ota-to-hub"
  resource_group_name       = var.resource_group_name
  virtual_network_name      = azurerm_virtual_network.spoke_ota.name
  remote_virtual_network_id = azurerm_virtual_network.hub.id

  allow_forwarded_traffic = true
  use_remote_gateways     = true  # Spoke uses hub's gateway
}
EOF
```

### Exercise 2: NSG with Rules

```hcl
cat > nsg.tf << 'EOF'
# ─── NSG for AKS Subnet ─────────────────────────
resource "azurerm_network_security_group" "aks" {
  name                = "nsg-aks-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
}

resource "azurerm_network_security_rule" "allow_lb" {
  name                        = "AllowLoadBalancer"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "443"
  source_address_prefix       = "AzureLoadBalancer"
  destination_address_prefix  = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.aks.name
}

resource "azurerm_network_security_rule" "deny_internet" {
  name                        = "DenyInternetInbound"
  priority                    = 4000
  direction                   = "Inbound"
  access                      = "Deny"
  protocol                    = "*"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "Internet"
  destination_address_prefix  = "*"
  resource_group_name         = var.resource_group_name
  network_security_group_name = azurerm_network_security_group.aks.name
}

# Associate NSG to Subnet
resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.aks.id
  network_security_group_id = azurerm_network_security_group.aks.id
}
EOF
```

### Exercise 3: Private Endpoint for Storage

```hcl
cat > private_endpoint.tf << 'EOF'
resource "azurerm_private_endpoint" "blob" {
  name                = "pe-blob-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = azurerm_subnet.private_endpoints.id

  private_service_connection {
    name                           = "blob-connection"
    private_connection_resource_id = var.storage_account_id
    is_manual_connection           = false
    subresource_names              = ["blob"]
  }

  private_dns_zone_group {
    name                 = "blob-dns"
    private_dns_zone_ids = [azurerm_private_dns_zone.blob.id]
  }
}

resource "azurerm_private_dns_zone" "blob" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = var.resource_group_name
}

resource "azurerm_private_dns_zone_virtual_network_link" "blob" {
  name                  = "blob-vnet-link"
  resource_group_name   = var.resource_group_name
  private_dns_zone_name = azurerm_private_dns_zone.blob.name
  virtual_network_id    = azurerm_virtual_network.spoke_ota.id
}
EOF
```

```bash
terraform init
terraform validate
terraform plan  # Review — won't apply without real Azure creds
```

---

## 🔥 Scenario Challenge

### Scenario: "Design network for multi-cluster AKS deployment"

**Requirements:**
- 3 AKS clusters (dev, staging, prod) — each in separate VNet
- Shared services (monitoring, CI/CD) in hub
- PostgreSQL accessible only from AKS clusters
- No public internet access to any backend service
- On-prem connectivity via VPN

**Your Design:**

```
┌─────────────────────────────────────────────────────────┐
│ Design Answer:                                          │
│                                                         │
│ 1. Hub VNet: Firewall + VPN GW + Bastion + DNS         │
│ 2. 3 Spoke VNets: dev/staging/prod (each /16)          │
│ 3. Each spoke: AKS subnet(/21) + PE subnet(/24)        │
│ 4. Hub-to-spoke peering with gateway transit            │
│ 5. Azure Firewall for egress filtering                  │
│ 6. Private Endpoints for PostgreSQL in each spoke       │
│ 7. Private DNS zones linked to all VNets                │
│ 8. NSG: DB subnet allows only AKS subnet CIDR          │
│ 9. UDR: Force all egress through Firewall               │
│ 10. Network Watcher for monitoring                      │
└─────────────────────────────────────────────────────────┘
```

---

## 🏗️ Real Project Reference: TVS OTA Platform

**English:**
In TVS Connected Vehicle OTA platform:
- **3 VNets**: OTA (firmware delivery), PKI (certificate management), Edge (device communication)
- **Hub VNet**: Azure Firewall + VPN Gateway for factory connectivity
- **Private Endpoints**: Blob Storage (firmware packages), PostgreSQL (OTA metadata), Key Vault (signing keys)
- **NSG Strategy**: Whitelist-only — default deny all, explicit allow per service
- **Result**: Zero public-facing backend services, all PaaS accessed via private IPs

**தமிழ்:**
TVS Connected Vehicle OTA platform-ல்:
- **3 VNets**: OTA (firmware delivery), PKI (certificate management), Edge (device communication)
- **Hub VNet**: Azure Firewall + VPN Gateway தொழிற்சாலை connectivity-க்கு
- **Private Endpoints**: Blob Storage, PostgreSQL, Key Vault அனைத்தும் private IPs வழியாக
- **NSG Strategy**: Default deny all, ஒவ்வொரு service-க்கும் explicit allow
- **முடிவு**: பொது-facing backend services இல்லை, அனைத்து PaaS private IPs வழியாக அணுகப்படுகின்றன

---

## 🎤 Interview Q&A

### Q1: "How would you design Azure networking for a microservices platform?"

**English Answer:**
"I'd use hub-spoke topology. Hub VNet has shared services — Azure Firewall for centralized egress control, VPN Gateway for on-prem connectivity, and Azure Bastion for secure admin access. Each environment (dev/staging/prod) gets its own spoke VNet peered to hub. Within each spoke, I separate subnets by function — AKS nodes get /21 for CNI IP requirements, databases get /24 with strict NSGs allowing only AKS subnet traffic. All PaaS services connect via Private Endpoints with Private DNS zones. This gives us defense-in-depth: network segmentation, no public endpoints, centralized monitoring through Firewall logs."

**தமிழ் Answer:**
"Hub-spoke topology பயன்படுத்துவேன். Hub VNet-ல் shared services — Azure Firewall centralized egress control-க்கு, VPN Gateway on-prem connectivity-க்கு, Azure Bastion secure admin access-க்கு. ஒவ்வொரு environment-க்கும் (dev/staging/prod) தனி spoke VNet hub-உடன் peered. ஒவ்வொரு spoke-லும் function-படி subnets பிரிக்கிறேன் — AKS nodes-க்கு /21 (CNI IP requirements), databases-க்கு /24 strict NSGs-உடன். அனைத்து PaaS services Private Endpoints + Private DNS zones வழியாக connect. இது defense-in-depth கொடுக்கிறது."

### Q2: "Private Endpoint vs Service Endpoint — when do you use which?"

**English Answer:**
"Private Endpoints always for production. Service Endpoints are simpler and free — they route traffic over Microsoft backbone and restrict by source subnet. But they have limitations: no on-prem access, no cross-region, still resolve to public IP. Private Endpoints give you a private IP in your VNet — works from on-prem via VPN, cross-region, and fully private DNS resolution. The extra cost (₹3-4/hour) is negligible for production security. I only use Service Endpoints in dev environments where cost optimization is priority."

**தமிழ் Answer:**
"Production-க்கு எப்போதும் Private Endpoints. Service Endpoints எளிமையானவை, இலவசம் — Microsoft backbone வழியாக traffic route செய்து source subnet-ஐ restrict செய்கின்றன. ஆனால் limitations உள்ளன: on-prem access இல்லை, cross-region இல்லை, public IP-க்கே resolve ஆகும். Private Endpoints உங்கள் VNet-ல் private IP கொடுக்கின்றன — VPN வழியாக on-prem-லிருந்து வேலை செய்யும், cross-region, fully private DNS. Cost (₹3-4/hour) production security-க்கு negligible. Dev environments-ல் மட்டும் Service Endpoints பயன்படுத்துவேன்."

### Q3: "How do you handle DNS with Private Endpoints?"

**English Answer:**
"Three-layer approach: First, create Azure Private DNS Zones for each service — `privatelink.blob.core.windows.net`, `privatelink.postgres.database.azure.com`, etc. Second, link these zones to all VNets that need access. Third, configure hub VNet's DNS resolver as custom DNS for all VNets so on-prem clients can also resolve private endpoints. The Terraform pattern is: private_endpoint → private_dns_zone_group → links DNS zone to VNet. Without this, clients resolve public IPs even with Private Endpoints deployed."

**தமிழ் Answer:**
"மூன்று-அடுக்கு அணுகுமுறை: முதலில், ஒவ்வொரு service-க்கும் Azure Private DNS Zones உருவாக்குங்கள். இரண்டாவது, access தேவைப்படும் அனைத்து VNets-க்கும் zones-ஐ link செய்யுங்கள். மூன்றாவது, hub VNet's DNS resolver-ஐ custom DNS-ஆக configure செய்யுங்கள் — on-prem clients-ம் resolve செய்ய முடியும். Terraform pattern: private_endpoint → private_dns_zone_group → VNet link. இது இல்லாமல், Private Endpoints deploy செய்தாலும் clients public IPs-க்கே resolve ஆகும்."

### Q4: "VNet Peering — what are the gotchas?"

**English Answer:**
"Four key gotchas: One, peering is non-transitive — spoke A can't reach spoke B through hub unless you have Azure Firewall or NVA doing routing. Two, address spaces can't overlap — plan CIDR carefully upfront. Three, peering must be created in BOTH directions — Terraform handles this but you need two resources. Four, `allow_gateway_transit` on hub and `use_remote_gateways` on spoke — miss this and on-prem traffic can't reach spokes. In TVS project, we hit the non-transitive issue when PKI VNet needed to reach OTA VNet — solved with Azure Firewall routing in hub."

**தமிழ் Answer:**
"நான்கு முக்கிய gotchas: ஒன்று, peering non-transitive — spoke A, hub வழியாக spoke B-ஐ அடைய முடியாது Azure Firewall/NVA routing இல்லாமல். இரண்டு, address spaces overlap ஆகக்கூடாது. மூன்று, peering இரு திசைகளிலும் உருவாக்க வேண்டும். நான்கு, `allow_gateway_transit` hub-ல், `use_remote_gateways` spoke-ல் — இது தவறினால் on-prem traffic spokes-ஐ அடையாது. TVS project-ல் PKI VNet, OTA VNet-ஐ அடைய வேண்டியபோது non-transitive issue-ஐ சந்தித்தோம் — hub-ல் Azure Firewall routing மூலம் தீர்த்தோம்."

### Q5: "How do you size subnets for AKS with Azure CNI?"

**English Answer:**
"Azure CNI assigns real VNet IPs to every pod. Formula: (max_nodes × max_pods_per_node) + upgrade_buffer. For a 20-node cluster with 30 pods per node: 20 × 31 (node IP + pod IPs) = 620 IPs, plus 10% upgrade buffer = ~700 IPs → /22 (1024 IPs). I always round up to next power of 2. Common mistake is using /24 (256 IPs) — fine for 8 nodes but scaling fails. In TVS, we used /21 (2048 IPs) per AKS subnet to allow growth without re-IPing."

**தமிழ் Answer:**
"Azure CNI ஒவ்வொரு pod-க்கும் real VNet IPs assign செய்கிறது. Formula: (max_nodes × max_pods_per_node) + upgrade_buffer. 20-node cluster, 30 pods/node: 20 × 31 = 620 IPs, +10% = ~700 → /22 (1024 IPs). அடுத்த power of 2-க்கு round up செய்வேன். /24 (256 IPs) பயன்படுத்துவது common mistake — 8 nodes-க்கு சரி ஆனால் scaling fail ஆகும். TVS-ல் /21 (2048 IPs) per AKS subnet பயன்படுத்தினோம்."

---

## ✅ Self-Check

- [ ] Can you explain hub-spoke topology and WHY it's preferred?
- [ ] Can you calculate subnet size for AKS CNI?
- [ ] Can you explain Private Endpoint vs Service Endpoint with real reasons?
- [ ] Can you describe the full Private DNS flow?
- [ ] Can you design a network for 3-environment multi-cluster AKS?
- [ ] Can you explain VNet Peering gotchas (non-transitive, gateway transit)?
- [ ] Can you explain NSG best practices (subnet-level, default deny)?
- [ ] Can you draw the TVS 3-VNet topology from memory?
