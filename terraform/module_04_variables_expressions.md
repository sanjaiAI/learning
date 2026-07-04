# Module 04: Variables, Expressions & Dynamic Patterns

## 📖 Story: The Configuration Blueprint

**English:**
Imagine you're an architect designing houses for different clients. The blueprint (your Terraform code) stays the same, but each client wants different colors, room sizes, and materials. Variables are your customization parameters — they let you build the same house differently for each client without redrawing the blueprint.

In our TVS OTA project, we deploy the same infrastructure across dev/staging/prod — but each environment has different cluster sizes, instance types, and security rules. Variables + expressions make this possible with ONE codebase.

**தமிழ்:**
நீங்கள் வெவ்வேறு வாடிக்கையாளர்களுக்கு வீடுகளை வடிவமைக்கும் கட்டிடக்கலைஞர் என்று கற்பனை செய்யுங்கள். Blueprint (உங்கள் Terraform code) ஒரே மாதிரி, ஆனால் ஒவ்வொரு வாடிக்கையாளரும் வெவ்வேறு நிறங்கள், அறை அளவுகள் விரும்புகிறார்கள். Variables என்பது உங்கள் customization parameters — blueprint-ஐ மாற்றாமல் ஒவ்வொரு வாடிக்கையாளருக்கும் வேறுபட்ட வீட்டை கட்ட உதவுகிறது.

TVS OTA project-ல், dev/staging/prod-க்கு ஒரே infrastructure deploy செய்கிறோம் — ஆனால் ஒவ்வொரு environment-க்கும் வெவ்வேறு cluster sizes, instance types உள்ளன. Variables + expressions ஒரே codebase-ல் இதை சாத்தியமாக்குகின்றன.

---

## 📊 Core Concepts

### 1. Variable Types

**English:**
Terraform supports these primitive and complex types:

| Type | Description | Example |
|------|-------------|---------|
| `string` | Text value | `"us-east-1"` |
| `number` | Numeric value | `3` |
| `bool` | True/False | `true` |
| `list(type)` | Ordered collection | `["a", "b", "c"]` |
| `set(type)` | Unique unordered collection | `toset(["a", "b"])` |
| `map(type)` | Key-value pairs | `{env = "prod", team = "ops"}` |
| `object({})` | Structured type with named attributes | Complex configs |
| `tuple([])` | Fixed-length, mixed-type list | `["hello", 42, true]` |

**தமிழ்:**
Terraform இந்த primitive மற்றும் complex types-ஐ ஆதரிக்கிறது:
- `string` — உரை மதிப்பு
- `number` — எண் மதிப்பு  
- `bool` — True/False
- `list` — வரிசைப்படுத்தப்பட்ட தொகுப்பு
- `map` — key-value ஜோடிகள்
- `object` — named attributes கொண்ட structured type
- `tuple` — நிலையான நீளம், கலப்பு வகை list

```hcl
# String
variable "region" {
  type    = string
  default = "asia-south1"
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
variable "availability_zones" {
  type    = list(string)
  default = ["a", "b", "c"]
}

# Map
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "e2-medium"
    prod = "n2-standard-4"
  }
}

# Object — THE POWER TYPE for architects
variable "cluster_config" {
  type = object({
    name         = string
    node_count   = number
    machine_type = string
    preemptible  = bool
    labels       = map(string)
  })
}

# Tuple — rarely used but interview-worthy
variable "mixed_config" {
  type    = tuple([string, number, bool])
  default = ["production", 3, true]
}
```

### 2. Variable Precedence (CRITICAL for interviews!)

**English:**
When the same variable has multiple values, Terraform uses this precedence (highest wins):

```
1. -var and -var-file CLI flags (HIGHEST priority)
2. *.auto.tfvars / *.auto.tfvars.json (alphabetical order)
3. terraform.tfvars / terraform.tfvars.json
4. Environment variables (TF_VAR_name)
5. Variable default value (LOWEST priority)
```

**தமிழ்:**
ஒரே variable-க்கு பல values இருக்கும்போது, Terraform இந்த முன்னுரிமை வரிசையைப் பயன்படுத்துகிறது (highest wins):
- CLI flags (-var) — மிக உயர்ந்த முன்னுரிமை
- auto.tfvars files
- terraform.tfvars
- Environment variables (TF_VAR_name)
- Default value — மிகக் குறைந்த முன்னுரிமை

```hcl
# Environment variable
# export TF_VAR_region="us-west-2"

# terraform.tfvars
region = "asia-south1"

# CLI override (wins!)
# terraform apply -var="region=europe-west1"
```

### 3. Locals — Computed Values

**English:**
Locals are like "private variables" — computed within the module, not exposed to callers. Use them to avoid repetition and create derived values.

