# 15 - Ephemeral Values and Write-Only Arguments

## Learning Objectives
- Understand what ephemeral values are and why they shouldn't be stored in state.
- Learn about write-only arguments in Terraform resources.
- Master best practices for handling sensitive data that shouldn't persist.
- Apply these concepts to real-world Azure resource scenarios.

---

## 1. Overview: Ephemeral Values vs Persistent State

Terraform state stores resource attributes to track infrastructure. However, some values should **never** be stored in state:

- **Ephemeral Values**: Data that changes frequently or shouldn't be tracked (e.g., temporary tokens, session data)
- **Write-Only Arguments**: Resource attributes that can be set but never read back (e.g., passwords, secrets)

Understanding these concepts is critical for security and proper Terraform usage.

---

## 2. What Are Ephemeral Values?

### Definition

**Ephemeral values** are resource attributes that:
- Change frequently or are temporary
- Should not be stored in Terraform state
- May contain sensitive information
- Are not needed for resource management

### Examples of Ephemeral Values

- **One-time passwords** (OTP codes)
- **Session keys** (encryption keys that rotate)
- **Dynamic credentials** (RBAC Creds)
- **Time-sensitive data** (expiration timestamps)

### Why Ephemeral Values Matter

1. **Security**: Sensitive data shouldn't persist in state files
2. **State file size**: Reduces state file bloat
3. **State file security**: Limits exposure of sensitive information
4. **Best practices**: Aligns with security best practices

---

## 3. Write-Only Arguments

### Definition

**Write-only arguments** are resource attributes that:
- You provide the value (password, secret, key, certificate).
- Azure accepts it but never returns it in API responses.
- Terraform does not store the value in state.
- Terraform cannot detect drift if the value changes outside Terraform.
- You must use data sources or Key Vault to read values later.

### Common Write-Only Arguments in Azure

#### Example 1: Azure SQL Administrator Password

```hcl
resource "azurerm_mssql_server" "main" {
  name                         = "prod-sql"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.admin_password  # write-only
}

```

**Key Points:**
- Password is set during creation
- Azure doesn't return the password in API responses
- Terraform state doesn't contain the password
- If you change the password, Terraform won't detect drift

#### Example 2: Azure AD Application Client Secret

```hcl
resource "azuread_application_password" "app_secret" {
  application_object_id = azuread_application.app.id
  value                 = var.client_secret  # write-only
}

```

**Key Points:**
- Entra ID never returns the secret again.
- Terraform state stores only metadata, not the secret value.
- Password is not in state file

#### Example 3: Key Vault Secret Values (when created via resource)

```hcl
resource "azurerm_key_vault_secret" "api_key" {
  name         = "api-key"
  value        = var.api_key  # write-only
  key_vault_id = azurerm_key_vault.kv.id
}

```

**Key Points:**
- Terraform sends the secret to Azure.
- Azure stores it encrypted.
- Terraform state does not contain the plain text.

# To read the daya you must use a data source

```hcl
data "azurerm_key_vault_secret" "api_key" {
  name         = "api-key"
  key_vault_id = azurerm_key_vault.kv.id
}
```

---

## 4. Handling Ephemeral Values

### Strategy 1: Don't Store in State

For values that change frequently or are temporary:

```hcl
# ❌ BAD - Embedding a short-lived SAS token in Terraform-managed user data
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  user_data = base64encode(templatefile("init.sh", {
    # SAS token expires every hour → DO NOT store in state
    sas_token = var.storage_sas_token
  }))
}


# ✅ GOOD - Fetch the token at runtime using Managed Identity
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "web-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"
  admin_username      = "azureuser"

  identity {
    type = "SystemAssigned"
  }

  user_data = base64encode(templatefile("init.sh", {
    # VM will request a token at runtime via IMDS
    token_endpoint = "http://169.254.169.254/metadata/identity/oauth2/token"
  }))
}

```

### Strategy 2: Use External Data Sources

For values that need to be fetched but shouldn't persist:

```hcl
# Fetch a short‑lived Entra ID access token at apply time
data "external" "aad_token" {
  program = [
    "bash", "-c",
    "az account get-access-token --resource https://storage.azure.com/ --output json"
  ]
}

```

