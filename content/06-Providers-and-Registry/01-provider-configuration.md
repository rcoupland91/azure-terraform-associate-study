# Provider Configuration

## Learning Objectives
- Understand how to configure Terraform providers.
- Learn provider version constraints and `required_providers` block.
- Master provider aliases for multiple provider instances.
- Understand provider requirements and configuration precedence.

---

## 1. What is a Provider?

A **provider** is a plugin that Terraform uses to interact with APIs of cloud platforms, services, or other infrastructure systems.

**Examples:**
- `aws` - Amazon Web Services
- `azurerm` - Microsoft Azure
- `google` - Google Cloud Platform
- `null` - Utility provider (does nothing)
- `local` - Local system (files, etc.)

---

## 2. Basic Provider Configuration

### Simple Provider Block

```hcl
provider "azurerm" {
  features {}
}

```

**Key points:**
- Provider name: `azurerm`
- Applies to all resources using this provider (unless overridden)

### Common Azure Provider Arguments

```hcl
provider "azurerm" {
  resource_provider_registrations = "none"  # Azure has many resource providers (e.g., Microsoft.Compute, Microsoft.Network). By default, the AzureRM provider may try to automatically register any missing providers when you deploy resources. Setting this to "none" disables that behavior.
  alias                           = "test"  # This gives the provider configuration a name so you can reference it explicitly.
  subscription_id                 = {subID} # This tells Terraform which Azure subscription to use for this provider instance.
  features {}                               # This block is required even if empty.  It enables provider‑specific features and settings.
}
```

**Best Practice:** Use Managed Identity/Entra ID Workload Identity Federation or Azure CLI authentication, not hardcoded in Terraform files.

---

## 3. Provider Version Constraints

### Why Version Constraints Matter

Providers are constantly updated. Version constraints ensure:
- Compatibility with your code
- Predictable behavior
- Control over when to upgrade

### The `required_providers` Block

**Location:** Must be in a `terraform` block (usually in `versions.tf` or `provider.tf`)

```hcl
terraform {
  required_providers {
    azurerm = {
      source                = "hashicorp/azurerm"
      version               = "~> 4.57.0"
      configuration_aliases = [azurerm.test]
    }
  }
  backend "azurerm" {
  }
}
```

### Version Constraint Syntax

| Constraint | Meaning | Example |
|------------|---------|---------|
| `">= 1.0"` | Greater than or equal to 1.0 | `1.0`, `2.5`, `3.0` ✅ |
| `"<= 2.0"` | Less than or equal to 2.0 | `1.5`, `2.0` ✅ |
| `"~> 2.0"` | Allow >= 2.0, < 3.0 | `2.0`, `2.9` ✅, `3.0` ❌ |
| `"~> 2.1"` | Allow >= 2.1, < 3.0 | `2.1`, `2.9` ✅, `2.0`, `3.0` ❌ |
| `"= 1.5.0"` | Exactly 1.5.0 | `1.5.0` ✅, `1.5.1` ❌ |
| `"> 1.0, < 2.0"` | Between 1.0 and 2.0 | `1.5` ✅, `2.0` ❌ |
| `"!= 2.0"` | Not equal to 2.0 | `1.9`, `2.1` ✅, `2.0` ❌ |

**Most common:** `~>` (pessimistic constraint operator)

**Example:**
```hcl
version = "~> 5.0"    # Allows 5.0.0, 5.1.0, 5.9.9, but NOT 6.0.0
version = "~> 5.25"   # Allows 5.25.0, 5.25.1, but NOT 5.26.0 or 6.0.0
```

### Multiple Provider Constraints

```hcl
terraform {
  required_providers {
    azurerm = {
      source                = "hashicorp/azurerm"
      version               = "~> 4.49.0"
      configuration_aliases = [azurerm.test]
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.7.0"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.13.1"
    }
  }

}
```

---

## 4. Provider Sources

### Provider Source Format

```
<HOSTNAME>/<NAMESPACE>/<PROVIDER-NAME>
```

**Common patterns:**
- `hashicorp/azurerm` - Official HashiCorp Azure provider
- `custom-org/custom-provider` - Custom provider

### Default Registry Providers

For providers in the Terraform Registry, you can omit the full source:

```hcl
required_providers {
  azurerm = {
    version = "~> 5.0"
    # source defaults to registry.terraform.io/hashicorp/azurerm
  }
}
```

### Explicit Source (Full Format)

```hcl
required_providers {
  azurerm = {
    source  = "hashicorp/azurerm"
    version = "~> 4.49.0"
  }
}
```

### Third-Party Providers

```hcl
required_providers {
  datadog = {
    source  = "DataDog/datadog"
    version = "~> 3.0"
  }
}
```

---

## 5. Provider Aliases

### Why Use Aliases?

Use aliases when you need:
- Multiple instances of the same provider (different regions, accounts, etc.)
- Different configurations for the same provider

