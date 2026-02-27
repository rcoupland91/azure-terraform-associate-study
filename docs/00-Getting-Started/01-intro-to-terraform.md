# Introduction to Terraform

Terraform is an open-source **Infrastructure as Code (IaC)** tool developed by **HashiCorp**. It lets you define, provision, and manage infrastructure using simple configuration files instead of clicking through cloud consoles.

Terraform works across multiple cloud providers, but in this repository, the main focus is on **Azure**, since it’s the most widely used platform in real-world DevOps and Cloud Engineering projects.

This document provides a practical introduction to Terraform — what it is, how it works, and why it’s essential for anyone building and managing modern cloud infrastructure.

---

## Why Infrastructure as Code Matters

**Infrastructure as Code (IaC)** allows engineers to manage infrastructure the same way developers manage application code — using files that can be versioned, tested, and deployed automatically.

**Key Benefits of IaC:**

* **Automation:** Removes manual setup and configuration, reducing human error.
* **Consistency:** Reproducible environments every time (“it works on my machine” no longer applies).
* **Version Control:** Track and roll back infrastructure changes through Git just like source code.
* **Cost and Time Savings:** Less time managing servers manually → more time building reliable systems.

---

## Terraform Core Workflow

Terraform follows a straightforward workflow:

1. **Write:** Define your infrastructure in `.tf` files using the **HashiCorp Configuration Language (HCL)**.
2. **Initialize:** Download required providers and modules.

   ```bash
   terraform init
   ```
3. **Plan:** Preview what Terraform will change before applying.

   ```bash
   terraform plan
   ```
4. **Apply:** Build or update your infrastructure to match the configuration.

   ```bash
   terraform apply
   ```
5. **Destroy:** Clean up when you’re done.

   ```bash
   terraform destroy
   ```

This workflow makes Terraform a powerful tool for both production infrastructure and lab or sandbox automation.

---

## Managing Azure Services with Terraform

Terraform connects to Azure through the **AzureRM provider** — a plugin that exposes all Azure resources (Virtual machines, Storage Accounts, VNETs, Keyvaults, etc.) to Terraform.

Example provider setup:

```hcl
provider "azurerm" {
 resource_provider_registrations = "none"
 subscription_id                 = local.subscription_id
 tenant_id                       = var.tenant_id
 features {}
}
```

**Example: Provisioning a Keyvault Instance**

```hcl
resource "azurerm_key_vault" "example" {
  name                        = "examplekeyvault"
  location                    = "uksouth"
  resource_group_name         = "testresourcegroup"
  tenant_id                   = "29334838934093843894328" #test tenant ID
}
```

This small configuration defines and deploys a basic keyvault instance automatically — no console clicks required.

---

## Azure Resources Managed with Terraform

* Virtual Machines
* Storage Accounts
* Key Vaults
* VNETs and Subnets
* NSGs
* Function Apps
* SQL MI
* Frontdoor/Application Gateways

---

## Terraform Configuration Language (HCL) Basics

Terraform’s **HCL** (HashiCorp Configuration Language) is declarative — you describe *what* you want, and Terraform figures out *how* to get there.

**Core building blocks:**

* **Provider:** Connects Terraform to a cloud or platform (like AzureRM).
* **Resource:** Represents infrastructure components (e.g., VM, Storage Account).
* **Variable:** Adds flexibility by letting you reuse values.
* **Output:** Displays key information after deployment (e.g., public IPs).

Example with a variable:

```hcl
variable "location" {
  description = "Location of Keyvault"
  default     = "uksouth"
}

resource "azurerm_key_vault" "example" {
  name                        = "examplekeyvault"
  location                    = var.location
  resource_group_name         = "testresourcegroup"
  tenant_id                   = "29334838934093843894328" #test tenant ID
}
```

---

## File Structure Overview

A typical Terraform configuration includes:

* **`provider.tf`** – provider setup (AzureRM, etc.)
* **`variables.tf`** – declared variables
* **`main.tf`** – main infrastructure resources
* **`outputs.tf`** – key output values (IP addresses, URLs, etc.)

---

## Practice Questions

### Question 1
What is Infrastructure as Code (IaC)?
A) Writing infrastructure documentation in code format
B) Managing infrastructure through configuration files instead of manual processes
C) Using code to test infrastructure
D) Converting infrastructure diagrams to code

<details>
<summary>Show Answer</summary>
Answer: **B** - IaC is the practice of managing infrastructure through declarative configuration files that can be versioned, tested, and automated, rather than manual console clicks or scripts.
</details>

---

### Question 2
Which Terraform command downloads provider plugins and initializes the backend?
A) `terraform plan`
B) `terraform apply`
C) `terraform init`
D) `terraform validate`

<details>
<summary>Show Answer</summary>
Answer: **C** - `terraform init` initializes the working directory, downloads providers, sets up the backend, and installs modules.
</details>

---

### Question 3
What does HCL stand for in Terraform?
A) HashiCorp Command Language
B) HashiCorp Configuration Language
C) HashiCorp Cloud Language
D) HashiCorp Coding Language

<details>
<summary>Show Answer</summary>
Answer: **B** - HCL stands for HashiCorp Configuration Language, the declarative language used to write Terraform configurations.
</details>

---

## Conclusion

Terraform bridges the gap between infrastructure and automation.
It helps teams move faster, stay consistent, and version everything from compute to networking.

In this repo, you'll find guided labs and examples — starting simple and progressing toward advanced automation with **workspaces, backends, and CI/CD integration** — all built around real Azure use cases.

