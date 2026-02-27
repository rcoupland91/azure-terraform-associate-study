# Variables And Outputs

## Learning Objectives
- Understand how **variables** make Terraform configurations reusable and dynamic.
- Learn the different **variable types**, **defaults**, and **precedence**.
- Use **outputs** to share data between resources, modules, and users.
- Follow best practices for sensitive data, organization, and documentation.

---

## 🧩 1. Why Use Variables?

Hardcoding values (like instance types or AMI IDs) limits flexibility and reusability.  
Variables allow Terraform to behave like a *parameterized template*, making it easy to:
- Reuse configurations across environments (dev, test, prod)
- Avoid duplication
- Inject values dynamically (from CLI, files, or pipelines)

---

## 2. Declaring Variables

Variables are typically defined in a file named `variables.tf`:

```hcl
variable "account_tier" {
  type        = string
  default     = "Standard"
}
```

You reference a variable in code with the prefix var.:

```
resource "azurerm_storage_account" "data" {
  name                     = "test-sa"
  resource_group_name      = "test-rg"
  location                 = "westeurope"
  account_tier             = var.account_tier
  account_replication_type = "LRS"
}
```

If no default is provided, Terraform will prompt you during plan or apply.

---

## 3. Variable Types

Terraform supports multiple data types.

| Type             | Example                                    | Description                |
| ---------------- | ------------------------------------------ | -------------------------- |
| **string**       | `"Standard"`                               | Text value                 |
| **number**       | `3`                                        | Integer or float           |
| **bool**         | `true`                                     | Boolean value              |
| **list(string)** | `["dev", "test", "prod"]`                  | Ordered list of strings    |
| **map(string)**  | `{ region = "uksouth", env = "dev" }`      | Key/value pairs            |
| **object**       | `object({ name = string, size = number })` | Complex structure          |
| **tuple**        | `[true, "Standard", 2]`                    | Fixed collection of values |
| **set(string)**  | `set(["Standard", "Premium"])`             | Unique, unordered values   |

---

## 4. Variable Precedence 
Terraform reads variables from multiple sources.
If the same variable is defined in multiple places, the following precedence order applies (highest → lowest):

| Source                           | Example                                   | Priority   |
| -------------------------------- | ----------------------------------------- | ---------- |
| CLI flags                        | `terraform apply -var="region=uksouth"`   | 🥇 Highest |
| `.tfvars` file passed explicitly | `terraform apply -var-file=prod.tfvars`   |            |
| Environment variables            | `export TF_VAR_region=uksouth`            |            |
| Auto-loaded files                | `*.auto.tfvars`                           |            |
| `terraform.tfvars`               | Auto-loaded default file                  |            |
| Variable default in code         | `default = "uksouth"`                     | 🥉 Lowest  |

Example question (exam-style):

A variable is defined in terraform.tfvars, as an environment variable, and in the variable block with a default. Which value will Terraform use?

Answer: The environment variable (TF_VAR) value.

---

## 5. Sensitive Variable (Credentials/Passwords)

Mark sensitive variables to prevent Terraform from displaying them in logs or outputs with the sensitive = true tag. 
```
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}
```
Sensitive values still exist in memory and state, but they’ll be redacted in CLI output:
```
Outputs:

db_password = (sensitive value)
```
For true secrecy, store them securely (Azure Key Vault, Devops Variable groups, etc.) and retrieve with data sources.

---

## 6. Variable Files (.tfvars)
Instead of defining values interactively, use a .tfvars file.

terraform.tfvars:
```
region         = "uksouth"
account_tier   = "Standard"
environment    = "dev"
```
Then apply: 
```bash
terraform apply

# or specify custom files:
terraform apply -var-file=prod.tfvars
```
This keeps configurations clean and environment-specific.

---

