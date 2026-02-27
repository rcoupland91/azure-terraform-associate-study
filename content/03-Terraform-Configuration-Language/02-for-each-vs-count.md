# For Each Vs Count

## Learning Objectives
- Understand when to use `for_each` vs `count` to create multiple resource instances.
- Learn the key differences, limitations, and best practices for each.
- Master exam-style questions about resource creation patterns.
- Recognize common pitfalls and when each approach is appropriate.

---

## 1. Overview: Creating Multiple Resources

Terraform provides two meta-arguments to create multiple instances of a resource:
- **`count`**: Creates resources based on a number
- **`for_each`**: Creates resources based on a map or set

Both serve similar purposes but have different use cases and behaviors.

---

## 2. Using `count`

### Basic Syntax

```hcl
resource "azurerm_linux_virtual_machine" "web" {
  count               = 2
  name                = "myvm-${count.index}"
  resource_group_name = "test-rg"
  location            = "uksouth"
  size                = "Standard_B1s"

  admin_username      = "azureuser"

  network_interface_ids = [
    "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/networkInterfaces/myvm-${count.index}-nic"
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }
}
```

**What it does:**
- Creates 3 identical EC2 instances
- `count.index` provides 0, 1, 2 for each instance
- Resource address: `azurerm_linux_virtual_machine.web[0]`, `azurerm_linux_virtual_machine.web[1]`, `azurerm_linux_virtual_machine.web[2]`

### Referencing Resources Created with `count`

```hcl
# Output all instance IDs
output "instance_ids" {
  value = azurerm_linux_virtual_machine.web[*].id
}

# Output specific instance
output "first_instance_id" {
  value = azurerm_linux_virtual_machine.web[0].id
}

# Reference in another resource
resource "azurerm_network_interface" "vmnic" {
  count               = 2
  name                = "vmnic-${count.index}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.app.id
    private_ip_address_allocation = "Dynamic"
    load_balancer_backend_address_pools_ids = [
      azurerm_lb_backend_address_pool.example.id
    ]
  }
}
```

### When to Use `count`

✅ **Good for:**
- Creating a known number of identical resources
- Simple scenarios where you just need N copies
- When order/index matters

❌ **Not ideal for:**
- Resources that need unique names/identifiers
- When you might need to remove middle items (causes recreation)
- Maps or sets of items with unique keys

---

## 3. Using `for_each`

### Basic Syntax (with Map)

```bash
storage_accounts = [
  { name = "sa-01" },
  { name = "sa-02" }
]
```

```hcl
resource "azurerm_storage_account" "example" {
  for_each = { for account in var.storage_accounts : account.name => account if var.environment != "dr" }

  name                     = each.value.name
  resource_group_name      = azurerm_resource_group.rg[each.value.name].name
  location                 = var.location
  account_tier             = try(each.value.account_tier, "Standard")
  account_kind             = try(each.value.account_kind, "StorageV2")
  account_replication_type = try(each.value.account_replication_type, "LRS")
}
```

**What it does:**
- Creates 3 instances with unique keys: `sa-01`, `sa-02`
- `each.key = the key values as per "account.name" (sa-01)
- `each.value` = the map value (e.g., "sa-01")
- Resource address: `azurerm_storage_account.example["sa-01"]`, `azurerm_storage_account.example["sa-02"]`, etc.

### Referencing Resources Created with `for_each`

```hcl
# Output all instance IDs as map
output "instance_ids" {
  value = {
    for k, instance in azurerm_storage_account.example : k => instance.id
  }
}

# Output specific instance
output "storage_account_output" {
  value = azurerm_storage_account.example["sa-01"].id
}

# Reference in another resource
resource "azurerm_storage_account_network_rules" "sa_fw_rules" {
   storage_account_id = values(azurerm_storage_account.example)[*].id
}
```

### When to Use `for_each`

✅ **Good for:**
- Resources with unique identifiers (names, tags)
- Maps or sets of items
- When you need to add/remove specific items without affecting others
- Resources that should not be recreated when list order changes

❌ **Not ideal for:**
- Simple "create N copies" scenarios (count is simpler)
- When order/index is what matters

---

## 4. Key Differences: `count` vs `for_each`

| Feature | `count` | `for_each` |
|---------|---------|------------|
| **Input Type** | Number | Map or Set |
| **Resource Address** | `resource[0]`, `resource[1]` | `resource["key"]` |
| **Index Access** | `count.index` | `each.key`, `each.value` |
| **Removing Middle Item** | Recreates all items after it | Only affects that specific item |
| **Order Matters** | ✅ Yes | ❌ No |
| **Use with Maps** | ❌ No (convert to list) | ✅ Yes |
| **Use with Sets** | ❌ No | ✅ Yes |
| **Better for Unique IDs** | ❌ No | ✅ Yes |

---

## 5. Practical Comparison Examples

### Example 1: Simple Web Servers

**Using `count`:**
```hcl
resource "azurerm_linux_virtual_machine" "web" {
  count               = 2
  name                = "myvm-${count.index}"
  resource_group_name = "test-rg"
  location            = "uksouth"
  size                = "Standard_B1s"

  admin_username      = "azureuser"

  network_interface_ids = [
    "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/networkInterfaces/myvm-${count.index}-nic"
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }
}
```

**Using `for_each`:**
```hcl
resource "azurerm_linux_virtual_machine" "web" {
  for_each = toset(["web-1", "web-2", "web-3"])
  
  ami           = "ami-0123456789abcdef0"  # Example AMI ID
  instance_type = "t2.micro"
  tags = {
    Name = each.key
  }
}

