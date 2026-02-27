# 14 - Custom Validation Rules

## Learning Objectives
- Understand how to validate variable inputs using `validation` blocks.
- Learn how to use `precondition` and `postcondition` blocks for resource validation.
- Master custom error messages and validation conditions.
- Apply validation rules to common real-world scenarios.

---

## 1. Overview of Validation in Terraform

Terraform provides several mechanisms to validate configurations and catch errors early:

1. **Variable Validation** - Validate input variables before they're used
2. **Preconditions** - Validate assumptions before resource creation/modification
3. **Postconditions** - Validate resource outputs after creation

These validation mechanisms help catch configuration errors early and provide clear error messages.

---

## 2. Variable Validation Blocks

### Purpose

Variable validation allows you to enforce rules on variable values before Terraform uses them in your configuration. This catches errors during `terraform plan` or `terraform apply`.

### Basic Syntax

```hcl
variable "vm_size" {
  description = "Azure VM size"
  type        = string

  validation {
    condition     = can(regex("^Standard_B[0-9]+[a-z]?$", var.vm_size))
    error_message = "VM size must be a B-series instance (e.g., Standard_B2s, Standard_B4ms)."
  }
}

```

### Validation Block Components

- **`condition`**: A boolean expression that must evaluate to `true` for validation to pass
- **`error_message`**: Custom error message shown when validation fails

### Common Validation Patterns

#### Pattern 1: String Format Validation

```hcl
variable "vm_size" {
  description = "Azure VM size"
  type        = string

  validation {
    condition     = can(regex("^Standard_B[0-9]+[a-z]?$", var.vm_size))
    error_message = "VM size must be a B-series instance (e.g., Standard_B2s, Standard_B4ms)."
  }
}

```

#### Pattern 2: Value Range Validation

```hcl
variable "vm_count" {
  description = "Number of Azure VMs to deploy"
  type        = number

  validation {
    condition     = var.vm_count >= 1 && var.vm_count <= 20
    error_message = "VM count must be between 1 and 20."
  }
}

```

#### Pattern 3: Allowed Values Validation

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "test", "stage", "prod"], var.environment)
    error_message = "Environment must be one of: dev, test, stage, prod."
  }
}

```

#### Pattern 4: CIDR Block Validation

```hcl
variable "vnet_cidr" {
  description = "VNet CIDR block"
  type        = string

  validation {
    condition     = can(cidrhost(var.vnet_cidr, 0))
    error_message = "VNet CIDR must be a valid IPv4 CIDR block (e.g., 10.1.0.0/16)."
  }
}

```

#### Pattern 5: Multiple Validation Rules

```hcl
variable "admin_password" {
  description = "Admin password for Azure VM"
  type        = string
  sensitive   = true

  validation {
    condition     = length(var.admin_password) >= 12
    error_message = "Password must be at least 12 characters long."
  }

  validation {
    condition     = can(regex("[A-Z]", var.admin_password))
    error_message = "Password must contain at least one uppercase letter."
  }

  validation {
    condition     = can(regex("[a-z]", var.admin_password))
    error_message = "Password must contain at least one lowercase letter."
  }

  validation {
    condition     = can(regex("[0-9]", var.admin_password))
    error_message = "Password must contain at least one number."
  }

  validation {
    condition     = can(regex("[^A-Za-z0-9]", var.admin_password))
    error_message = "Password must contain at least one special character."
  }
}

```

### Using Functions in Validation

```hcl
variable "tags" {
  description = "Resource tags"
  type        = map(string)
  
  validation {
    condition     = alltrue([for k, v in var.tags : length(k) <= 128 && length(v) <= 256])
    error_message = "Tag keys must be <= 128 characters and values <= 256 characters."
  }
}
```

---

## 3. Precondition Blocks

### Purpose

Preconditions validate assumptions about resources or data sources **before** Terraform creates or modifies resources. They're placed in `lifecycle` blocks.

### Basic Syntax

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  name           = var.name
  size = var.size
  
  lifecycle {
    precondition {
      condition     = var.size != "Standard_DS3_v2"
      error_message = "Standard_DS3_v2 instance type is not supported."
    }
  }
}
```

