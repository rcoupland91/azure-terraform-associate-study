# Lifecycle Blocks

## Learning Objectives
- Understand all Terraform lifecycle meta-arguments and when to use each.
- Learn how to use `depends_on` for explicit dependency management.
- Learn how to prevent accidental resource destruction.
- Master `create_before_destroy` for zero-downtime updates.
- Control resource replacement behavior with lifecycle rules.

---

## 1. What are Lifecycle Blocks?

The `lifecycle` block is a **meta-argument** that controls how Terraform creates, updates, and destroys resources. It's placed inside a resource block.

**General syntax:**
```hcl
resource "azurerm_linux_virtual_machine" "web" {
  Name           = "web-vm-01"
  size = "Standard_B1s"
  
  lifecycle {
    # Lifecycle rules go here
  }
}
```

---

## 2. Lifecycle Meta-Arguments Overview

Terraform provides four lifecycle rules:

1. **`prevent_destroy`** - Prevents resource destruction
2. **`create_before_destroy`** - Creates new resource before destroying old
3. **`ignore_changes`** - Ignores changes to specific attributes
4. **`replace_triggered_by`** - Forces replacement when referenced resources change

Additionally, Terraform provides the **`depends_on`** meta-argument for explicit dependency management, which is closely related to lifecycle management.

---

## 3. `depends_on` - Explicit Dependencies

### Purpose

The `depends_on` meta-argument creates **explicit dependencies** between resources when Terraform cannot automatically infer the dependency from resource references. It ensures resources are created, updated, or destroyed in the correct order.

### Implicit vs Explicit Dependencies

**Implicit Dependencies** (automatic):
```hcl
resource "azurerm_key_vault" "main" {
  name                        = "${local.location_abbreviated}-${var.subscription}-${var.environment}-${each.value.name}-kv"
}

resource "azurerm_key_vault_access_policy" "user_access" {

  key_vault_id = azurerm_key_vault.main.id ## Implicit Dependency
}
```

**Explicit Dependencies** (using `depends_on`):
```hcl
resource "azurerm_private_endpoint" "privateendpoint_aifoundry" {
  depends_on = [
    azurerm_private_endpoint.pe_aisearch,
    azapi_resource.ai_foundry
  ]
}
```

### When to Use `depends_on`

Use `depends_on` when:

1. **No direct reference exists:**
   ```hcl
   # Resource A must exist before Resource B, but B doesn't reference A
   resource "azapi_resource" "ai_foundry" {
   }
   
   resource "azurerm_private_endpoint" "pe_aifoundry" {
  depends_on = [
    azurerm_private_endpoint.pe_aisearch,
    azapi_resource.ai_foundry
  ]}
   ```

2. **Side effects or external dependencies:**
   ```hcl
   resource "azapi_resource" "ai_foundry" {}
   ```
   ```hcl    
   resource "time_sleep" "wait_project_identities" {
   depends_on = [
    azapi_resource.ai_foundry
  ]
   create_duration = "10s"
}
   ```

### Common Use Cases

#### Use Case 1: A VM must wait for a Log Analytics Workspace to finish provisioning

```hcl

resource "azurerm_log_analytics_workspace" "log-analytics" {
  name                = "log-analytics1"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "myvm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  depends_on = [
    azurerm_log_analytics_workspace.log-analytics
  ]
}


```

#### Use Case 2: Ensure a secret is created only after the access policy exists

```hcl
resource "azurerm_key_vault" "kv" {
  name                = "my-kv"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku_name            = "standard"
}

resource "azurerm_key_vault_access_policy" "ap" {
  key_vault_id = azurerm_key_vault.kv.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = data.azurerm_client_config.current.object_id

  secret_permissions = ["Set", "Get"]
}

resource "azurerm_key_vault_secret" "secret" {
  name         = "db-password"
  value        = "supersecret"
  key_vault_id = azurerm_key_vault.kv.id

  depends_on = [azurerm_key_vault_access_policy.ap]
}

```

#### Use Case 3: A module that deploys an AKS cluster must wait for a Private DNS zone and a Key Vault

```hcl
resource "azurerm_private_dns_zone" "aks_dns" {
  name                = "privatelink.westeurope.azmk8s.io"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_key_vault" "kv" {
  name                = "my-kv"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku_name            = "standard"
}

module "aks" {
  source = "./modules/aks"

  cluster_name = "my-aks"
  location     = azurerm_resource_group.rg.location
  rg_name      = azurerm_resource_group.rg.name

  depends_on = [
    azurerm_private_dns_zone.aks_dns,
    azurerm_key_vault.kv
  ]
}

```

