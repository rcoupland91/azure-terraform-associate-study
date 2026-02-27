# Troubleshooting and Debugging Terraform

## Learning Objectives
- Identify common Terraform error categories and their solutions.
- Learn debugging techniques using TF_LOG and other tools.
- Understand how to resolve state-related issues.
- Master state manipulation commands for fixing issues.

---

## 1. Overview
Terraform errors usually fall into a few buckets:

- Backend/state problems (Storage Accounts, resource locks)
- Auth/provider problems (Azure creds, region, profile)
- Drift or “resource already exists” problems
- Bad config problems (syntax, wrong refs, cycles)
- Provisioner / apply-time problems

This section gives you a fast way to figure out which bucket you’re in, and what to run first.

## 2. Turn on Logging (First Thing)
When Terraform is being vague, turn on TF_LOG.

**Linux/macOS:**

```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
terraform apply
```

**Windows (PowerShell):**

```powershell
$env:TF_LOG="DEBUG"
$env:TF_LOG_PATH="terraform.log"
terraform apply
```

After that, check `terraform.log` in the current folder. Turn it off when done:

```bash
unset TF_LOG
```

## 3. Common Error: "Error loading state: AccessDenied" (Azure Storage Account backend)

**What it means:**
- Terraform tried to read/write state to the Storage Account and Azure blocked it.
- Usually wrong azure app reg attempting to access with wrong perms, wrong storage account

**Checklist:**

1. Check AzureRM tfstate name in backend:

   ```hcl
   backend "azurerm" {
    resource_group_name  = "testrg"
    storage_account_name = "testtfstatesa"
    container_name       = "test-tfstate"
    key = "testfile.tfstate"

   }
   ```

2. Confirm your App Reg has correct RBAC perms :
   - store
   - read
   - lock
   - update

If your pipeline fails with:  
Error loading state: AccessDenied: Access Denied  
Your next step is: enable TF_LOG=DEBUG and re-run — not “terraform login.”

## 4. Common Error: "Error acquiring the state lock"
Happens when:

- Another terraform apply is running
- A previous run crashed and left the lock
- You killed terraform mid-run

**What to do:**

1. If you know no one else is running Terraform:

   ```bash
   terraform force-unlock <LOCK_ID>
   ```

   LOCK_ID will be in the error message.

2. If using resource locks, you can also remove the lock item manually (Azure Storage account/Resource Group lock)

## 5. Common Error: "Resource already exists"
This shows up when:

- Someone created the resource in the console
- You imported the resource but didn’t move it in state
- Your name/tag is not unique

**Fix options:**

**Option A: Import it**

```bash
terraform import -var-file ".tfvars" -var-file ".tfvars" $TerraformResource $ResourceID
```

**Option B: Rename / change resource name in config**  
Use terraform state mv if you messed up the name:

```bash
terraform state mv module.testapp_vm module.testapp_vm[0]
```

Or
```bash
terraform state mv module.testapp_vm module.testapp_vm["run"]
```

## 6. Common Error: "Dependency cycle"
Terraform can’t figure out which resource to create first.

**Typical cause:**

- Output of A references B
- B references A through a data source or variable

**Fix:**

- Remove circular reference
- Sometimes use depends_on  
  **Example:**

  ```bash
  resource "azurerm_private_endpoint" "test-endpoint" {
  depends_on = [
    azurerm_private_endpoint.first-endpoint,
  ]
  }
  ```

## 7. Debugging Provisioners
Provisioners fail a lot more than people admit.

If you see:  
Error: remote-exec provisioner error  
it usually means:

- SSH couldn’t connect (wrong key, wrong user, instance not ready)
- Command failed (apt-get locked, yum unavailable)

**What to check:**

- Does the VM have a public IP?
- Is security group allowing SSH (22) from your runner(or agent) / your IP?
- Is the SSH user correct? (ubuntu vs ec2-user vs centos)
- Add a sleep:

  ```hcl
  provisioner "remote-exec" {
    inline = [
      "sleep 15",
      "sudo apt-get update -y"
    ]
  }
  ```

Or better: move config to user_data.

## 8. State Surgery (When Things Are Out of Sync)

**A. Remove something from state but don’t delete in Azure:**

```bash
terraform state rm azure_vm.old
```

Use when Terraform thinks it owns a resource but it shouldn’t.

**B. Rename something in state:**

```bash
terraform state mv azure_vm.web azure_vm.web01
```

Use when you copied a block and changed the name but Terraform is confused.

**C. Show what’s in state:**

```bash
terraform state list
terraform state show azure_vm.web
```

## 9. Drift Detection
If someone changes things in the console, terraform plan will show changes.  
To refresh state without planning:

```bash
terraform refresh
```

(Heads up: refresh is being phased/moved in newer versions — but the idea is “sync with remote.”)

## 10. Good Troubleshooting Flow

1. terraform init (does backend work?)
2. terraform validate (is the config valid?)
3. terraform plan (does the provider/auth/state work?)
4. export TF_LOG=DEBUG and re-run if still broken
5. Check Storage Account permissions
6. If state is stuck → terraform force-unlock
7. If resource name is wrong → terraform state mv
8. If console drift → terraform plan → apply

---

## 11. Practice Questions

### Question 1
What is the first step when encountering a vague Terraform error?  
A) Run terraform force-unlock.  
B) Enable TF_LOG=DEBUG.  
C) Run terraform refresh.  
D) Delete the state file

<details>  
<summary>Show Answer</summary>  
Answer: **B** - Enabling debug logging with `TF_LOG=DEBUG` provides detailed insights into the issue without altering state or resources. This helps identify the root cause before taking corrective action.
</details>

---

### Question 2
You see "Error acquiring the state lock". What is the most likely cause?
A) Another Terraform process is running
B) The S3 bucket doesn't exist
C) Credentials are invalid
D) The state file is corrupted

<details>
<summary>Show Answer</summary>
Answer: **A** - State lock errors occur when another Terraform operation is running (or crashed and left a stale lock). If no one else is running Terraform, you can use `terraform force-unlock <LOCK_ID>` to release it.
</details>

---

### Question 3
What command removes a resource from Terraform state without destroying it in the cloud?
A) `terraform destroy`
B) `terraform state rm`
C) `terraform state mv`
D) `terraform state delete`

<details>
<summary>Show Answer</summary>
Answer: **B** - `terraform state rm` removes a resource from state, but leaves the actual infrastructure intact in Azure. This is useful when moving resources between Terraform configurations or if a resource is now managed elsewhere.
</details>