### Common Use Cases

#### Use Case 1: Validate Data Source Results - Validate that a Shared Image Gallery version exists

```hcl
data "azurerm_shared_image_version" "latest" {
  name                = var.image_version
  image_name          = var.image_name
  gallery_name        = var.gallery_name
  resource_group_name = var.gallery_rg
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  source_image_id = data.azurerm_shared_image_version.latest.id

  lifecycle {
    precondition {
      condition     = data.azurerm_shared_image_version.latest.id != null
      error_message = "No valid image version found in Shared Image Gallery."
    }
  }
}

```

#### Use Case 2: Validate Module Inputs - Validate VNet CIDR and enforce minimum subnet size

```hcl
module "network" {
  source = "./modules/network"

  vnet_cidr = var.vnet_cidr

  lifecycle {
    precondition {
      condition     = can(cidrhost(var.vnet_cidr, 0))
      error_message = "VNet CIDR must be a valid IPv4 CIDR block."
    }

    precondition {
      condition     = tonumber(split("/", var.vnet_cidr)[1]) <= 24
      error_message = "VNet CIDR must be /24 or larger (e.g., /16, /20)."
    }
  }
}

```

#### Use Case 3: Validate Resource Dependencies - Prevent NSG rules from allowing 0.0.0.0/0 in production

```hcl
resource "azurerm_network_security_group" "web" {
  name                = "web-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "allow-http"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = var.allowed_cidr
    destination_address_prefix = "*"
  }

  lifecycle {
    precondition {
      condition     = var.allowed_cidr != "0.0.0.0/0" || var.environment == "dev"
      error_message = "Cannot allow 0.0.0.0/0 in non-development environments."
    }
  }
}

```

---

## 4. Postcondition Blocks

### Purpose

Postconditions validate resource outputs **after** Terraform creates or modifies resources. They're placed in `lifecycle` blocks and can reference the resource's own attributes.

### Basic Syntax

```hcl
resource "azurerm_mssql_database" "main" {
  name           = "prod-db"
  server_id      = azurerm_mssql_server.sql.id
  sku_name       = "S2"

  lifecycle {
    postcondition {
      condition     = self.status == "Online"
      error_message = "Azure SQL database must be in 'Online' state after creation."
    }
  }
}

```

### Common Use Cases

#### Use Case 1: Validate Resource State - Ensure an Azure SQL database is fully provisioned

```hcl
resource "azurerm_mssql_database" "main" {
  name           = "prod-db"
  server_id      = azurerm_mssql_server.sql.id
  sku_name       = "S2"

  lifecycle {
    postcondition {
      condition     = self.status == "Online"
      error_message = "Azure SQL database must be in 'Online' state after creation."
    }
  }
}

```

#### Use Case 2: Validate Output Values - Validate that a Public IP actually received an address

```hcl
resource "azurerm_public_ip" "pip" {
  name                = "web-pip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"

  lifecycle {
    postcondition {
      condition     = self.ip_address != null && self.ip_address != ""
      error_message = "Public IP must have a valid assigned IP address."
    }
  }
}

```

#### Use Case 3: Validating Key Vault secrets or identity assignments

```hcl
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = var.db_password
  key_vault_id = azurerm_key_vault.kv.id

  lifecycle {
    postcondition {
      condition     = self.id != null
      error_message = "Key Vault secret creation failed."
    }
  }
}

```

---

## 5. Combining Preconditions and Postconditions

You can use both preconditions and postconditions in the same resource:

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

  lifecycle {
    postcondition {
      condition     = self.private_ip_address != null
      error_message = "VM must have a private IP assigned."
    }

    postcondition {
      condition     = self.provisioning_state == "Succeeded"
      error_message = "VM provisioning did not complete successfully."
    }
  }
}