## 7. Outputs
Outputs expose information after Terraform creates resources — for example, the public IP of an instance or the ARN of a resource.
```
output "virtual_machine_ip" {
  description = "IP of the Virtual Machine"
  value       = azurerm_linux_virtual_machine.web.private_ip_address
}
```
Run: 
```
terraform output
terraform output virtual_machine_ip
```
Marking Outputs as Sensitive
```
output "db_password" {
  value     = azurerm_mssql_managed_instance.main.password
  sensitive = true
}
```
This hides it in CLI output but still stores it in state.

---

## 8. Sharing Data Between Modules

Outputs are also how modules pass data between each other.

Child module vnet/outputs.tf:
```
output "subnet_id" {
  value = azurerm_subnet.private.id
}
```
Root Module (main.tf) :
```
module "vnet" {
  source = "./vnet"
}

resource "azurerm_private_endpoint" "example" {
  name                = "my-pep"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = module.vnet.subnet_id

  private_service_connection {
    name                           = "myConnection"
    private_connection_resource_id = azurerm_storage_account.example.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}

```

---

## 9. Practical Example 
variables.tf
```
variable "account_tier" {
  type        = string
  default     = "Standard"
}
```
main.tf
```
provider "azurerm" {
  region = "uksouth"
}

resource "azurerm_storage_account" "data" {
  name                     = "test-sa"
  resource_group_name      = "test-rg"
  location                 = "uksouth"
  account_tier             = var.account_tier
  account_replication_type = "LRS"
}

output "storage_account_id" {
  value       = azurerm_storage_account.web.id
  description = "Storage Accounts ID"
}
```
Run: 
```bash
terraform apply -var="account_tier=Standard"
```

**Note:** Resource IDs in examples are placeholders. In real deployments, use ` data sources to fetch the latest Resource IDs.
---

## 10. Best Practices

| Practice                                          | Reason                                              |
| ------------------------------------------------- | --------------------------------------------------- |
| Group variables logically in `variables.tf`       | Improves readability                                |
| Use clear names and descriptions                  | Easier for teams to maintain                        |
| Use `.tfvars` files for environment-specific data | Keeps configs clean                                 |
| Never store secrets in plain `.tfvars` or code    | Use Azure Key Vaults, Devops Variables, etc         |
| Use outputs sparingly                             | Only expose what's necessary                        |
| Combine outputs with `sensitive = true`           | Hide confidential info                              |

---

## 11. Practice Questions

### Question 1
A variable is defined in `terraform.tfvars`, as an environment variable `TF_VAR_region`, and in the variable block with a default value. Which value will Terraform use?
A) The default value
B) The value from terraform.tfvars
C) The environment variable value
D) Terraform will prompt for input

<details>
<summary>Show Answer</summary>
Answer: **C** - Environment variables (TF_VAR_*) have higher precedence than .tfvars files and defaults. Precedence order: CLI flags > .tfvars > environment variables > defaults.
</details>

---

### Question 2
What does `sensitive = true` do for a variable?
A) Encrypts the value in the state file
B) Prevents the value from being stored in state
C) Hides the value from CLI output but still stores it in state
D) Requires the value to be provided via secret manager

<details>
<summary>Show Answer</summary>
Answer: **C** - `sensitive = true` redacts the value from Terraform CLI output and logs, but the value is still stored in state. For true security, use external secret management.
</details>

---

### Question 3
How do you reference a module's output value in the root module?
A) `module.<module-name>.<output-name>`
B) `output.<module-name>.<output-name>`
C) `var.<module-name>.<output-name>`
D) `module.<module-name>.output.<output-name>`

<details>
<summary>Show Answer</summary>
Answer: **A** - Module outputs are accessed using `module.<module-name>.<output-name>`. For example, `module.vpc.vpc_id` gets the vpc_id output from the vpc module.
</details>

---

## 12. Key Takeaways

- Variables make Terraform reusable, modular, and environment-friendly.
- Precedence determines which variable value Terraform uses at runtime.
- Sensitive variables mask output but still exist in state — protect the state file.
- Outputs help share data between resources, modules, and users.
- Together, variables + outputs make your Terraform code maintainable and scalable.







