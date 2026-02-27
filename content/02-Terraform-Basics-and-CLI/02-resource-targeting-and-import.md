# Resource Targeting And Import

## Learning Objectives
- Learn how to target specific resources for Terraform operations.
- Understand the `terraform import` process for bringing existing infrastructure under management.
- Master common import scenarios and best practices.
- Know when and how to use targeting effectively.

---

## 1. Resource Targeting

### What is Resource Targeting?

Resource targeting allows you to apply Terraform operations to **specific resources** instead of all resources in your configuration.

**Syntax:**
```bash
terraform plan -target=resource_address
terraform apply -target=resource_address
terraform destroy -target=resource_address
```

### Resource Address Format

Resources are identified by their address:
- Simple resource: `resource_type.resource_name`
- Resource with count: `resource_type.resource_name[0]`
- Resource with for_each: `resource_type.resource_name["key"]`
- Module resource: `module.module_name.resource_type.resource_name`

### Basic Targeting Examples

**Target a specific resource:**
```bash
terraform plan -target=azure_vm.web
terraform apply -target=azure_vm.web
```

**Target multiple resources:**
```bash
terraform apply \
  -target=azure_vm.web \
  -target=azure_nsg.web
```

**Target a resource in a module:**
```bash
terraform apply -target=module.testapp_vm
```

**Target resources with count/for_each:**
```bash
# Count
terraform apply -target=module.testapp_vm[0]

# For_each
terraform apply -target=module.testapp_vm["run"]
```

### Use Cases for Targeting

1. **Testing specific resources:**
   ```bash
   terraform apply -target=azure_vm.web
   ```
   Apply only the web instance to test changes quickly.

2. **Partial applies:**
   ```bash
   terraform apply -target=azure_vnet.main -target=azure_subnet.private
   ```
   Create networking before compute resources.

3. **Incremental updates:**
   ```bash
   terraform plan -target=azure_vm.app
   terraform apply -target=azure_vm.app
   ```
   Update one component at a time.

4. **Emergency fixes:**
   ```bash
   terraform apply -target=network_security_group.critical
   ```
   Quickly fix a critical security group without touching other resources.

### Important Limitations

⚠️ **Targeting doesn't resolve dependencies automatically:**
- If resource B depends on resource A, targeting B will fail unless A exists
- Terraform may create dependencies if it can determine them statically
- Some dependencies require manual targeting

**Example:**
```bash
# This might fail if vnet resource doesn't exist
terraform apply -target=azure_vm.web

# You need to target both
terraform apply \
  -target=azure_vnet.main \
  -target=azure_vm.web
```

### Targeting Best Practices

✅ **Do:**
- Use for testing specific resources during development
- Use for incremental rollouts
- Use for fixing individual components
- Always verify dependencies

❌ **Don't:**
- Don't rely on targeting as a permanent workflow
- Don't skip dependency resources
- Don't use targeting to avoid fixing dependency issues
- Don't forget to run full `terraform plan` periodically

---

## 2. Importing Existing Infrastructure

### What is Import?

**Import** brings existing infrastructure under Terraform management without recreating it.

**When to use:**
- Infrastructure was created manually (console, CLI, etc.)
- Migrating from another IaC tool (CloudFormation, etc.)
- Resources were created before Terraform was adopted
- Fixing state drift

### Import Command Syntax

```bash
terraform import resource_address infrastructure_id
```