resource "azurerm_linux_virtual_machine" "web" {
  for_each = toset(["web-1", "web-2", "web-3"])
  name                = "${each.value}-nic"
  resource_group_name = "test-rg"
  location            = "uksouth"
  size                = "Standard_B1s"

  admin_username      = "azureuser"

  network_interface_ids = [
    "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/networkInterfaces/myvm-${each.value}-nic"
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }
}

```

### Example 2: Different Instance Types

**Using `for_each` (better choice):**
```hcl
resource "azurerm_linux_virtual_machine" "web" {
  for_each = {
    frontend = "Standard_B2s"
    backend  = "Standard_D2s_v3"
    cache    = "Standard_B1ms"
  }
  
  name                = each.key
  resource_group_name = "test-rg"
  location            = "uksouth"
  size                = each.value
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.webnic[each.key].id
  ]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }
}
```

**Why `for_each` here?**
- Each virtual machine has unique configuration
- Names map to roles (frontend, backend, cache)
- Removing one doesn't affect others' addresses

---

## 6. Common Pitfalls and Gotchas

### Pitfall 1: Removing Middle Item with `count`

```hcl
# Initial: count = 3 creates [0], [1], [2]
# Change to: count = 2
# Result: [0] stays, [1] becomes new [1] (was [2]), [2] destroyed
# Old [1] is destroyed even though you only wanted to remove [2]
```

**Solution:** Use `for_each` if you need to remove specific items.

### Pitfall 2: `for_each` Requires Map or Set

```hcl
# ❌ This will ERROR
resource "azurerm_linux_virtual_machine" "web" {
  for_each = ["web-1", "web-2"]  # List, not set!
}

# ✅ Correct - convert to set
resource "azurerm_linux_virtual_machine" "web" {
  for_each = toset(["web-1", "web-2"])
}
```

### Pitfall 3: Cannot Use Both `count` and `for_each`

```hcl
# ❌ ERROR: Cannot use both count and for_each
resource "azurerm_linux_virtual_machine" "web" {
  count    = 3
  for_each = { a = "b" }
}
```

**You must choose one or the other.**

### Pitfall 4: Changing from `count` to `for_each`

```hcl
# Initial state with count
resource "azurerm_linux_virtual_machine" "web" {
  count = 2
  # Creates: azurerm_linux_virtual_machine.web[0], azurerm_linux_virtual_machine.web[1]
}

# Changing to for_each
resource "azurerm_linux_virtual_machine" "web" {
  for_each = toset(["web-1", "web-2"])
  # Creates: azurerm_linux_virtual_machine.web["web-1"], azurerm_linux_virtual_machine.web["web-2"]
}
```

**This will cause Terraform to destroy old resources and create new ones.**  
**Solution:** Use `terraform state mv` to migrate:
```bash
terraform state mv 'azurerm_linux_virtual_machine.web[0]' 'azurerm_linux_virtual_machine.web["web-1"]'
terraform state mv 'azurerm_linux_virtual_machine.web[1]' 'azurerm_linux_virtual_machine.web["web-2"]'
```

---

## 7. Converting Between `count` and `for_each`

### Converting List to `for_each`

```hcl
variable "server_names" {
  type    = list(string)
  default = ["web-1", "web-2", "web-3"]
}

# Option 1: Convert list to set
resource "azurerm_linux_virtual_machine" "web" {
  for_each = toset(var.server_names)
  # ...
}

# Option 2: Convert list to map with index
resource "azurerm_linux_virtual_machine" "web" {
  for_each = {
    for idx, name in var.server_names : name => idx
  }
  # ...
}
```

### Converting Map to `count`

```hcl
variable "vms" {
  type = map(string)
  default = {
    web-1 = "t2.micro"
    web-2 = "t3.small"
  }
}

# Convert map to count (loses key names)
resource "azurerm_linux_virtual_machine" "web" {
  count = length(var.vms)

  size = values(var.vms)[count.index]
  tags = {
    Name = keys(var.vms)[count.index]
  }
}
```

**Note:** Converting map → count loses the ability to reference by key and causes issues when removing items.

---

## 8. Real-World Examples

### Example 1: Multi-Region Resources

```hcl
variable "regions" {
  type = set(string)
  default = ["uksouth", "ukwest"]
}