**Note:** In practice, use RBAC roles or Managed Identities instead of temporary credentials in Terraform.

### Strategy 3: Mark as Sensitive

For values that must be used but shouldn't be displayed:

```hcl
variable "sql_admin_password" {
  type        = string
  sensitive   = true
}

resource "azurerm_mssql_server" "main" {
  name                         = "prod-sql"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_admin_password
}

```

---

## 5. Best Practices for Write-Only Arguments

### ✅ Do's

1. **Use sensitive variables:**
   ```hcl
variable "sql_admin_password" {
  type      = string
  sensitive = true
}

   ```

2. **Using external secret managers (Azure Key Vault):**

# Store secret in Key Vault
# Read secret at apply time
# Use secret in a resource
   ```hcl
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = var.db_password
  key_vault_id = azurerm_key_vault.kv.id
}
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = azurerm_key_vault.kv.id
}

resource "azurerm_mssql_server" "main" {
  name                         = "prod-sql"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = data.azurerm_key_vault_secret.db_password.value
}

   ```

3. **Encrypting sensitive values (Azure AD application secrets):**

# You can output the key ID, but never the secret value.

   ```hcl
resource "azuread_application_password" "app_secret" {
  application_object_id = azuread_application.app.id
  value                 = var.client_secret  # write-only
}

   ```

4. **Document write-only arguments:**
   ```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  # admin_password is write-only
  # Azure never returns it, and Terraform cannot detect drift
  admin_password = var.admin_password
}

   ```

### ❌ Don'ts

1. **Don't try to read write-only values:**
   ```hcl
   # ❌ BAD - This won't work
    output "vm_password" {
      value = azurerm_linux_virtual_machine.vm.admin_password
}
   ```

2. **Don’t hardcode secrets*:**
   ```hcl
   # ❌ BAD
    variable "admin_password" {
     default = "Password123!"
}
   ```

3. **Don't commit secrets to version control:**
   ```hcl
   # ❌ BAD - Never commit .tfvars with secrets
   # terraform.tfvars (committed to Git)
   sql_password = "MySecretPassword"
   ```

4. **Don't assume write-only values persist:**
   ```hcl
   # ❌ BAD - Password change won't be detected
   resource "azurerm_mssql_server" "main" {
     password = var.db_password
     # If password changes in Azure Portal, Terraform won't know
   }
   ```

---

## 6. Real-World Examples

### Example 1: Azure SQL with Key Vault

```hcl
data "azurerm_key_vault_secret" "sql_password" {
  name         = "sql-admin-password"
  key_vault_id = azurerm_key_vault.kv.id
}

resource "azurerm_mssql_server" "main" {
  name                         = "prod-sql"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = data.azurerm_key_vault_secret.sql_password.value
}

```

### Example 2: Entra ID application with write‑only secret

```hcl
resource "azuread_application" "app" {
  display_name = "my-app"
}

resource "azuread_application_password" "app_secret" {
  application_object_id = azuread_application.app.id
  value                 = var.client_secret  # write-only
}

```

### Example 3: VM with write‑only admin password

```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "prod-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2s"

  admin_username = "azureuser"
  admin_password = var.admin_password  # write-only
}

```

### Example 4: Handling Password Changes

```hcl
# Problem: Azure SQL Server with ignored password drift
# If password changes outside Terraform, Terraform won't detect it
# Solution: Use lifecycle ignore_changes or manage via Key Vault

resource "azurerm_mssql_server" "main" {
  name                         = "prod-sql"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_admin_password  # write-only

  lifecycle {
    ignore_changes = [
      administrator_login_password
    ]
  }
}


# Another approach: Replace Azure SQL Server when password changes
resource "azurerm_mssql_server" "main" {
  name                         = "prod-sql"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_admin_password

  lifecycle {
    replace_triggered_by = [
      var.sql_admin_password
    ]
  }
}

```

---

## 7. Ephemeral Values in State Management

### Understanding State File Contents

```hcl
# State file contains:
{
  "resources": [
    {
      "type": "azurerm_storage_account",
      "name": "state",
      "instances": [
        {
          "attributes": {
            "id": "/subscriptions/.../storageAccounts/tfstateprod",
            "name": "tfstateprod",
            "location": "uksouth",
            "account_tier": "Standard",
            "account_replication_type": "LRS",
            "primary_blob_endpoint": "https://tfstateprod.blob.core.windows.net/",
            "primary_access_key": null,   # ❌ Not stored (write-only)
            "secondary_access_key": null  # ❌ Not stored (write-only)
          }
        }
      ]
    }
  ]
}

```

