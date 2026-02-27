# Modules And Backends

## Learning Objectives
- Understand what **modules** are and how they make Terraform configurations reusable.
- Learn how to structure and call **child modules**.
- Explore the purpose and configuration of **backends** for storing state remotely.
- Apply best practices for using modules and managing backend configurations securely.

---

## 1. What Are Terraform Modules?

A **module** is a collection of Terraform configuration files in a folder.  
Modules let you group related resources together and reuse them across multiple projects or environments.

You already use one module every time you run Terraform — the **root module** (your main working directory).

---

## 2. Why Use Modules?

Modules help you:
- Reuse and standardize infrastructure patterns  
- Avoid repetitive code  
- Simplify environment management (dev, stage, prod)  
- Share architecture blueprints with teams

**Example use case:**  
Instead of writing VNET, Subnets, and network security group resources multiple times, you define them once in a module and call it wherever needed.

---

## 3. Module Structure

A simple reusable module usually includes:

```
my-vnet-module/
├── main.tf          # defines the actual resources
├── variables.tf     # defines inputs
├── outputs.tf       # defines what values get returned
└── README.md        # explains usage
```

Then your root module (where you run Terraform) can call it like this:

```hcl
module "vnet" {
  source = "./modules/my-vnet-module"
  cidr_block = "10.0.0.0/16"
  env = "dev"
}
```

---

## 4. Module Sources

Terraform can load modules from multiple sources:

| Source Type            | Example                                                                                              | Use Case                 |
| ---------------------- | -----------------------------------------------------------------------------------------------------| ------------------------ |
| **Local path**         | `source = "./modules/vnet"`                                                                          | Within your project      |
| **Git repo**           | `source = "git::https://github.com/TerminalsandCoffee/terraform-vpc.git"`                            | Shared code or team repo |
| **Terraform Registry** | `source = "terraform-azurerm-avm-res-network-virtualnetwork"`                                        | Public community modules |
| **Storage Account**    | `source = "azurerm://<storage-account-name>.blob.core.windows.net/<container>/<path-to-zip>"`        | Private hosted modules   |

---

## 5. Passing Variables into Modules

Modules use **input variables** to accept configuration values from the caller.

**Inside the module (`variables.tf`):**

```hcl
variable "cidr_block" {
  type        = string
  description = "vnet CIDR range"
}

variable "env" {
  type        = string
  default     = "dev"
}
```

**In the root module (where module is called):**

```hcl
module "vnet" {
  source      = "./modules/vnet"
  cidr_block  = "10.0.0.0/16"
  env         = "prod"
}
```

---

## 6. Returning Outputs from Modules

Modules can export values (like resource IDs) to the root module.

**Inside the module (`outputs.tf`):**

```hcl
output "vnet_id" { 
  description = "ID of the created VNet" 
  value = azurerm_virtual_network.main.id 
  }
```

**In the root module:**

```hcl
output "vnet_id" {
  value = module.vnet.vnet_id
}
```

Run:

```bash
terraform output vnet_id
```

---

## 7. Module Example (End-to-End)

**Folder structure:**

```
azure-terraform-associate-study/
├── main.tf
└── modules/
    └── storage_account/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**storage_account/main.tf:**

```hcl
resource "azurerm_storage_account" "example" {

  name                      = var.sa_name
  resource_group_name       = "test-rg"
  location                  = var.location
  account_tier              = "Standard"
  account_replication_type  = each.value.replication_type
  shared_access_key_enabled = false
}
```

**storage_account/variables.tf:**

```hcl
variable "location" {
  type        = string
  description = "Location of resource"
}

variable "sa_name" {
  type        = string
  description = "SA Name"
}
```

**storage_account/outputs.tf:**

```hcl
output "sa_name" {
  value       = azurerm_storage_account.example.name
  description = "Name of the Storage Account"
}
```

**main.tf (root):**

```hcl
provider "azurerm" {
  region = "uksouth"
}

module "storage" {
  source      = "./modules/storage_account"
  sa_name = "test-sa"
  location = "uksouth"
  env         = "dev"
}