resource "azurerm_storage_account" "logs" {
  for_each = var.regions
  
  name = "company-logs-${each.value}"
  
  # Use provider alias for each region
  provider = azurerm.region[each.value]
}
```

**Why `for_each`?** Each storage account has unique name based on region key.

---

### Example 2: Dynamic Security Groups

```hcl
locals {
  security_groups = {
    web = {
      ports = [80, 443]
      cidr  = "0.0.0.0/0"
    }
    db = {
      ports = [3306]
      cidr  = "10.0.0.0/16"
    }
  }
}

resource "azurerm_network_security_group" "ingress" {
  for_each = {
    for sg_name, sg_config in local.security_groups :
    sg_name => sg_config
  }
}
```

---

### Example 3: Conditional Resource Creation

```hcl
# Using count (simpler for boolean)
variable "enabled_for_disk_encryption" {
  type    = bool
  default = true
}

resource "azurerm_key_vault" "main" {
  count = var.enabled_for_disk_encryption ? 1 : 0
  
  # ...
}

# Using for_each (for conditional map)
variable "environments" {
  type = map(object({
    enabled = bool
    region  = string
  }))
  default = {
    prod = { enabled = true, region = "uksouth" }
    dev  = { enabled = false, region = "ukwest" }
  }
}

resource "azurerm_key_vault" "main" {
  for_each = {
    for k, v in var.environments : k => v
    if v.enabled
  }
  
  # Only creates resources for enabled environments
}
```

---

## 9. Exam-Style Practice Questions

### Question 1
You need to create 5 identical VMs. Which approach is simplest?
A) `for_each` with a set
B) `count = 5`
C) `for_each` with a map
D) Define 5 separate resources

<details>
<summary>Show Answer</summary>
Answer: **B** - `count = 5` is the simplest for identical resources.
</details>

---

### Question 2
What happens if you remove the middle item from a `count`-based resource list?
A) Only that item is removed
B) All items after it are recreated
C) Nothing happens
D) All items are recreated

<details>
<summary>Show Answer</summary>
Answer: **B** - With `count`, removing a middle item causes all subsequent items to be recreated with new indices.
</details>

---

### Question 3
Which data types can be used with `for_each`?
A) List and Map
B) Set and Map only
C) Number and List
D) Any data type

<details>
<summary>Show Answer</summary>
Answer: **B** - `for_each` only accepts maps or sets. Lists must be converted using `toset()`.
</details>

---

### Question 4
You have a map of instance configurations with unique names. Removing one instance should not affect others. Which should you use?
A) `count`
B) `for_each`
C) Both work the same
D) Neither - use separate resources

<details>
<summary>Show Answer</summary>
Answer: **B** - `for_each` is better for maps with unique keys because removing one item doesn't affect others' addresses.
</details>

---

### Question 5
What is the correct syntax to reference a resource created with `for_each`?
A) `azurerm_linux_virtual_machine.web[0]`
B) `azurerm_linux_virtual_machine.web["web-1"]`
C) `azurerm_linux_virtual_machine.web.web-1`
D) Both A and B work

<details>
<summary>Show Answer</summary>
Answer: **B** - Resources created with `for_each` use map-style addresses: `resource["key"]`. `count` uses index: `resource[0]`.
</details>

---

## 10. Decision Tree

```
Need to create multiple instances?
│
├─ Are they identical?
│  ├─ Yes → Use `count = N`
│  └─ No → Continue
│
├─ Do they have unique names/keys?
│  ├─ Yes → Use `for_each` with map
│  └─ No → Continue
│
├─ Is it a list of items?
│  ├─ Yes → Convert to set: `for_each = toset(list)`
│  └─ No → Continue
│
└─ Need to add/remove specific items without affecting others?
   ├─ Yes → Use `for_each`
   └─ No → `count` is fine
```

---

## 11. Key Takeaways

- **`count`**: Use for a known number of identical resources. Creates indexed addresses `[0]`, `[1]`, etc.
- **`for_each`**: Use for maps/sets with unique keys. Creates map-style addresses `["key"]`.
- **`count` limitation**: Removing middle items causes recreation of subsequent items.
- **`for_each` limitation**: Must use maps or sets, not plain lists.
- **Cannot combine**: You can't use both `count` and `for_each` on the same resource.
- **Conversion**: Lists can be converted to sets with `toset()` for `for_each`.
- **Migration**: Changing from `count` to `for_each` requires state migration with `terraform state mv`.

---

## References

- [Terraform count Meta-Argument](https://developer.hashicorp.com/terraform/language/meta-arguments/count)
- [Terraform for_each Meta-Argument](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)
- [When to Use `for_each` Instead of `count`](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each#when-to-use-for_each-instead-of-count)