### Basic Alias Example

```hcl
# Default Azure provider (e.g., test subscription)
provider "azurerm" {
  features {}
  subscription_id = var.test_subscription_id
}

# Aliased provider for a second subscription (e.g., ukwest)
provider "azurerm" {
  alias           = "secondary"
  features        = {}
  subscription_id = var.secondary_subscription_id
}


# Use default provider
resource "azurerm_resource_group" "core_rg" {
  name     = "rg-core"
  location = "uksouth"
  # Uses default provider (core subscription)
}


# Use aliased provider
resource "azurerm_resource_group" "secondary_rg" {
  provider = azurerm.secondary

  name     = "rg-secondary"
  location = "ukwest"
  # Uses the aliased provider (secondary subscription)
}

```

### Multiple Provider Instances

```hcl
# Main account
# Default Azure provider (main subscription)
provider "azurerm" {
  features {}
  subscription_id = var.main_subscription_id
  tenant_id       = var.tenant_id
}

# Different Azure subscription
provider "azurerm" {
  alias           = "dev_sub"
  features        = {}
  subscription_id = var.dev_subscription_id
  tenant_id       = var.tenant_id
}

# Same subscription, different region
provider "azurerm" {
  alias           = "eu_region"
  features        = {}
  subscription_id = var.main_subscription_id
  tenant_id       = var.tenant_id
}


resource "azurerm_resource_group" "main" {
  name     = "rg-main"
  location = "uksouth"
  # Uses default provider (main subscription)
}


resource "azurerm_resource_group" "dev" {
  provider = azurerm.dev_sub

  name     = "rg-dev"
  location = "uksouth"
  # Uses dev subscription
}


resource "azurerm_resource_group" "europe" {
  provider = azurerm.eu_region

  name     = "rg-europe"
  location = "westeurope"
  # Uses main subscription but deploys in EU region
}
```

### Alias in Modules

**Root module:**
```hcl
# Default Azure provider
provider "azurerm" {
  features {}
  subscription_id = var.main_subscription_id
}

# Aliased provider for another subscription
provider "azurerm" {
  alias           = "dev_sub"
  features        = {}
  subscription_id = var.dev_subscription_id
}

# Aliased provider for same subscription but different region (optional)
provider "azurerm" {
  alias           = "eu_region"
  features        = {}
  subscription_id = var.main_subscription_id
}

```

**Child module (`modules/vnet/main.tf`):**
```hcl
# Declare the provider so the module can accept one from the root
provider "azurerm" {}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-main"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = ["10.0.0.0/16"]
}

```

---

## 6. Provider Configuration Precedence

When the same provider is configured multiple times, Terraform uses this order (highest → lowest):

1. **Provider argument in resource block** (`provider = azurerm.dev_sub`)
2. **Provider alias in module block** (`providers = {azurerm = azurerm.dev_sub}`)
3. **Default provider configuration** (non-aliased provider)
4. **Environment variables** (e.g., `ARM_SUBSCRIPTION_ID`)
5. **Azure CLI authentication** (`az login`)

---

## 7. Implicit Provider Configuration

### Default Behavior

If you don't specify a provider, Terraform uses the **default (non-aliased) provider**:

```hcl
provider "azurerm" {
  subscription_id = var.main_subscription_id
  features {}
}

# This resource uses the default provider above
resource "azurerm_key_vault" "kv" {
  name           = "kv-test"
  resource_group_name = "kv-rg"
}
```

### Explicit Provider Reference

```hcl
provider "azurerm" {
  subscription_id = var.main_subscription_id
  features {}
  alias = "uksouth"
}

# Explicitly use aliased provider
resource "azurerm_key_vault" "kv" {
  provider = azurerm.uksouth
  
  name           = "kv-test"
  resource_group_name = "kv-rg"
}
```

---

## 8. Provider Requirements in Modules

### Passing Providers to Modules

**Parent module:**
```hcl
provider "azurerm" {
  subscription_id = var.main_subscription_id
  features {}
}

provider "azurerm" {
  alias  = "west"
  subscription_id = var.west_subscription_id
}

module "multi_region" {
  source = "./modules/vpc"
  
  providers = {
    azurerm         = azurerm          # Pass default provider
    azurerm.west    = azurerm.west     # Pass aliased provider
  }
}
```

**Child module (`modules/vnet/main.tf`):**
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.49.0"
    }
  }
}

# Configuration Requirements block (Terraform 0.13+)
terraform {
  required_providers {
    azurerm = {
      source                = "hashicorp/azurerm"
      version               = "~> 5.0"
      configuration_aliases = [azurerm.west]  # Declare required alias
    }
  }
}
```

---

## 9. Provider Configuration Best Practices

### ✅ Do's

1. **Use `required_providers` block:**
   ```hcl
   terraform {
     required_providers {
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~> 4.49.0"
       }
     }
   }
   ```

2. **Use version constraints:**
   - `~> 4.49.0 for patch/minor updates
   - Pin exact version in production if needed