```

---

## 6. Real-World Examples

### Example 1: Complete Variable Validation

```hcl
variable "vm_config" {
  description = "Azure VM configuration"
  type = object({
    vm_size      = string
    vm_count     = number
    environment  = string
    allowed_cidr = string
  })

  validation {
    condition     = contains(["dev", "staging", "prod"], var.vm_config.environment)
    error_message = "Environment must be dev, staging, or prod."
  }

  validation {
    condition     = var.vm_config.vm_count >= 1 && var.vm_config.vm_count <= 20
    error_message = "VM count must be between 1 and 20."
  }

  validation {
    condition     = can(regex("^Standard_[A-Za-z0-9]+$", var.vm_config.vm_size))
    error_message = "VM size must follow Azure naming (e.g., Standard_B2s)."
  }

  validation {
    condition     = can(cidrhost(var.vm_config.allowed_cidr, 0))
    error_message = "Allowed CIDR must be a valid IPv4 CIDR block."
  }
}

resource "azurerm_linux_virtual_machine" "web" {
  count               = var.vm_config.vm_count
  name                = "web-${count.index}"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = var.vm_config.vm_size
  admin_username      = "azureuser"
  network_interface_ids = [
    azurerm_network_interface.nic[count.index].id
  ]

  lifecycle {
    precondition {
      condition     = var.vm_config.allowed_cidr != "0.0.0.0/0" || var.vm_config.environment == "dev"
      error_message = "Cannot allow 0.0.0.0/0 in non-dev environments."
    }
  }
}

```

### Example 2: Data Source Validation

```hcl
data "azurerm_virtual_network" "selected" {
  name                = var.vnet_name
  resource_group_name = var.vnet_rg

  lifecycle {
    precondition {
      condition     = data.azurerm_virtual_network.selected.id != null
      error_message = "VNet ${var.vnet_name} not found in resource group ${var.vnet_rg}."
    }

    precondition {
      condition     = length(data.azurerm_virtual_network.selected.dns_servers) > 0
      error_message = "VNet must have DNS servers configured."
    }
  }
}

resource "azurerm_network_interface" "nic" {
  name                = "web-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = data.azurerm_virtual_network.selected.subnets[0].id
    private_ip_address_allocation = "Dynamic"
  }

  lifecycle {
    postcondition {
      condition     = self.private_ip_address != null
      error_message = "NIC must have a private IP assigned."
    }
  }
}

```

### Example 3: Module Output Validation

```hcl
module "network" {
  source = "./modules/network"

  vnet_cidr = "10.0.0.0/16"

  lifecycle {
    precondition {
      condition     = can(cidrhost("10.0.0.0/16", 0))
      error_message = "Invalid CIDR block provided to network module."
    }
  }
}

resource "azurerm_linux_virtual_machine" "web" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.nic.id
  ]

  lifecycle {
    precondition {
      condition     = module.network.public_subnet_id != null
      error_message = "Network module must provide a public subnet ID."
    }
  }
}

```

---

## 7. Best Practices

### ✅ Do's

1. **Use validation for user inputs:**
   ```hcl
   variable "port" {
     type = number
     validation {
       condition     = var.port > 0 && var.port <= 65535
       error_message = "Port must be between 1 and 65535."
     }
   }
   ```

2. **Provide clear error messages:**
   ```hcl
   validation {
     condition     = var.size != "Standard_DS3_v2"
     error_message = "Standard_DS3_v2 instance type is deprecated. Use a newer sku suze instead."
   }
   ```

3. **Validate data source results:**
   ```hcl
data "azurerm_shared_image_version" "latest" {
  name                = var.image_version
  image_name          = var.image_name
  gallery_name        = var.gallery_name
  resource_group_name = var.gallery_rg

  lifecycle {
    precondition {
      condition     = self.id != null
      error_message = "No image version found in Shared Image Gallery."
    }
  }
}

   ```

4. **Use postconditions to verify resource state:**
   ```hcl
