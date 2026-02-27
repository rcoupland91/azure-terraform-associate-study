# Advanced Terraform Features

## Learning Objectives
- Understand Terraform workspaces for environment management.
- Learn how to use data sources to query existing infrastructure.
- Understand provisioners and when to use them.
- Practice with workspaces, data sources, and provisioners through hands-on examples.

---

## 1. Workspaces – Managing Multiple Environments

**Concept:**  
Workspaces in Terraform allow you to use the same configuration for multiple environments (like dev, stage, and prod) without duplicating code. Each workspace maintains its own state file, meaning the same config can manage separate resources across different environments.

**Commands:**

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace list
terraform workspace select dev
```

**Example:**

```hcl
resource "azurerm_storage_container" "demo" {
  name = "demo-${terraform.workspace}-container"
}
```

When applied in each workspace:  
- `dev` → creates `demo-dev-container`  
- `prod` → creates `demo-prod-container`

**Best Practice:**  
Use workspaces for small environment differences. For major variations, use separate folders or repos.

---

## 2. Data Sources – Reading Existing Infrastructure

**Concept:**  
Data sources allow Terraform to query existing resources and reuse their attributes in your configuration. This is helpful when you want to reference infrastructure not managed by your Terraform code.

**Example:**

```hcl
data "terraform_remote_state" "network" {
  backend = "azurerm"
  config = {
    resource_group_name  = "state-rg"
    storage_account_name = "statesa"
    container_name       = "state-tfstate"
    key = "state.tfstate"

    subscription_id = {SubID}
  }
}

resource "azurerm_private_endpoint" "pe_aisearch" {
  name                = "name-pe"
  location            = var.location
  resource_group_name = "pe-rg"
  subnet_id           = data.terraform_remote_state.network.outputs.subnet_ids[<subnetnames-as-per-output>]
}

```

**Key Point:**  
Data sources are read-only and will not alter or destroy the referenced resources.

---

## 3. Provisioners – Running Commands on Resources

**Concept:**  
Provisioners let you execute scripts or commands after a resource is created. They are often used to perform initial setup tasks such as configuration, file transfers, or installing software.

**Types of Provisioners:**  
- **local-exec**: Runs on the local machine executing Terraform.  
- **remote-exec**: Runs commands on the remote resource via SSH or WinRM.

**Example: local-exec**

```hcl
resource "azurerm_linux_virtual_machine" "example" {
  name                = "example-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "azureuser"

  network_interface_ids = [
    azurerm_network_interface.example.id
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

  provisioner "local-exec" {
    command = "echo ${self.public_ip_address} >> public_ips.txt"
  }
}

```

**Example: remote-exec**

```hcl
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

resource "azurerm_network_security_group" "ssh" {
  name                = "allow-ssh"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "SSH"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_public_ip" "main" {
  name                = "example-ip"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Dynamic"
}

resource "azurerm_network_interface" "main" {
  name                = "example-nic"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.main.id
  }
}

resource "azurerm_network_interface_security_group_association" "ssh" {
  network_interface_id      = azurerm_network_interface.main.id
  network_security_group_id = azurerm_network_security_group.ssh.id
}

resource "azurerm_linux_virtual_machine" "example" {
  name                = "example-vm"
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

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update -y",
      "echo 'Hello from remote-exec' > /tmp/hello.txt"
    ]

    connection {
      type        = "ssh"
      host        = azurerm_public_ip.main.ip_address
      user        = "azureuser"
      private_key = file("~/.ssh/id_rsa")
    }
  }
}
```

**Best Practice:**  
Use provisioners sparingly and only when other options (like user_data, cloud-init, or configuration management tools such as Ansible) are not practical.

---
## 4. Terraform Import – When & How to Use It 

`terraform import` is your lifeline when:

- A resource **already exists** in the cloud  
- You want **Terraform to manage** it going forward  
- **Without recreating** it  
- **Zero downtime**

### Import Rules (do it in this exact order)

1. Write the full resource block in your config first  
2. Then run the import command:

```bash
terraform import storage_account.demo my-sa
```
After import – reality check

State now knows everything about the real resource 
Your config must still match reality (attributes, tags, etc.)
You’ll likely need to manually clean up/fix the config afterward

Import moves: Real world → State file
Import does NOT generate or fix your config!

---

## Lab Exercise

1. Create two workspaces:  
   ```bash
   terraform workspace new dev
   terraform workspace new prod
   ```  
2. Deploy an Virtual Machine using a **data source** to fetch the latest Ubuntu Image.  
3. Add a **local-exec** provisioner to log the VMs public IP into a text file.  
4. Switch between `dev` and `prod` workspaces to confirm each maintains its own instance and state.  
5. Clean up resources when done:  
   ```bash
   terraform destroy
   ```

---

## 5. Key Takeaways

- **Workspaces**: Isolate environments with unique state files.  
- **Data sources**: Read attributes of existing infrastructure.  
- **Provisioners**: Run setup scripts during resource creation or destruction.
- **Import**: Bring existing infrastructure under Terraform management (write config first, then import - it moves real world → state, but doesn't generate code). 

---

## 6. Practice Questions

### Question 1
What happens if you run `terraform apply` in a new workspace without selecting it first?  
A) It applies to the default workspace.  
B) It errors out.  
C) It creates resources in all workspaces.  
D) It creates a new workspace automatically

<details>  
<summary>Show Answer</summary>  
Answer: **A** - Terraform always operates in the current workspace. If you haven't selected a workspace, it uses the "default" workspace. You must use `terraform workspace select` to switch.
</details>

---

### Question 2
What is the main difference between a data source and a resource?
A) Data sources are read-only, resources are managed
B) Data sources cost money, resources are free
C) Data sources only work with Azure, resources work everywhere
D) There is no difference

<details>
<summary>Show Answer</summary>
Answer: **A** - Data sources are read-only queries that fetch information about existing infrastructure without managing it. Resources are created, updated, and destroyed by Terraform.
</details>

---

### Question 3
When should you use provisioners instead of user_data or cloud-init?
A) Always - provisioners are the recommended approach
B) When you need to run commands after resource creation that can't be done with user_data
C) Never - provisioners should never be used
D) Only for Windows instances

<details>
<summary>Show Answer</summary>
Answer: **B** - Provisioners should be a last resort. Use user_data, cloud-init, or configuration management tools (Ansible, Chef) first. Provisioners are useful for post-creation tasks that can't be handled by built-in initialization methods. While not deprecated, they are discouraged in favor of more reliable alternatives.
</details>