3. **Store credentials securely:**
   - Never commit credentials to version control

4. **Use aliases for multiple configurations:**
   - Different subscriptions
   - Different authentication methods

### ❌ Don'ts

1. **Don't hardcode credentials:**
   ```hcl
   # ❌ BAD
   provider "azurerm" {
  client_id     = "5b432b33-a8c6-4c34-99a1-26916d4c65b6"
  client_secret = "secret"
  tenant_id     = "185562ad-39bc-4840-8e40-be6216340c52"
   }
   ```

2. **Don't omit version constraints:**
   - Can cause unexpected breaking changes
   - Makes upgrades unpredictable

3. **Don't use `*` for version:**
   ```hcl
   # ❌ BAD
   version = "*"
   ```

---

## 10. Exam-Style Practice Questions

### Question 1
What does the version constraint `"~> 5.0"` mean?
A) Exactly version 5.0
B) Greater than or equal to 5.0, less than 6.0
C) Any version starting with 5
D) Latest version 5.x

<details>
<summary>Show Answer</summary>
Answer: **B** - `~>` (pessimistic constraint) allows >= 5.0.0 and < 6.0.0, so it accepts patch and minor updates but not major version changes.
</details>

---

### Question 2
How do you use a provider alias in a resource?
A) `alias = azurerm.west`
B) `provider = azurerm.west`
C) `use_provider = azurerm.west`
D) `provider_alias = azurerm.west`

<details>
<summary>Show Answer</summary>
Answer: **B** - Use `provider = azurerm.west` to reference an aliased provider in a resource block.
</details>

---

### Question 3
Where must the `required_providers` block be placed?
A) In the provider block
B) In a terraform block
C) In variables.tf
D) Anywhere in the configuration

<details>
<summary>Show Answer</summary>
Answer: **B** - `required_providers` must be inside a `terraform` block, typically in `versions.tf` or `provider.tf`.
</details>

---

### Question 4
What is the default provider source for `hashicorp/azurerm`?
A) `hashicorp/azurerm`
B) `registry.terraform.io/hashicorp/azurerm`
C) `terraform.io/hashicorp/azurerm`
D) No default, must specify

<details>
<summary>Show Answer</summary>
Answer: **B** - The full source is `registry.terraform.io/hashicorp/azurerm`, but you can omit the full path for registry providers and just use `hashicorp/azurerm`.
</details>

---

### Question 5
You need to create resources in multiple Azure subscriptions. What's the best approach?
A) Multiple provider blocks with different subscription values
B) Provider aliases
C) Use the same provider and change subscription in each resource
D) Create separate Terraform configurations

<details>
<summary>Show Answer</summary>
Answer: **B** - Use provider aliases to define multiple provider instances (e.g., `azurerm.uksouth`, `azurerm.ukwest`) and reference them with `provider = azurerm.ukwest` in resources.
</details>

---

## 11. Common Patterns

### Pattern 1: Multi-Region Setup

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
  }
}

provider "azurerm" {
  subscription_id = var.main_subscription_id
  features {}
}

provider "azurerm" {
  alias  = "west"
  region = var.west_subscription_id
}

resource "azurerm_key_vault" "primary" {
  name = "primary-kv"
}

resource "azurerm_key_vault" "backup" {
  provider = azurerm.west
  bucket   = "backup-kv"
}
```

### Pattern 2: Version Pinning

```hcl
terraform {
  required_version = ">= 1.12"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "5.25.0"  # Exact version for stability
    }
  }
}
```

### Pattern 3: Flexible Version Range

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 4.0, < 6.0"  # Allow 4.x and 5.x
    }
  }
}
```

---

## 12. Key Takeaways

- **`required_providers`**: Declares provider requirements in a `terraform` block.
- **Version constraints**: Use `~>` for pessimistic constraints (allows patch/minor, not major).
- **Provider aliases**: Use `alias` to create multiple provider instances for different configs.
- **Provider reference**: Use `provider = azurerm.alias_name` in resources to use aliased providers.
- **Source format**: `registry.terraform.io/namespace/provider-name` (can omit registry for official providers).
- **Never hardcode credentials**: Use environment variables or Azure CLI configuration.
- **Module providers**: Pass providers to modules using `providers = { azurerm = azurerm.west }` block.

---

## References

- [Terraform Provider Requirements](https://developer.hashicorp.com/terraform/language/providers/requirements)
- [Provider Configuration](https://developer.hashicorp.com/terraform/language/providers/configuration)
- [Provider Aliases](https://developer.hashicorp.com/terraform/language/providers/configuration#alias-multiple-provider-instances)
- [Version Constraint Syntax](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)