**தமிழ்:**
Locals என்பது "private variables" போன்றது — module-க்குள் compute செய்யப்படும், callers-க்கு expose ஆகாது. மீண்டும் மீண்டும் எழுதுவதை தவிர்க்கவும் derived values உருவாக்கவும் பயன்படுத்துங்கள்.

```hcl
locals {
  # Derived naming
  name_prefix = "${var.project}-${var.environment}"
  
  # Computed tags (merge common + specific)
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Team        = "platform-engineering"
  }
  
  # Conditional logic
  is_production = var.environment == "prod"
  node_count    = local.is_production ? 5 : 2
  
  # Complex derivation
  subnet_cidrs = [for i, az in var.availability_zones : 
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
}
```

### 4. Variables vs Locals vs Data Sources — When to Use What

**English:**

| Use Case | Use | Why |
|----------|-----|-----|
| User-configurable value | `variable` | Caller controls it |
| Computed/derived value | `local` | Internal logic, DRY |
| Existing resource lookup | `data source` | Read external state |
| Repeated expression | `local` | Avoid copy-paste |
| Sensitive input (passwords) | `variable` with `sensitive = true` | Mask in output |

**தமிழ்:**
- User மாற்ற வேண்டிய value → `variable`
- Code-க்குள் compute செய்யப்படும் value → `local`
- ஏற்கனவே உள்ள resource-ஐ படிக்க → `data source`
- மீண்டும் மீண்டும் வரும் expression → `local`
- Sensitive input → `variable` with `sensitive = true`

---

## 📊 Expressions Deep Dive

### 5. Conditional Expressions

```hcl
# Ternary — most common
resource "google_container_node_pool" "main" {
  node_count = var.environment == "prod" ? 5 : 2
  
  # Conditional resource creation
  preemptible = var.environment != "prod" ? true : false
}

# Conditional with complex logic
locals {
  machine_type = (
    var.environment == "prod" ? "n2-standard-8" :
    var.environment == "staging" ? "n2-standard-4" :
    "e2-medium"
  )
}
```

### 6. For Expressions

**English:**
For expressions transform collections — like map/filter in Python.

**தமிழ்:**
For expressions collections-ஐ மாற்றும் — Python-ல் map/filter போன்றது.

```hcl
# List comprehension — create list of names
locals {
  upper_zones = [for z in var.zones : upper(z)]
  # ["A", "B", "C"]
}

# Map comprehension — transform to map
locals {
  zone_map = {for z in var.zones : z => "${var.region}-${z}"}
  # {a = "asia-south1-a", b = "asia-south1-b"}
}

# Filtering with if clause
locals {
  production_clusters = [
    for name, config in var.clusters : name
    if config.environment == "prod"
  ]
}

# Nested for — flatten later
locals {
  all_subnets = flatten([
    for vpc_name, vpc_config in var.vpcs : [
      for subnet in vpc_config.subnets : {
        vpc_name    = vpc_name
        subnet_name = subnet.name
        cidr        = subnet.cidr
      }
    ]
  ])
}
```

### 7. Splat Expressions

```hcl
# Without splat
output "instance_ids" {
  value = [for i in google_compute_instance.nodes : i.id]
}

# With splat (shorter!)
output "instance_ids" {
  value = google_compute_instance.nodes[*].id
}

# Splat on nested attributes
output "all_ips" {
  value = google_compute_instance.nodes[*].network_interface[0].access_config[0].nat_ip
}
```

### 8. Dynamic Blocks (INTERVIEW FAVORITE!)

**English:**
Dynamic blocks generate repeated nested blocks inside resources. Think of them as "for loops for resource configuration blocks."

**தமிழ்:**
Dynamic blocks resources-க்குள் repeated nested blocks-ஐ உருவாக்கும். "Resource configuration blocks-க்கான for loops" என்று நினைக்கவும்.

```hcl
# EB CI/CT Project — Dynamic security rules
variable "firewall_rules" {
  type = list(object({
    name      = string
    port      = number
    protocol  = string
    source    = list(string)
  }))
  default = [
    {
      name     = "allow-ssh"
      port     = 22
      protocol = "tcp"
      source   = ["10.0.0.0/8"]
    },
    {
      name     = "allow-https"
      port     = 443
      protocol = "tcp"
      source   = ["0.0.0.0/0"]
    },
    {
      name     = "allow-jenkins"
      port     = 8080
      protocol = "tcp"
      source   = ["10.0.0.0/8"]
    }
  ]
}

resource "google_compute_firewall" "rules" {
  name    = "${local.name_prefix}-fw"
  network = google_compute_network.main.id

  dynamic "allow" {
    for_each = var.firewall_rules
    content {
      protocol = allow.value.protocol
      ports    = [allow.value.port]
    }
  }

  source_ranges = flatten([for rule in var.firewall_rules : rule.source])
}

# Dynamic blocks with conditionals
resource "google_container_cluster" "main" {
  name = var.cluster_name

  dynamic "maintenance_policy" {
    for_each = var.environment == "prod" ? [1] : []
    content {
      recurring_window {
        start_time = "2024-01-01T02:00:00Z"
        end_time   = "2024-01-01T06:00:00Z"
        recurrence = "FREQ=WEEKLY;BYDAY=SA"
      }
    }
  }
}
```