**Components:**
- `resource_address`: Terraform resource address (e.g., `'azurerm_private_dns_a_record.kv_record[\"key-vault-01\"]'`  )
- `id`: Azure-specific ID (e.g.,  `"/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net/A/key-vault-01")`

### Step-by-Step Import Process

**Step 1: Add resource block to configuration**
```hcl
resource "azurerm_private_dns_a_record" "kv_record" {
  for_each = { for dns in var.dns_records : dns.name => dns
  # Configuration must match existing resource attributes
    name                = "test-DNS-record"
    zone_name           = "privatelink.blob.core.windows.net"
    resource_group_name = "test-rg"
}
```

**Step 2: Run import command**
```bash
terraform import 'azurerm_private_dns_a_record.kv_record[\"key-vault-01\"]' "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/privateDnsZones/privatelink.vaultcore.azure.net/A/key-vault-01"
```

**Step 3: Verify in state**
```bash
terraform state show 'azurerm_private_dns_a_record.kv_record[\"key-vault-01\"]
```

**Step 4: Review plan**
```bash
terraform plan
```
Terraform will show any differences between config and actual resource.

**Step 5: Update configuration to match reality**
Update your `.tf` file to match the imported resource's actual attributes.

**Step 6: Apply to sync**
```bash
terraform apply
```
This should show no changes if config matches reality.

### Common Import Examples

#### Importing an Azure Storage Account

```hcl
# Configuration
resource "azurerm_storage_account" "data" {
  name                     = "test-sa"
  resource_group_name      = "test-rg"
  location                 = "westeurope"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

```

```bash
terraform import azurerm_storage_account.data "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Storage/storageAccounts/test-sa"
```

#### Importing an Azure VM

```hcl
# Configuration
resource "azurerm_linux_virtual_machine" "web" {
  name                = "myvm01"
  resource_group_name = "test-rg"
  location            = "uksouth"
  size                = "Standard_B1s"

  admin_username      = "azureuser"

  network_interface_ids = [
    "/subscriptions/<SUB_ID>/resourceGroups/test-rg/providers/Microsoft.Network/networkInterfaces/myvm01-nic"
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

```bash
terraform import azurerm_linux_virtual_machine.web "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm01"

```

**Note:** Import only the instance. Additional resources may need separate imports.

#### Importing a VNET

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "my-vnet"
  resource_group_name = "test-rg"
  location            = "uksouth"
  address_space       = ["10.0.0.0/16"]
}

```

```bash
terraform import azurerm_virtual_network.main "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/virtualNetworks/my-vnet"
```

#### Importing Resources with Count

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

```bash
terraform import azurerm_linux_virtual_machine.web[0] "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm-0"

terraform import azurerm_linux_virtual_machine.web[1] "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm-1"
```

#### Importing Resources with for_each

```hcl
storage_accounts = [
  { name = "appstore" },
  { name = "webstore" }
]
```
```hcl

resource "azurerm_storage_account" "logs" {
  for_each = { for account in var.storage_accounts : account.name => account }

  name                     = each.value.name
  resource_group_name      = azurerm_resource_group.rg[each.value.name].name
  location                 = "uksouth"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

```

```bash
terraform import 'azurerm_storage_account.logs["appstore"]' "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>/providers/Microsoft.Storage/storageAccounts/appstore"
terraform import 'azurerm_storage_account.logs["webstore"]' "/subscriptions/<SUB_ID>/resourceGroups/<RG_NAME>/providers/Microsoft.Storage/storageAccounts/webstore"
```

#### Importing Module Resources

```hcl
module "vnet" {
  source = "./modules/vnet-gw"
}
```

```bash
terraform import module.vpc.azure_vnet.main vnet-01
```

### Finding Resource IDs

**Azure CLI:**
```bash
# List all VMs with name + resource ID
az vm list --query '[].{name:name, id:id, rg:resourceGroup}' -o table

# List all storage accounts
az storage account list --query '[].{name:name, rg:resourceGroup, id:id}' -o table

# Get a specific storage account’s resource ID
az storage account show -g <RESOURCE_GROUP> -n <ACCOUNT_NAME> --query id -o tsv

# Vnet
az network vnet list --query '[].{name:name, rg:resourceGroup, id:id, address:addressSpace}' -o table
```

**Azure Portal:**
# Virtual Machine
- Azure Portal → Virtual Machines → Settings → Properties → Resource ID

# Storage Account 
- Azure Portal → Storage Accounts → Settings → Properties → Resource ID

# Virtual Network 
- Azure Portal → Virtual Networks → Settings → Properties → Resource ID

### Import Challenges

#### Challenge 1: Configuration Mismatch

After import, `terraform plan` shows many changes because your configuration doesn't match reality.

**Solution:**
1. Run `terraform show azurerm_linux_virtual_machine.web` (or resource equivalent) to see actual attributes
2. Update your configuration to match
3. Run `terraform plan` again to verify

#### Challenge 2: Missing Dependencies

Resource exists but depends on other resources not in Terraform.

**Example:**
```bash
terraform import azurerm_linux_virtual_machine.web "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm01"
# Error: Network Security group nsg-test doesn't exist in state
```

**Solution:**
1. Import dependencies first:
   ```bash
   terraform import azurerm_network_security_group.web sg-test
   terraform import azurerm_linux_virtual_machine.web "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm01"
   ```

#### Challenge 3: Complex Resources

Some resources have many attributes that must match exactly.

**Solution:**
- Use `terraform show` to get all attributes
- Copy attributes into configuration
- Or use tools like `terraformer` for bulk imports

### Import vs Manual State Manipulation

**Import (recommended):**
- ✅ Safe and verified
- ✅ Creates proper resource in state
- ✅ Validates resource exists

**Manual state add (advanced, risky):**
```bash
# NOT recommended - use import instead
terraform state rm azurerm_linux_virtual_machine.web
terraform import azurerm_linux_virtual_machine.web "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm01"
```

### Bulk Import Strategies

**Option 1: Script multiple imports**
```bash
#!/bin/bash
terraform import azurerm_linux_virtual_machine.web[0] "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm-0"
terraform import azurerm_linux_virtual_machine.web[1] "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm-1"
```

**Option 2: Use terraformer (third-party tool)**
```bash
terraformer import azure --resources=virtual_network,subnet,network_security_group --subscription="<SUBSCRIPTION_ID>" --resource-group="<RESOURCE_GROUP_NAME>"
```

---

## 3. Combining Targeting and Import

### Workflow: Import and Verify

```bash
# 1. Import the resource
terraform import azurerm_linux_virtual_machine.web "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm"

# 2. Plan with targeting to see differences
terraform plan -target=azurerm_linux_virtual_machine.web

# 3. Update configuration if needed

# 4. Apply to sync
terraform apply -target=azurerm_linux_virtual_machine.web
```

### Workflow: Import Dependencies First

```bash
# 1. Import Vnet
terraform import azurerm_virtual_network.main "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Network/virtualNetworks/my-vnet"

# 2. Import network security group
terraform import azurerm_network_security_group.web sg-test

# 3. Import Virtual Machine (depends on above)
terraform import azurerm_linux_virtual_machine.web "/subscriptions/{subID}/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/myvm01"

# 4. Plan all imported resources
terraform plan -target=azurerm_virtual_network.main -target=azurerm_network_security_group.web -target=azurerm_linux_virtual_machine.web
```

---

## 4. Practice Questions

### Question 1
You want to apply changes to only one Virtual Machine without affecting others. What command should you use?
A) `terraform apply -filter=azurerm_linux_virtual_machine.web`
B) `terraform apply -target=azurerm_linux_virtual_machine.web`
C) `terraform apply -resource=azurerm_linux_virtual_machine.web`
D) `terraform apply azurerm_linux_virtual_machine.web`