#### Use Case 4: Data Source Dependencies

```hcl
resource "azurerm_subnet" "app" {
  name                 = "app"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}

data "azurerm_subnet" "app_data" {
  name                 = azurerm_subnet.app.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  resource_group_name  = azurerm_resource_group.rg.name

  depends_on = [azurerm_subnet.app]
}

```

### Important Notes

1. **`depends_on` affects order, not relationships:**
   - It only controls creation/destruction order
   - It doesn't create a data dependency
   - Resources can still reference each other without `depends_on` if there's a direct reference

2. **Use sparingly:**
   - Terraform usually infers dependencies automatically
   - Only use when automatic inference isn't sufficient
   - Overuse can make configurations harder to understand

3. **Works with modules:**
   ```hcl
   module "app" {
     source = "./modules/app"
     
     depends_on = [
       module.database,
       module.cache
     ]
   }
   ```

4. **Works with data sources:**
   ```hcl
resource "azurerm_subnet" "app" {
  name                 = "app"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}

data "azurerm_subnet" "app_data" {
  name                 = azurerm_subnet.app.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  resource_group_name  = azurerm_resource_group.rg.name

  depends_on = [azurerm_subnet.app]
}
   ```

### Combining `depends_on` with Lifecycle Rules

```hcl
resource "azurerm_key_vault" "kv" {
  name                = "my-kv"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  lifecycle {
    prevent_destroy = true
  }
}

resource "azurerm_virtual_machine_extension" "ext" {
  name                 = "config"
  virtual_machine_id   = azurerm_linux_virtual_machine.vm.id
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.1"

  depends_on = [
    azurerm_key_vault.kv
  ]
}

```

### Real-World Example: Complete Dependency Chain

```hcl
# 1. Virtual Network (no dependencies)
resource "azurerm_virtual_network" "vnet" {
  name                = "main-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# 2. Subnets (depend on Vnet)
resource "azurerm_subnet" "public" {
  name                 = "public-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "private" {
  name                 = "private-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}

# 3. Public IP (depends on nothing)
resource "azurerm_public_ip" "nat_pip" {
  name                = "nat-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
}


# 4. NAT Gateway (depends on public subnet + PIP)
resource "azurerm_nat_gateway" "nat" {
  name                = "main-nat"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  public_ip_address_ids = [
    azurerm_public_ip.nat_pip.id
  ]

  depends_on = [
    azurerm_public_ip.nat_pip,
    azurerm_subnet.public
  ]
}

resource "azurerm_subnet_nat_gateway_association" "private_nat" {
  subnet_id      = azurerm_subnet.private.id
  nat_gateway_id = azurerm_nat_gateway.nat.id

  depends_on = [
    azurerm_nat_gateway.nat
  ]
}

# 5. Route Tables (depend on gateways + subnets)
resource "azurerm_route_table" "public" {
  name                = "public-rt"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  route {
    name           = "internet"
    address_prefix = "0.0.0.0/0"
    next_hop_type  = "Internet"
  }

  depends_on = [
    azurerm_subnet.public
  ]
}

resource "azurerm_route_table" "private" {
  name                = "private-rt"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  route {
    name                   = "nat"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = azurerm_nat_gateway.nat.private_ip_address
  }

  depends_on = [
    azurerm_nat_gateway.nat,
    azurerm_subnet.private
  ]
}

# 6. VM (depends on all networking)
resource "azurerm_network_interface" "nic" {
  name                = "vm-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.public.id
    private_ip_address_allocation = "Dynamic"
  }

  depends_on = [
    azurerm_route_table.public,
    azurerm_public_ip.nat_pip
  ]
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  depends_on = [
    azurerm_network_interface.nic
  ]
}
```

---

## 4. `prevent_destroy`

### Purpose
Prevents Terraform from destroying the resource, even when running `terraform destroy` or when the resource is removed from configuration.

### Syntax
```hcl
resource "azurerm_key_vault" "kv" {
  name                = "my-kv"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  lifecycle {
    prevent_destroy = true
  }
}
```

### Use Cases
- **Critical production resources** (databases, state containers)
- **Resources that can't be recreated** (unique names, historical data)
- **Safety net** for important infrastructure

### Behavior

```hcl
# With prevent_destroy = true
terraform destroy
# Error: Instance cannot be destroyed because prevent_destroy is set to true
```

### Overriding `prevent_destroy`

To destroy a protected resource, you must:
1. Remove `prevent_destroy = true` from the configuration
2. Run `terraform apply` to update the lifecycle rule
3. Then run `terraform destroy`