output "sa_name" {
  value = module.storage.sa_name
}
```

Run:

```bash
terraform init
terraform apply -auto-approve
```

---

## 8. What Is a Backend?

A **backend** defines *where and how Terraform stores its state*.
It determines how state is loaded, stored, and locked during operations.

Without a backend, Terraform defaults to a **local backend** — `terraform.tfstate` in your working directory.

You can configure a **remote backend** for shared, secure state (e.g., Azure Storage Containers, Terraform Cloud).

---

## 9. Backend Example (Storage Account/Container)

```hcl
terraform {
  backend "azurerm" {
    resource_group_name   = "my-tfstate-rg"
    storage_account_name  = "mytfstateaccount"
    container_name        = "tfstate"
    key                   = "network/terraform.tfstate"
  }
}

```

### Key points:

* `container_name`: where the state file is stored
* `key`: Path to the state file

Run:

```bash
terraform init
```

Terraform will migrate or set up your remote state automatically.

---

## 10. Backend Migration

If you initially used a **local state**, and then add a backend block, Terraform will detect the change and prompt you:

> "Do you want to copy your existing state to the new backend?"

Always say **yes** to migrate the existing resources safely.

You can also migrate manually with:

```bash
terraform init -migrate-state
```

---

## 11. Backend Types

Common backend types and their use cases:

| Backend                          | Use Case                                       | Supports Locking?                         ` |
| -------------------------------- | ---------------------------------------------- | ------------------------------------------  |
| **local**                        | Simple, single-user local files                | ❌ No                                       | 
| **s3**                           | Team collaboration via AWS S3 + DynamoDB       | ✅ Yes                                      |
| **terraform cloud / enterprise** | Managed remote state with workspace automation | ✅ Yes                                      |
| **azurerm**                      | Platform-specific remote storage               | ✅ Yes, with Azure Blob Storage leases      |

---

## 12. Important Backend Facts for the Exam

* Backends are configured in the **terraform block**, not the provider block.
* Backend settings **can’t use variables** or `count/for_each`.
* `terraform init` is required after any backend change.
* Backends store **only state**, not configuration.
* Terraform never includes backend credentials in state files.

---

## 13. Best Practices

| Practice                                 | Reason                                    |
| ---------------------------------------- | ----------------------------------------- |
| Store reusable logic in modules          | Promotes DRY (Don’t Repeat Yourself)      |
| Use remote state backends                | Enable team collaboration and persistence |
| Version control your modules             | Track changes safely                      |
| Keep backend config minimal              | Avoid embedding secrets in backend blocks |
| Use consistent naming for module folders | Improves readability                      |
| Test modules independently               | Validate logic before reuse               |

---

## 14. Practice Questions

### Question 1
Which module source type would you use to load a module from a Git repository?
A) `source = "./modules/vnet"`
B) `source = "git::https://github.com/user/module.git"`
C) `source = "terraform-azurerm-modules/vnet/azurerm"`
D) `source = "modules/vnet"`

<details>
<summary>Show Answer</summary>
Answer: **B** - Git repositories use the `git::` protocol prefix. Option A is a local path, C is the Terraform Registry format, and D is invalid.
</details>

---

### Question 2
Can you use variables in the backend configuration block?
A) Yes, any variable type
B) Yes, but only string variables
C) No, backend configuration is static and cannot use variables
D) Only in certain backends

<details>
<summary>Show Answer</summary>
Answer: **C** - Backend configuration blocks cannot use variables, functions, or any expressions. They must be static values. Use `-backend-config` flag or backend config files for dynamic values.
</details>

---

### Question 3
What is the purpose of the `required_providers` block in a module?
A) Configure provider credentials
B) Declare which providers and versions the module needs
C) Set provider region
D) Define provider aliases

<details>
<summary>Show Answer</summary>
Answer: **B** - The `required_providers` block declares provider dependencies and version constraints. It doesn't configure credentials or settings, which are done in provider blocks.
</details>

---

## 15. Key Takeaways

* **Modules** group resources and make Terraform reusable, maintainable, and modular.
* **Inputs** pass data *into* a module; **outputs** pass data *out*.
* **Backends** control *where* Terraform stores state (local vs remote).
* **Remote backends (Storage Account/Blob Storage)** provide collaboration, encryption, and locking.
* Backend config is static — cannot depend on variables.
* Together, modules and backends form the foundation of scalable Terraform architecture.

---

## References

* [Terraform Modules Documentation](https://developer.hashicorp.com/terraform/language/modules)
* [Terraform Backends Documentation](https://developer.hashicorp.com/terraform/language/settings/backends)
* [Terraform Registry - Azure Modules](https://registry.terraform.io/namespaces/Azure)
* [Azure Backend Example (Blob Storage)](https://developer.hashicorp.com/terraform/language/backend/azurerm)