<details>
<summary>Show Answer</summary>
Answer: **B** - Use `-target` flag to apply operations to specific resources. The syntax is `terraform apply -target=resource_address`.
</details>

---

### Question 2
What is the correct import command syntax?
A) `terraform import infrastructure_id resource_address`
B) `terraform import resource_address infrastructure_id`
C) `terraform import -resource=resource_address infrastructure_id`
D) `terraform import resource_address -id=infrastructure_id`

<details>
<summary>Show Answer</summary>
Answer: **B** - The syntax is `terraform import resource_address infrastructure_id`. The resource address (terraform address) comes first, then the actual infrastructure ID (azure resource).
</details>

---

### Question 3
After importing a resource, `terraform plan` shows many changes. What should you do?
A) Run `terraform apply` immediately
B) Delete the imported resource and recreate it
C) Update your configuration to match the actual resource attributes
D) Ignore the changes

<details>
<summary>Show Answer</summary>
Answer: **C** - After import, you should review `terraform state show` to see actual attributes, then update your configuration to match. This prevents unwanted changes on the next apply.
</details>

---

## 5. Key Takeaways

- **Targeting**: Use `-target` to operate on specific resources. Syntax: `terraform plan -target=resource_address`.
- **Import**: Brings existing infrastructure under Terraform management. Syntax: `terraform import resource_address infrastructure_id`.
- **Import process**: Add resource block → import → verify state → update config → apply.
- **Targeting limitations**: Doesn't automatically resolve all dependencies - may need to target dependencies manually.
- **Configuration matching**: After import, update your `.tf` file to match actual resource attributes.
- **Dependencies**: Import dependent resources (network security groups, Vnets) before resources that depend on them.
- **Verification**: Always run `terraform plan` after import to identify configuration mismatches.

---

## References

- [Terraform Resource Targeting](https://developer.hashicorp.com/terraform/cli/commands/plan#resource-targeting)
- [Terraform Import](https://developer.hashicorp.com/terraform/cli/commands/import)
- [Import Command](https://developer.hashicorp.com/terraform/cli/commands/import)