**Or** use `terraform destroy -target` (this still respects `prevent_destroy` - you'll get an error).

**Actually, the only way is to remove the lifecycle rule first.**

### Example: Protecting State Bucket

```hcl
resource "azurerm_key_vault" "kv" {
  name                = "my-kv"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  lifecycle {
    prevent_destroy = true ## Dont ever delete this KV
  }
}
```

---

## 5. `create_before_destroy`

### Purpose
Creates the new resource **before** destroying the old one. Essential for zero-downtime updates.

### Syntax
```hcl
resource "azurerm_network_interface" "nic" {
  name                = "app-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.new_subnet.id
    private_ip_address_allocation = "Dynamic"
  }

  lifecycle {
    create_before_destroy = true
  }

  depends_on = [
    azurerm_subnet.new_subnet
  ]
}
```

### Default Behavior (without `create_before_destroy`)

```hcl
# Changing azurerm_network_interface name from app-nic to app-nic1
# Default: Destroy old → Create new
# Result: Downtime during transition
```

### With `create_before_destroy = true`

```hcl
# Changing azurerm_network_interface name from app-nic to app-nic1
# With rule: Create new → Destroy old
# Result: Both exist temporarily, no downtime
```

### Use Cases
- **Load balancer targets** - Keep old instance until new one is healthy
- **Database instances** - Ensure new instance is ready before destroying old
- **Any resource where downtime is unacceptable**

### Important Considerations

1. **Resource name conflicts:**
```hcl

resource "azurerm_network_interface" "nic" {
  name                = "app-nic-${random_id.suffix.hex}" # If name is unique, you may need to make it dynamic
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.new_subnet.id
    private_ip_address_allocation = "Dynamic"
  }

  lifecycle {
    create_before_destroy = true
  }

  depends_on = [
    azurerm_subnet.new_subnet
  ]
}
```

2. **State file size:**
   - Both resources exist in state temporarily
   - Can cause issues if resource names are fixed

3. **Cost implications:**
   - Two resources exist briefly
   - May incur double cost during transition

### Example: Network Interface (NIC) replacement without downtime

```hcl
resource "azurerm_network_interface" "nic" {
  name                = "app-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.app.id
    private_ip_address_allocation = "Dynamic"
  }

  lifecycle {
    create_before_destroy = true
  }
}

```

---

## 6. `ignore_changes`

### Purpose
Tells Terraform to ignore changes to specific attributes during `terraform plan` and `terraform apply`.

### Syntax
```hcl
resource "azurerm_monitor_diagnostic_setting" "ai_foundry_diag" {

  name               = "diag-log"
  target_resource_id = azapi_resource.ai_foundry[each.value.name].id
  log_analytics_workspace_id = var.diag_log_workspace

  dynamic "enabled_log" {
    for_each = try(data.azurerm_monitor_diagnostic_categories.foundry_service[each.value.name].log_category_types, [])
    content {
      category = enabled_log.value
    }
  }

  depends_on = [
    azapi_resource.ai_foundry,
  ]

  lifecycle {
    ignore_changes = [
      # Ignore changes to enabled_log
      enabled_log,
    ]
  }
}
```

### Ignoring All Changes to an Attribute List

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  # ...
  
  lifecycle {
    ignore_changes = [
      tags,                    # All tags ignored
      user_data,               # All user_data ignored
    ]
  }
}
```

### Ignoring Specific Attributes

```hcl
resource "azurerm_service_plan" "fnapp" {
  name                = "app-sp"
  resource_group_name = "test-rg
  location            = var.location
  os_type             = each.value.os_type
  sku_name            = each.value.sku_name

  tags = local.tags
  
  lifecycle {
    ignore_changes = [
      tags["Environment"],       # Only ignore this specific tag
    ]
  }
}
```

**Note:** Attribute-level ignoring (like `tags["Environment"]`) may not work in all Terraform versions. Generally, `ignore_changes = [tags]` ignores all tags.

### Use Cases

1. **External modifications:**
   ```hcl
   # Someone changes tags in Azure Portal
   # Terraform won't try to revert them
   lifecycle {
     ignore_changes = [tags]
   }
   ```

2. **Auto-scaling adjustments:**
   ```hcl
  resource "azurerm_linux_virtual_machine_scale_set" "shared_agent_scale_set" {
  name                = "linus-scale-set"
  instances           = var.shared_agents_count #1

  lifecycle {
    ignore_changes = [
      tags["__AzureDevOpsElasticPool"],
      tags["__AzureDevOpsElasticPoolTimeStamp"],
      instances, # number of isntances configured under AzDO settings
    ]
  }
   }
   ```

### Combining with `replace_triggered_by`

```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "app-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  lifecycle {
    create_before_destroy = true
    replace_triggered_by  = [
      azurerm_network_interface.nic
    ]
  }
}