---

## 📊 Built-in Functions (Interview Must-Know)

### 9. Essential Functions

```hcl
# lookup — safe map access with default
locals {
  instance_type = lookup(var.instance_types, var.environment, "e2-medium")
  # Returns "e2-medium" if key not found
}

# merge — combine maps (right wins on conflicts)
locals {
  all_tags = merge(
    local.common_tags,
    var.extra_tags,
    { Name = "${local.name_prefix}-resource" }
  )
}

# flatten — collapse nested lists
locals {
  all_subnets = flatten([
    module.vpc_dev.subnet_ids,
    module.vpc_prod.subnet_ids,
  ])
}

# try — graceful error handling (Terraform 0.13+)
locals {
  db_port = try(var.database_config.port, 5432)
  # Returns 5432 if .port doesn't exist or errors
}

# can — test if expression is valid (returns bool)
locals {
  has_custom_port = can(var.database_config.port)
}

# coalesce — first non-null/non-empty value
locals {
  region = coalesce(var.override_region, var.default_region, "us-central1")
}

# formatlist — format each item in a list
locals {
  fqdn_list = formatlist("%s.%s", var.hostnames, var.domain)
  # ["web.example.com", "api.example.com"]
}

# zipmap — create map from two lists
locals {
  subnet_map = zipmap(var.subnet_names, var.subnet_cidrs)
}

# cidrsubnet — calculate subnet CIDRs
locals {
  subnets = [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i)]
  # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}
```

### 10. Type Constraints and Validation Rules

**English:**
Validation blocks enforce rules on variable inputs — catch errors BEFORE `terraform apply`.

**தமிழ்:**
Validation blocks variable inputs-ல் rules-ஐ enforce செய்யும் — `terraform apply`-க்கு முன்பே errors-ஐ கண்டறியும்.

```hcl
# String validation — environment names
variable "environment" {
  type        = string
  description = "Deployment environment"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Number range validation
variable "node_count" {
  type = number
  
  validation {
    condition     = var.node_count >= 1 && var.node_count <= 100
    error_message = "Node count must be between 1 and 100."
  }
}

# Regex validation — naming convention
variable "project_name" {
  type = string
  
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,20}$", var.project_name))
    error_message = "Project name must be lowercase, start with letter, 3-21 chars."
  }
}

# Complex object validation
variable "cluster_config" {
  type = object({
    name         = string
    node_count   = number
    machine_type = string
    environment  = string
  })

  validation {
    condition     = contains(["dev", "staging", "prod"], var.cluster_config.environment)
    error_message = "Cluster environment must be dev, staging, or prod."
  }

  validation {
    condition     = var.cluster_config.node_count >= 1
    error_message = "Node count must be at least 1."
  }
}

# EB CI/CT — Validate pipeline agent names
variable "agent_pool_name" {
  type = string
  
  validation {
    condition     = can(regex("^(eb|tvs)-(dev|prod)-agent-[0-9]+$", var.agent_pool_name))
    error_message = "Agent pool name must match pattern: {project}-(dev|prod)-agent-{number}."
  }
}
```

### 11. Sensitive Variables

**English:**
Mark variables as sensitive to prevent them appearing in CLI output, logs, or state file display.

**தமிழ்:**
CLI output, logs, அல்லது state file display-ல் தோன்றாமல் தடுக்க variables-ஐ sensitive என mark செய்யுங்கள்.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

variable "api_keys" {
  type      = map(string)
  sensitive = true
}

# Sensitive in locals
locals {
  connection_string = "postgresql://${var.db_user}:${var.db_password}@${var.db_host}:5432"
}

output "connection_info" {
  value     = local.connection_string
  sensitive = true  # MUST mark output sensitive too
}
```

> ⚠️ **Interview trap:** `sensitive = true` hides from CLI only — value is STILL in state file in plaintext! Use remote state encryption + access controls.

---

## 🧠 Byheart for Interview

### Variable Precedence (memorize this order!)
```
CLI -var > CLI -var-file > *.auto.tfvars > terraform.tfvars > TF_VAR_env > default
```

### When to use what:
```
variable  → Input from user/caller (configurable)
local     → Computed value (internal, DRY)
data      → Read existing infrastructure
output    → Expose values to callers/other modules
```

### Key function signatures:
```
lookup(map, key, default)     → Safe map access
merge(map1, map2, ...)        → Combine maps (right wins)
flatten([list1, list2])       → Collapse nested lists
try(expr, fallback)           → Graceful error handling
can(expr)                     → Test if expression valid (bool)
coalesce(val1, val2, ...)     → First non-null value
cidrsubnet(prefix, bits, num) → Calculate subnet CIDR
```

### Dynamic block pattern:
```hcl
dynamic "BLOCK_NAME" {
  for_each = COLLECTION
  content {
    attr = BLOCK_NAME.value.field
  }
}
```

---

## ⚡ Quick Hands-on Lab

### Lab Setup

```bash
ssh root@203.57.85.108
mkdir -p ~/tf-lab/module04-variables && cd ~/tf-lab/module04-variables
```

### Exercise 1: Variable Types and Precedence

```bash
cat > variables.tf << 'EOF'
variable "project" {
  type        = string
  description = "Project name"
  default     = "demo-project"
}

