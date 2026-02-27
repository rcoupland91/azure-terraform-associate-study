# State Management

## Learning Objectives
- Understand what Terraform state is and why it’s critical.
- Learn how to configure **remote state** storage using Blob Storage in Azure
- Explore best practices for securing and managing state files.
- Understand **state locking**, **migration**, and **drift detection**.

---

## 🧩 1. What Is Terraform State?

Terraform is declarative — it describes *desired infrastructure*.  
To track what’s already deployed, it maintains a **state file** (`terraform.tfstate`).

The state file:
- Maps Terraform resources → real-world infrastructure (IDs, ARNs, IPs)
- Stores attributes, dependencies, and metadata
- Enables drift detection during `terraform plan`

Without it, Terraform wouldn’t know what exists and could recreate resources unnecessarily.

**Analogy:**  
Terraform state = Terraform’s “memory” or “etcd” (like Kubernetes).  
Lose it, and Terraform forgets your infrastructure.

### You must be able to say this cleanly in interviews

“Terraform state maps real infrastructure to the configuration by storing IDs, attributes, and metadata Terraform needs to plan updates.”

“Terraform uses state to determine the difference between desired config and real infrastructure.”

---

## 🗂️ 2. Local vs Remote State

### Local State
- Default behavior (file saved in your working directory)
- Fine for single-user testing
- Risks: lost state, drift, no collaboration, secrets exposure

### Remote State
- Stored in a shared backend (Azure Blob Storage, Terraform Cloud, etc.)
- Enables team collaboration
- Secures access, adds locking, and provides durability

---

## ☁️ 3. Using Azure for Remote State

### Step 1: Creating the backend resources

```bash
az group create -n tfstate-rg -l uksouth

az storage account create -n tfstateaccount01 -g tfstate-rg -l uksouth --sku Standard_LRS

az storage container create --account-name tfstateaccount01 -n tfstate

## Azure automatically encrypts data at rest
```


### Step 2: Configure the backend
```bash
terraform {
  backend "azurerm" {
    resource_group_name   = "tfstate-rg"
    storage_account_name  = "tfstateaccount01"
    container_name        = "tfstate"
    key                   = "dev/network/terraform.tfstate"
  }
}

```
Then run: 
```
terraform init
```
Terraform will ask to migrate local state → remote backend.

---

## 4. State Locking

To prevent two people running terraform apply simultaneously:

- Azure uses Blob Leases for locking.
- When Terraform runs, it acquires a lease on the .tfstate blob.
- If locked, you’ll see:

```
Error acquiring state lock
```

To unlock:

```
terraform force-unlock <lock-id>
```
Used carefully — only when you KNOW no process is running.

---

## 5. Security & Best Practices

| Practice                               | Purpose                                        |
| -------------------------------------- | ---------------------------------------------- |
| Restrict  access                       | Only admins/CI should access the blob container  |
| Enable Storage Account versioning      | Recover corrupted or deleted state             |
| Use `prevent_destroy` lifecycle rule   | Avoid accidental destruction of critical infra |
| Never commit `.tfstate` files          | They may contain secrets                       |

---

## 6. Important Commands

| Command                          | Description                                       |
| -------------------------------- | ------------------------------------------------- |
| `terraform state list`           | Lists all resources tracked in state              |
| `terraform state show <addr>`    | Shows attributes of a specific resource           |
| `terraform state mv <old> <new>` | Moves or renames resources inside state           |
| `terraform state rm <addr>`      | Removes a resource from state without deleting it |
| `terraform state pull`           | Retrieves current state (useful for debugging)    |

---

## 7. Drift 
Drift happens when the cloud changes WITHOUT Terraform knowing.

Examples of drift:
- Someone added tags in Azure Portal
- A resource was deleted manually
- A security group rule changed
- A load balancer got a new listener

Terraform detects drift during:
- terraform plan
- terraform apply
- terraform plan -refresh-only

Note: “Terraform will NOT fix drift automatically. It will show it in the plan.”
And if something is deleted manually? Terraform sees: “Expected → found none → MUST be recreated.”

---

## 8. Real-World Example

```
terraform {
  backend "azurerm" {
    resource_group_name   = "tfstate-rg"
    storage_account_name  = "tfstateaccount01"
    container_name        = "tfstate"
    key                   = "labs/webserver/terraform.tfstate"
  }
}


provider "azurerm" {
  features {}
}


resource "azurerm_resource_group" "main" {
  name     = "example-rg"
  location = "uksouth"
}

resource "azurerm_virtual_network" "main" {
  name                = "example-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "main" {
  name                 = "example-subnet"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "main" {
  name                = "example-nic"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_linux_virtual_machine" "web" {
  name                = "example-webserver"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.main.id
  ]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }
}

output "web_ip" {
  value = azurerm_linux_virtual_machine.web.public_ip_address
}


```
---

## 8. Key Takeaways
- State = Terraform’s source of truth for infrastructure.
- Storage account + container (locking built‑in via blob leases)
- Always secure state (encryption + RBAC).
- Never modify terraform.tfstate manually — use terraform state commands.
- Think of the state file as Terraform’s equivalent of etcd in Kubernetes.

---

## 9. Practice Questions

### Question 1
What is the primary purpose of Terraform state?
A) Store Terraform configuration files
B) Track the mapping between Terraform resources and real infrastructure
C) Cache provider plugins
D) Store variable values

<details>
<summary>Show Answer</summary>
Answer: **B** - Terraform state tracks the mapping between resources defined in code and actual infrastructure in the cloud, storing IDs, and attributes needed for management.
</details>

---

### Question 2
What does Blob Leases provide in an Storage Account backend configuration?
A) Stores the state file
B) Provides state locking to prevent concurrent modifications
C) Encrypts the state file
D) Backs up the state file

<details>
<summary>Show Answer</summary>
Answer: **B** - Blob Leases provides state locking, preventing multiple users or processes from modifying state simultaneously, which prevents corruption.
</details>

---

### Question 3
Which command should you use to move a resource from one address to another in state without recreating it?
A) `terraform state rm`
B) `terraform state mv`
C) `terraform state push`
D) `terraform state pull`

<details>
<summary>Show Answer</summary>
Answer: **B** - `terraform state mv` renames or moves resources within state without affecting the actual infrastructure, useful when refactoring code.
</details>

---

## 10. Lab Challenge
Create a remote state backend using:
1. Storage Account Container (my-terraform-lab-state)
2. Blob leases (terraform-lock-lab)
3. A simple Virtual Machine
4. Verify remote state exists by browsing to the container
5. Try intentionally applying twice to test locking