```

---

## 7. `replace_triggered_by`

### Purpose
Forces resource replacement when referenced resources or their attributes change.

### Syntax
```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "app-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  lifecycle {
    create_before_destroy = true
    replace_triggered_by  = [
      azurerm_network_interface.nic
    ]
  }
}

```

### Use Cases

1. **VM and NIC dependencies:**
   ```hcl
resource "azurerm_network_interface" "nic" {
  name                = "web-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  lifecycle {
    create_before_destroy = true
    replace_triggered_by  = [
      azurerm_network_interface.nic
    ]
  }
}

   ```

2. **Subnet or VNet changes that require NIC replacement:**
   ```hcl
resource "azurerm_subnet" "app" {
  name                 = "app-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_network_interface" "nic" {
  # ...
  lifecycle {
    replace_triggered_by = [
      azurerm_subnet.app
    ]
  }
}

   ```

3. **Public IP changes that require dependent resources to be replaced:**
   ```hcl
resource "azurerm_public_ip" "pip" {
  name                = "web-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
}

resource "azurerm_lb" "lb" {
  name                = "web-lb"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  frontend_ip_configuration {
    name                 = "public"
    public_ip_address_id = azurerm_public_ip.pip.id
  }

  lifecycle {
    create_before_destroy = true
    replace_triggered_by  = [
      azurerm_public_ip.pip
    ]
  }
}

   ```

### Important Notes

- **Only accepts resource addresses**, not arbitrary expressions
- **Must reference resources or their attributes** (e.g., `azurerm_key_vault_key.example.id`)
- **Triggers replacement**, not just update

---

## 8. Combining Lifecycle Rules

You can use multiple lifecycle rules together:

```hcl
resource "azurerm_linux_virtual_machine" "critical" {
  name                = "critical-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = var.image_version   # e.g., "latest" or a pinned version
  }

  tags = {
    Environment = "prod"
  }

  lifecycle {
    prevent_destroy       = true
    create_before_destroy = true
    ignore_changes        = [tags]
    replace_triggered_by  = [
      var.image_version
    ]
  }
}

```

**Note:** `replace_triggered_by` cannot accept variables directly. You'd need to reference a resource that changes when the variable changes.

---

## 9. Real-World Examples

### Example 1: Production Database (Azure SQL)

```hcl
resource "azurerm_mssql_server" "prod" {
  name                         = "prod-sql-server"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.admin_password

  lifecycle {
    prevent_destroy = true
    ignore_changes = [
      tags,
      identity,                     # Azure may add system-assigned identity
    ]
  }
}

resource "azurerm_mssql_database" "production" {
  name           = "prod-db"
  server_id      = azurerm_mssql_server.prod.id
  sku_name       = "S2"
  max_size_gb    = 250

  lifecycle {
    prevent_destroy = true
    ignore_changes = [
      max_size_gb,                  # Azure auto-scales storage
      zone_redundant,               # Azure may toggle this based on region
    ]
  }
}

```

### Example 2: Load Balancer Target (Azure VM + Backend Pool)

```hcl
resource "azurerm_linux_virtual_machine" "app" {
  name                = "app-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  lifecycle {
    create_before_destroy = true
  }
}

resource "azurerm_lb_backend_address_pool_address" "app" {
  name                    = "app-backend"
  backend_address_pool_id = azurerm_lb_backend_address_pool.app.id
  virtual_network_id       = azurerm_virtual_network.vnet.id
  ip_address               = azurerm_network_interface.nic.private_ip_address

  lifecycle {
    create_before_destroy = true
  }
}

```

### Example 3: Auto-Managed Tags (Azure modifies tags frequently)

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  tags = {
    Name        = "web-server"
    Environment = "prod"
    ManagedBy   = "terraform"
    CostCenter  = "engineering"
  }

  lifecycle {
    ignore_changes = [
      tags["CostCenter"],           # Allow external tag management
      tags["resourceUsage"],        # Azure auto-applies this
      identity,                     # Azure may add system identity
    ]
  }
}

```

---

## 10. Exam-Style Practice Questions

### Question 1
Which lifecycle rule prevents Terraform from destroying a resource?
A) `create_before_destroy`
B) `prevent_destroy`
C) `ignore_changes`
D) `replace_triggered_by`