variable "environment" {
  type        = string
  description = "Environment name"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "node_count" {
  type    = number
  default = 2
  
  validation {
    condition     = var.node_count >= 1 && var.node_count <= 50
    error_message = "Node count must be 1-50."
  }
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "allowed_ports" {
  type    = list(number)
  default = [22, 80, 443]
}

variable "instance_types" {
  type = map(string)
  default = {
    dev     = "e2-small"
    staging = "e2-medium"
    prod    = "n2-standard-4"
  }
}

variable "cluster_config" {
  type = object({
    name         = string
    node_count   = number
    preemptible  = bool
    labels       = map(string)
  })
  default = {
    name        = "main-cluster"
    node_count  = 3
    preemptible = false
    labels      = { team = "platform" }
  }
}
EOF

cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
}

locals {
  name_prefix = "${var.project}-${var.environment}"
  is_prod     = var.environment == "prod"
  
  computed_node_count = local.is_prod ? max(var.node_count, 3) : var.node_count
  
  instance_type = lookup(var.instance_types, var.environment, "e2-micro")
  
  common_tags = merge(
    {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
    },
    var.cluster_config.labels
  )
}

output "name_prefix" {
  value = local.name_prefix
}

output "computed_config" {
  value = {
    instance_type = local.instance_type
    node_count    = local.computed_node_count
    is_production = local.is_prod
    tags          = local.common_tags
  }
}

output "allowed_ports_upper" {
  value = [for p in var.allowed_ports : p if p != 22]
}
EOF

cat > terraform.tfvars << 'EOF'
project     = "tvs-ota"
environment = "dev"
node_count  = 3
EOF
```

```bash
# Initialize and test
terraform init
terraform plan

# Test precedence — CLI overrides tfvars
terraform plan -var="environment=prod" -var="node_count=5"

# Test validation — should fail!
terraform plan -var="environment=invalid"

# Test with env var
export TF_VAR_environment="staging"
terraform plan
unset TF_VAR_environment
```

### Exercise 2: Dynamic Blocks and For Expressions

```bash
cat > dynamic_demo.tf << 'EOF'
variable "firewall_rules" {
  type = list(object({
    name     = string
    ports    = list(number)
    protocol = string
    sources  = list(string)
    priority = number
  }))
  default = [
    {
      name     = "allow-ssh"
      ports    = [22]
      protocol = "tcp"
      sources  = ["10.0.0.0/8"]
      priority = 100
    },
    {
      name     = "allow-web"
      ports    = [80, 443]
      protocol = "tcp"
      sources  = ["0.0.0.0/0"]
      priority = 200
    },
    {
      name     = "allow-monitoring"
      ports    = [9090, 9093, 3000]
      protocol = "tcp"
      sources  = ["10.0.0.0/8"]
      priority = 300
    }
  ]
}

locals {
  # For expression — filter high priority rules
  high_priority_rules = [
    for rule in var.firewall_rules : rule
    if rule.priority <= 200
  ]
  
  # For expression — create a map
  rule_map = {
    for rule in var.firewall_rules : rule.name => rule
  }
  
  # Flatten all ports from all rules
  all_ports = distinct(flatten([
    for rule in var.firewall_rules : rule.ports
  ]))
  
  # Conditional list
  monitoring_ports = var.enable_monitoring ? [9090, 9093, 3000] : []
}

output "high_priority_rules" {
  value = local.high_priority_rules[*].name
}

output "rule_map_keys" {
  value = keys(local.rule_map)
}

output "all_unique_ports" {
  value = sort(local.all_ports)
}

output "monitoring_ports" {
  value = local.monitoring_ports
}
EOF

terraform plan
```

### Exercise 3: Functions Playground

```bash
cat > functions_demo.tf << 'EOF'
locals {
  # lookup with default
  region_map = {
    india  = "asia-south1"
    us     = "us-central1"
    europe = "europe-west1"
  }
  selected_region = lookup(local.region_map, "japan", "us-central1")
  
  # merge maps
  base_labels = { app = "ota", managed_by = "terraform" }
  env_labels  = { environment = var.environment }
  all_labels  = merge(local.base_labels, local.env_labels)
  
  # try/can for safe access
  test_object = {
    nested = {
      value = "found-it"
    }
  }
  safe_value   = try(local.test_object.nested.value, "default")
  missing_val  = try(local.test_object.missing.value, "not-found")
  has_nested   = can(local.test_object.nested.value)
  
  # coalesce
  final_region = coalesce("", "", "asia-south1")  # returns "asia-south1"
  
  # cidrsubnet
  vpc_cidr = "10.0.0.0/16"
  subnet_cidrs = [for i in range(4) : cidrsubnet(local.vpc_cidr, 8, i)]
  
  # formatlist
  hostnames = ["web", "api", "worker"]
  fqdns     = formatlist("%s.tvs-ota.internal", local.hostnames)
  
  # zipmap
  zone_names = ["zone-a", "zone-b", "zone-c"]
  zone_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  zone_map   = zipmap(local.zone_names, local.zone_cidrs)
}

output "function_results" {
  value = {
    lookup_result  = local.selected_region
    merged_labels  = local.all_labels
    try_found      = local.safe_value
    try_missing    = local.missing_val
    can_result     = local.has_nested
    subnet_cidrs   = local.subnet_cidrs
    fqdns          = local.fqdns
    zone_map       = local.zone_map
  }
}
EOF

terraform plan
```

---

## 🔥 Scenario Challenge

### Scenario 1: Multi-Environment Cluster Configuration (TVS OTA)

**Problem:** You need to define a variable structure that supports deploying GKE clusters across 3 environments with different configurations. Production needs HA (multi-zone), staging needs cost optimization (preemptible), dev needs minimal resources.

```bash
cat > challenge_01.tf << 'EOF'
variable "environments" {
  type = map(object({
    cluster_name = string
    region       = string
    zones        = list(string)
    node_pools = list(object({
      name         = string
      machine_type = string
      node_count   = number
      preemptible  = bool
      disk_size_gb = number
      labels       = map(string)
    }))
    maintenance_window = optional(object({
      start_time = string
      end_time   = string
      recurrence = string
    }))
    enable_hpa       = bool
    enable_vpa       = bool
    max_pods_per_node = number
  }))
  
  default = {
    dev = {
      cluster_name = "tvs-ota-dev"
      region       = "asia-south1"
      zones        = ["asia-south1-a"]
      node_pools = [{
        name         = "default"
        machine_type = "e2-medium"
        node_count   = 2
        preemptible  = true
        disk_size_gb = 50
        labels       = { env = "dev", cost = "low" }
      }]
      maintenance_window = null
      enable_hpa         = false
      enable_vpa         = false
      max_pods_per_node  = 32
    }
    staging = {
      cluster_name = "tvs-ota-staging"
      region       = "asia-south1"
      zones        = ["asia-south1-a", "asia-south1-b"]
      node_pools = [{
        name         = "default"
        machine_type = "e2-standard-4"
        node_count   = 3
        preemptible  = true
        disk_size_gb = 100
        labels       = { env = "staging", cost = "medium" }
      }]
      maintenance_window = null
      enable_hpa         = true
      enable_vpa         = false
      max_pods_per_node  = 64
    }
    prod = {
      cluster_name = "tvs-ota-prod"
      region       = "asia-south1"
      zones        = ["asia-south1-a", "asia-south1-b", "asia-south1-c"]
      node_pools = [
        {
          name         = "system"
          machine_type = "n2-standard-4"
          node_count   = 3
          preemptible  = false
          disk_size_gb = 100
          labels       = { env = "prod", pool = "system" }
        },
        {
          name         = "workload"
          machine_type = "n2-standard-8"
          node_count   = 5
          preemptible  = false
          disk_size_gb = 200
          labels       = { env = "prod", pool = "workload" }
        }
      ]
      maintenance_window = {
        start_time = "2024-01-01T02:00:00Z"
        end_time   = "2024-01-01T06:00:00Z"
        recurrence = "FREQ=WEEKLY;BYDAY=SA"
      }
      enable_hpa        = true
      enable_vpa        = true
      max_pods_per_node = 110
    }
  }
}

locals {
  # Flatten all node pools across environments
  all_node_pools = flatten([
    for env_name, env_config in var.environments : [
      for pool in env_config.node_pools : {
        env_name     = env_name
        pool_name    = pool.name
        full_name    = "${env_config.cluster_name}-${pool.name}"
        machine_type = pool.machine_type
        node_count   = pool.node_count
        preemptible  = pool.preemptible
        disk_size_gb = pool.disk_size_gb
        labels       = pool.labels
      }
    ]
  ])
  
  # Only production environments
  prod_clusters = {
    for name, config in var.environments : name => config
    if !config.node_pools[0].preemptible
  }
  
  # Total node count across all environments
  total_nodes = sum([
    for pool in local.all_node_pools : pool.node_count
  ])
}

output "all_node_pools" {
  value = local.all_node_pools[*].full_name
}

output "prod_clusters" {
  value = keys(local.prod_clusters)
}

output "total_nodes" {
  value = local.total_nodes
}
EOF

terraform init && terraform plan
```

### Scenario 2: Dynamic Security Rules (EB CI/CT)

**Problem:** Generate firewall rules dynamically based on environment. Dev gets permissive rules, Prod gets strict rules with specific CIDR ranges.

```bash
cat > challenge_02.tf << 'EOF'
variable "security_config" {
  type = object({
    environment  = string
    admin_cidrs  = list(string)
    app_cidrs    = list(string)
    services = map(object({
      port     = number
      protocol = string
      public   = bool
    }))
  })
  default = {
    environment = "prod"
    admin_cidrs = ["10.0.0.0/8", "172.16.0.0/12"]
    app_cidrs   = ["10.0.0.0/8"]
    services = {
      jenkins = { port = 8080, protocol = "tcp", public = false }
      sonar   = { port = 9000, protocol = "tcp", public = false }
      nexus   = { port = 8081, protocol = "tcp", public = false }
      grafana = { port = 3000, protocol = "tcp", public = true }
    }
  }
}

locals {
  # Generate firewall rules based on service config
  firewall_rules = [
    for svc_name, svc_config in var.security_config.services : {
      name        = "allow-${svc_name}"
      port        = svc_config.port
      protocol    = svc_config.protocol
      source_ranges = svc_config.public ? ["0.0.0.0/0"] : var.security_config.admin_cidrs
      description = "${svc_name} access (${svc_config.public ? "public" : "internal"})"
    }
  ]
  
  # Separate public vs private services
  public_services = {
    for name, svc in var.security_config.services : name => svc
    if svc.public
  }
  
  private_services = {
    for name, svc in var.security_config.services : name => svc
    if !svc.public
  }
}

output "generated_rules" {
  value = local.firewall_rules
}

output "public_services" {
  value = keys(local.public_services)
}

output "private_services" {
  value = keys(local.private_services)
}
EOF

terraform plan
```

---

## 🏗️ Real Project Reference

### TVS OTA — Multi-Cluster Variable Design

```hcl
# Real pattern used: Complex object variables for multi-cluster deployment
# File: terraform/environments/prod.tfvars

project_id   = "tvs-ota-prod-001"
environment  = "prod"
region       = "asia-south1"

clusters = {
  ota-primary = {
    zone_count    = 3
    node_count    = 5
    machine_type  = "n2-standard-8"
    disk_size     = 200
    auto_scaling  = { min = 3, max = 10 }
    taints        = []
    gpu_enabled   = false
  }
  ota-ml-inference = {
    zone_count    = 2
    node_count    = 3
    machine_type  = "n2-highmem-4"
    disk_size     = 500
    auto_scaling  = { min = 2, max = 6 }
    taints        = [{ key = "workload", value = "ml", effect = "NoSchedule" }]
    gpu_enabled   = true
  }
}

# Conditional: Only prod gets dedicated monitoring cluster
monitoring_cluster = {
  enabled      = true
  machine_type = "e2-standard-4"
  node_count   = 3
}
```

### EB CI/CT — Dynamic Blocks for Pipeline Agents

```hcl
# Real pattern used: Dynamic blocks for security group rules per agent type
# File: modules/ci-agent/variables.tf

variable "agent_types" {
  type = map(object({
    instance_type = string
    count         = number
    tools         = list(string)
    ports         = list(number)
    schedule      = string  # cron for scaling
  }))
  
  validation {
    condition = alltrue([
      for name, agent in var.agent_types :
      can(regex("^[a-z]+-agent$", name))
    ])
    error_message = "Agent type names must match pattern: {type}-agent"
  }
}

# Dynamic block usage in security group
# dynamic "ingress" {
#   for_each = { for agent_name, config in var.agent_types : 
#     agent_name => config.ports }
#   content {
#     from_port   = ingress.value[0]
#     to_port     = ingress.value[length(ingress.value) - 1]
#     protocol    = "tcp"
#     cidr_blocks = var.internal_cidrs
#     description = "Access for ${ingress.key}"
#   }
# }
```

---

## 📋 Best Practices

### Variable Design Patterns

| Pattern | When | Example |
|---------|------|---------|
| **Flat variables** | Simple, few params | `var.region`, `var.env` |
| **Object variable** | Related params group | `var.cluster_config{}` |
| **Map of objects** | Multiple similar resources | `var.environments{}` |
| **Validation rules** | Prevent invalid inputs | `contains()`, `regex()` |
| **Optional attributes** | Not all envs need all fields | `optional(type, default)` |

### Anti-Patterns to Avoid

```hcl
# ❌ BAD: Too many individual variables
variable "cluster_name" {}
variable "cluster_region" {}
variable "cluster_zone" {}
variable "cluster_node_count" {}
variable "cluster_machine_type" {}

# ✅ GOOD: Grouped as object
variable "cluster" {
  type = object({
    name         = string
    region       = string
    zone         = string
    node_count   = number
    machine_type = string
  })
}

# ❌ BAD: Hardcoded conditional logic spread everywhere
resource "x" "y" {
  count = var.environment == "prod" ? 1 : 0
}
resource "a" "b" {
  count = var.environment == "prod" ? 1 : 0
}

# ✅ GOOD: Single local, reference everywhere
locals {
  is_prod = var.environment == "prod"
}
resource "x" "y" {
  count = local.is_prod ? 1 : 0
}

# ❌ BAD: Using variables for computed values
variable "full_name" {
  default = "" # Caller has to compute this
}

# ✅ GOOD: Use locals for derived values
locals {
  full_name = "${var.project}-${var.environment}-${var.component}"
}
```

### Security Best Practices

1. **Always validate** — Never trust input without validation rules
2. **Mark sensitive** — Passwords, API keys, certificates
3. **Never default sensitive values** — Force explicit input
4. **Use separate tfvars per env** — Don't mix prod/dev values
5. **Encrypt state** — `sensitive = true` doesn't encrypt state file

---

## 🎤 Interview Q&A

### Q1: What's the difference between variables and locals in Terraform?

**English:**
Variables are INPUT parameters — configurable by callers (users, modules, CI/CD). Locals are COMPUTED values — internal to the module, not exposed externally. Use variables when the value should change per deployment, use locals for derived/computed values that reduce repetition.

**தமிழ்:**
Variables என்பது INPUT parameters — callers (users, modules, CI/CD) மூலம் configurable. Locals என்பது COMPUTED values — module-க்கு internal, வெளியே expose ஆகாது. Deployment-க்கு மாற வேண்டிய value-க்கு variables பயன்படுத்துங்கள், repetition குறைக்க derived values-க்கு locals பயன்படுத்துங்கள்.

---

### Q2: Explain variable precedence in Terraform.

**English:**
Precedence from highest to lowest: CLI `-var` flags → `-var-file` flags → `*.auto.tfvars` (alphabetical) → `terraform.tfvars` → `TF_VAR_` environment variables → variable `default` block. This allows environment-specific overrides while maintaining sensible defaults.

**Architecture insight:** In our CI/CD pipelines, we use `terraform.tfvars` for common values, environment-specific `*.auto.tfvars` for env differences, and CLI `-var` only for secrets injected from vault.

**தமிழ்:**
முன்னுரிமை highest to lowest: CLI `-var` → `-var-file` → `*.auto.tfvars` → `terraform.tfvars` → `TF_VAR_` env vars → `default`. இது sensible defaults-ஐ பராமரிக்கும் அதே நேரத்தில் environment-specific overrides-ஐ அனுமதிக்கிறது.

---

### Q3: When would you use dynamic blocks vs count/for_each?

**English:**
- **count/for_each** — Creates multiple RESOURCE INSTANCES (separate resources)
- **dynamic blocks** — Creates multiple NESTED BLOCKS within a SINGLE resource

Use dynamic blocks when a resource needs repeated configuration blocks (like security group rules, disk attachments, DNS records). Use for_each when you need multiple separate resources.

Example: A firewall resource with 10 rules = dynamic block. Ten separate firewall resources = for_each.

**தமிழ்:**
- **count/for_each** — பல RESOURCE INSTANCES உருவாக்கும் (தனித்தனி resources)
- **dynamic blocks** — ஒரே SINGLE resource-க்குள் பல NESTED BLOCKS உருவாக்கும்

ஒரு resource-க்கு repeated config blocks தேவைப்படும்போது dynamic blocks பயன்படுத்துங்கள். பல தனி resources தேவைப்படும்போது for_each பயன்படுத்துங்கள்.

---

### Q4: How do you handle sensitive variables securely?

**English:**
Multi-layer approach:
1. Mark `sensitive = true` — hides from CLI output
2. Never commit `.tfvars` with secrets to git (`.gitignore`)
3. Use external secret stores (HashiCorp Vault, AWS Secrets Manager)
4. Encrypt state backend (S3 with KMS, GCS with CMEK)
5. Use `TF_VAR_` env vars injected by CI/CD from vault
6. Short-lived credentials (workload identity, OIDC) over static keys

**Critical:** `sensitive = true` only masks output — state file still has plaintext! State encryption is mandatory.

**தமிழ்:**
Multi-layer அணுகுமுறை:
1. `sensitive = true` — CLI output-ல் மறைக்கும்
2. Secrets உள்ள `.tfvars`-ஐ git-ல் commit செய்ய வேண்டாம்
3. External secret stores பயன்படுத்துங்கள் (Vault, Secrets Manager)
4. State backend-ஐ encrypt செய்யுங்கள்
5. CI/CD vault-ல் இருந்து inject செய்யப்படும் env vars பயன்படுத்துங்கள்
6. Static keys-க்கு பதிலாக short-lived credentials பயன்படுத்துங்கள்

---

### Q5: How do you design variables for a multi-environment Terraform project?

**English:**
Architecture pattern I follow:
1. **Base module** — All resources with variables for differences
2. **Object variables** — Group related config (`var.cluster_config`)
3. **Map of objects** — Multiple instances (`var.environments["prod"]`)
4. **Validation** — Catch errors early with `validation {}` blocks
5. **Separate tfvars** — `dev.tfvars`, `prod.tfvars` per environment
6. **Locals for derivation** — Compute names, tags, CIDRs from inputs
7. **optional() attributes** — Not all environments need all fields

Real example from TVS OTA: One module, three tfvars files, variables with complex objects defining cluster topology per environment.

**தமிழ்:**
நான் பின்பற்றும் Architecture pattern:
1. Base module — வேறுபாடுகளுக்கு variables கொண்ட அனைத்து resources
2. Object variables — தொடர்புடைய config-ஐ group செய்யுங்கள்
3. Map of objects — பல instances
4. Validation — errors-ஐ ஆரம்பத்திலேயே கண்டறியுங்கள்
5. தனி tfvars — environment-க்கு ஒன்று
6. Locals — inputs-ல் இருந்து names, tags compute செய்யுங்கள்
7. optional() attributes — அனைத்து environments-க்கும் அனைத்து fields தேவையில்லை

---

### Q6: What is `try()` and `can()` and when do you use them?

**English:**
- `try(expr1, expr2, ...)` — Returns first expression that doesn't error. Used for safe access to nested attributes or providing fallbacks.
- `can(expr)` — Returns `true` if expression evaluates without error, `false` otherwise. Used in conditions/validations.

```hcl
# try — safe nested access
db_port = try(var.config.database.port, 5432)

# can — validation conditions  
validation {
  condition = can(regex("^[a-z]+$", var.name))
  error_message = "Name must be lowercase letters only."
}
```

**தமிழ்:**
- `try()` — Error இல்லாத முதல் expression-ஐ return செய்யும். Nested attributes-ஐ safe-ஆக access செய்ய பயன்படும்.
- `can()` — Expression error இல்லாமல் evaluate ஆனால் `true`, இல்லையெனில் `false`. Validations-ல் பயன்படும்.

---

### Q7: How do you avoid variable sprawl in large Terraform projects?

**English:**
Strategies for managing variable complexity:
1. **Group into objects** — 10 flat vars become 1 object var
2. **Use modules** — Each module owns its variables
3. **Locals for computation** — Don't expose internal logic as variables
4. **Variable files per concern** — `network.tfvars`, `compute.tfvars`
5. **Convention over configuration** — Derive names from `project + env + component`
6. **optional() with defaults** — Reduce required inputs

Rule of thumb: If a variable isn't needed by the CALLER, it should be a local.

**தமிழ்:**
Variable complexity manage செய்யும் strategies:
1. Objects-ஆக group செய்யுங்கள் — 10 flat vars → 1 object var
2. Modules பயன்படுத்துங்கள் — ஒவ்வொரு module-ம் தன் variables-ஐ own செய்யும்
3. Internal logic-ஐ variables-ஆக expose செய்ய வேண்டாம்
4. Concern-க்கு variable files பிரிக்கவும்
5. Convention over configuration — names derive செய்யுங்கள்
6. optional() with defaults — required inputs குறைக்கவும்

---

## ✅ Self-Check

After completing this module, verify you can answer:

- [ ] List all Terraform variable types and when to use each
- [ ] Recite variable precedence order without looking
- [ ] Explain when to use variable vs local vs data source
- [ ] Write a dynamic block from memory
- [ ] Use `try()`, `can()`, `lookup()`, `merge()`, `flatten()` correctly
- [ ] Design a complex object variable for multi-env deployment
- [ ] Implement validation rules with regex and conditions
- [ ] Explain why `sensitive = true` alone isn't sufficient security
- [ ] Describe your variable design pattern for large projects
- [ ] Use for expressions with filtering (if clause)

---

## 🔑 Key Takeaways

1. **Variables are your API** — Design them like you'd design an API: clear, validated, documented
2. **Locals reduce repetition** — Compute once, use everywhere
3. **Dynamic blocks are powerful** — But use sparingly, readability matters
4. **Validate early** — Catch errors before `apply`, not after
5. **Precedence matters** — Know the order for debugging override issues
6. **Type constraints prevent bugs** — `object({})` is better than `any`
7. **Functions are your toolkit** — `merge`, `lookup`, `flatten`, `try` solve 80% of problems