### What Gets Stored vs What Doesn't

| Attribute Type | Stored in State?  | Example |
|----------------|------------------ |---------|
| **Readable attributes**  | ✅ Yes  | `primary_blob_endpoint`, `id`, `account_tier` |
| **Write-only arguments** | ❌ No   | `password`, `primary_access_key` |
| **Sensitive ephemeral**  | ❌ No   | Temporary tokens, session keys |
| **Computed attributes**  | ✅ Yes  | `id` (after creation) |

---

## 8. Exam-Style Practice Questions

### Question 1
What is a write-only argument in Terraform?
A) An argument that can only be read, not written
B) An argument that can be set but cannot be read back from the provider
C) An argument that is optional
D) An argument that must be provided

<details>
<summary>Show Answer</summary>
Answer: **B** - Write-only arguments can be set during resource creation but cannot be read back from the provider API. Examples include passwords and secrets.
</details>

---

### Question 2
Which of the following is an example of an ephemeral value that should not be stored in Terraform state?
A) Azure Storage Account primary endpoint
B) Azure Virtual Machine ID
C) Entra ID OAuth2 access token
D) Azure Resource Group name

<details>
<summary>Show Answer</summary>
Answer: **C** - Access tokens are short‑lived, security‑sensitive, and change frequently. Terraform state should never contain temporary credentials or secrets. Storage endpoints, VM IDs, and resource group names are stable identifiers that Terraform should store.
</details>

---

### Question 3
You set a password for an RDS database instance. Can you read that password back from Terraform state?
A) Yes, it's stored in the state file
B) No, passwords are write-only arguments
C) Only if you mark it as sensitive
D) Only if you use a data source

<details>
<summary>Show Answer</summary>
Answer: **B** - RDS database passwords are write-only arguments. They can be set during creation but cannot be read back from the AWS API, and therefore are not stored in Terraform state.
</details>

---

### Question 4
What is the best practice for handling database passwords in Terraform?
A) Store them in terraform.tfvars files
B) Hardcode them in the configuration
C) Use Azure Keyvault or mark as sensitive variables
D) Store them in the state file

<details>
<summary>Show Answer</summary>
Answer: **C** - Best practice is to use Azure Keyvault to store passwords securely, or at minimum mark password variables as sensitive. Never hardcode or commit passwords to version control.
</details>

---

### Question 5
If you change an SQL database password in the Azure Portal, will Terraform detect the change?
A) Yes, Terraform will detect it during the next plan
B) No, passwords are write-only so Terraform cannot detect changes
C) Only if you use ignore_changes lifecycle rule
D) Only if the password is stored in Secrets Manager

<details>
<summary>Show Answer</summary>
Answer: **B** - Since passwords are write-only arguments, Terraform cannot read the current password value from Azure. Therefore, it cannot detect if the password was changed outside of Terraform. You would need to update the Terraform configuration and apply to change the password.
</details>

---

## 9. Key Takeaways

- **Write-only arguments**: Can be set but cannot be read back from the provider (e.g., passwords, secrets).
- **Ephemeral values**: Temporary or frequently changing data that shouldn't be stored in state (e.g., session tokens).
- **Security**: Write-only and ephemeral values are not stored in Terraform state, improving security.
- **Best practices**: 
  - Use Azure Keyvault for sensitive values
  - Mark sensitive variables with `sensitive = true`
  - Never hardcode or commit secrets to version control
- **Limitations**: Terraform cannot detect changes to write-only arguments made outside of Terraform.
- **State file**: Write-only arguments appear as `null` or are omitted from state files.
- **Drift detection**: Changes to write-only arguments won't be detected during `terraform plan`.

---

## References

- [Terraform Sensitive Variables](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)
- [AWS Secrets Manager](https://learn.microsoft.com/en-us/azure/key-vault/general/overview)
- [Terraform State Security](https://developer.hashicorp.com/terraform/language/state/sensitive-data)