<details>
<summary>Show Answer</summary>
Answer: **B** - `prevent_destroy = true` prevents resource destruction.
</details>

---

### Question 2
You need to update an Virtual Machine instance with zero downtime. Which lifecycle rule should you use?
A) `prevent_destroy = true`
B) `create_before_destroy = true`
C) `ignore_changes = [instance_type]`
D) `replace_triggered_by = [azurerm_linux_virtual_machine.example.id]`

<details>
<summary>Show Answer</summary>
Answer: **B** - `create_before_destroy = true` creates the new resource before destroying the old, ensuring zero downtime.
</details>

---

### Question 3
You want Terraform to ignore changes made to tags in the Azure Portal. Which rule should you use?
A) `prevent_destroy = true`
B) `create_before_destroy = true`
C) `ignore_changes = [tags]`
D) No lifecycle rule needed

<details>
<summary>Show Answer</summary>
Answer: **C** - `ignore_changes = [tags]` tells Terraform to ignore tag modifications.
</details>

---

### Question 4
What happens when you try to destroy a resource with `prevent_destroy = true`?
A) Resource is destroyed after confirmation
B) Terraform prompts for confirmation
C) Terraform shows an error and stops
D) Resource is removed from state but not destroyed

<details>
<summary>Show Answer</summary>
Answer: **C** - Terraform will error and stop, preventing destruction of the protected resource.
</details>

---

### Question 5
Which lifecycle rule forces a resource to be recreated when another resource changes?
A) `replace_triggered_by`
B) `create_before_destroy`
C) `ignore_changes`
D) `prevent_destroy`

<details>
<summary>Show Answer</summary>
Answer: **A** - `replace_triggered_by` forces replacement when referenced resources change.
</details>

---

### Question 6
When should you use `depends_on`?
A) Always, to ensure correct resource ordering
B) Only when Terraform cannot automatically infer dependencies
C) Never, Terraform always infers dependencies correctly
D) Only for data sources

<details>
<summary>Show Answer</summary>
Answer: **B** - Use `depends_on` when Terraform cannot automatically infer dependencies from resource references, such as when resources must be created in order but don't directly reference each other.
</details>

---

### Question 7
What is the difference between implicit and explicit dependencies?
A) Implicit dependencies use `depends_on`, explicit don't
B) Explicit dependencies use `depends_on`, implicit are inferred from references
C) There is no difference
D) Implicit dependencies are faster

<details>
<summary>Show Answer</summary>
Answer: **B** - Explicit dependencies use the `depends_on` meta-argument. Implicit dependencies are automatically inferred by Terraform when one resource references another.
</details>

---

## 11. Decision Guide

**When to use `prevent_destroy`:**
- Critical resources (databases, state containers etc)
- Resources that can't be recreated
- Production safety net

**When to use `create_before_destroy`:**
- Zero-downtime updates required
- Load balancer targets
- Resources serving traffic

**When to use `ignore_changes`:**
- Attributes modified externally (console, scripts)
- Auto-scaling managed attributes
- Tags managed by other systems

**When to use `replace_triggered_by`:**
- Need to force recreation on dependency changes
- Launch template updates should recreate instances
- Configuration changes require full replacement

**When to use `depends_on`:**
- Resources must be created in order but don't reference each other
- Side effects or external dependencies
- Multiple resources must be ready before another can be created
- Data sources need resources to exist first

---

## 12. Key Takeaways

- **`depends_on`**: Creates explicit dependencies when Terraform cannot infer them automatically. Use when resources must be created in a specific order but don't directly reference each other.
- **`prevent_destroy`**: Protects resources from accidental destruction. Must be removed before destroying.
- **`create_before_destroy`**: Creates new resource before destroying old (zero-downtime updates). Watch for name conflicts.
- **`ignore_changes`**: Tells Terraform to ignore changes to specific attributes (useful for external modifications).
- **`replace_triggered_by`**: Forces resource replacement when referenced resources change. Only accepts resource addresses.
- **All rules can be combined** in a single `lifecycle` block.
- **`prevent_destroy` takes precedence** - even `terraform destroy -target` will fail.
- **Use lifecycle rules judiciously** - they can mask configuration drift and cause unexpected behavior.
- **Implicit vs Explicit**: Terraform usually infers dependencies automatically. Use `depends_on` only when necessary.

---

## References

- [Terraform Lifecycle Meta-Arguments](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)
- [Terraform depends_on Meta-Argument](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on)
- [Resource Behavior](https://developer.hashicorp.com/terraform/language/resources/behavior)