resource "azurerm_public_ip" "pip" {
  name                = "web-pip"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  allocation_method   = "Static"

  lifecycle {
    postcondition {
      condition     = self.ip_address != null
      error_message = "Public IP must have an assigned IP address."
    }
  }
}
   ```

### ❌ Don'ts

1. **Don't over-validate:**
   ```hcl
   # ❌ BAD - Too restrictive
   validation {
     condition     = var.instance_type == "Standard_DS3_v2"
     error_message = "Only Standard_DS3_v2 allowed."
   }
   
   # ✅ GOOD - Reasonable constraint
   validation {
     condition     = can(regex("^DS[2-3]\\.[a-z]+$", var.instance_type))
     error_message = "Instance type must be DS2 or DS3 family."
   }
   ```

2. **Don't use validation for business logic:**
   ```hcl
   # ❌ BAD - Business logic, not validation
   validation {
     condition     = var.cost < 100
     error_message = "Cost too high."
   }
   ```

3. **Don't ignore validation errors:**
   - Always fix validation errors rather than working around them
   - Validation exists to prevent configuration mistakes

---

## 8. Exam-Style Practice Questions

### Question 1
What is the purpose of a `validation` block in a variable definition?
A) To validate resource state after creation
B) To validate variable inputs before they're used
C) To validate module outputs
D) To validate provider configuration

<details>
<summary>Show Answer</summary>
Answer: **B** - Variable validation blocks validate input values before Terraform uses them in the configuration, catching errors during plan or apply.
</details>

---

### Question 2
Where do you place `precondition` and `postcondition` blocks?
A) In the variable block
B) In the resource lifecycle block
C) In the provider block
D) In the terraform block

<details>
<summary>Show Answer</summary>
Answer: **B** - Preconditions and postconditions are placed in the `lifecycle` block of a resource or data source.
</details>

---

### Question 3
What is the difference between a precondition and a postcondition?
A) Preconditions run after apply, postconditions run before
B) Preconditions validate before resource creation, postconditions validate after
C) There is no difference
D) Preconditions are for variables, postconditions are for resources

<details>
<summary>Show Answer</summary>
Answer: **B** - Preconditions validate assumptions before Terraform creates or modifies resources. Postconditions validate resource outputs after creation or modification.
</details>

---

### Question 4
You want to ensure an instance type variable only accepts DS2 or DS3 sku types. Which validation should you use?
A) `condition = var.instance_type == "Standard_DS2_v2" || var.instance_type == "Standard_DS3_v2"`
B) `condition = can(regex("^DS[2-3]\\.[a-z]+$", var.instance_type))`
C) `condition = var.instance_type != "Standard_DS1_v2"`
D) No validation needed

<details>
<summary>Show Answer</summary>
Answer: **B** - Using a regex pattern with `can()` allows validation of any DS2 or DS3 sku type, not just specific ones. This is more flexible than listing individual types.
</details>

---

### Question 5
When do variable validation blocks execute?
A) Only during `terraform apply`
B) Only during `terraform plan`
C) During both `terraform plan` and `terraform apply`
D) Only when the variable is used in a resource

<details>
<summary>Show Answer</summary>
Answer: **C** - Variable validation blocks execute during both `terraform plan` and `terraform apply`, catching errors early in the workflow.
</details>

---

## 9. Key Takeaways

- **Variable validation**: Use `validation` blocks in variable definitions to enforce rules on input values.
- **Preconditions**: Validate assumptions before resource creation/modification using `lifecycle { precondition { ... } }`.
- **Postconditions**: Validate resource outputs after creation/modification using `lifecycle { postcondition { ... } }`.
- **Error messages**: Always provide clear, helpful error messages in validation blocks.
- **Early detection**: Validation catches configuration errors during `terraform plan`, before any infrastructure changes.
- **Multiple validations**: You can have multiple validation blocks in a single variable definition.
- **Functions**: Use Terraform functions like `regex()`, `can()`, `contains()`, and `alltrue()` in validation conditions.

---

## References

- [Terraform Variable Validation](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules)
- [Terraform Preconditions and Postconditions](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions)
- [Terraform Functions](https://developer.hashicorp.com/terraform/language/functions)

