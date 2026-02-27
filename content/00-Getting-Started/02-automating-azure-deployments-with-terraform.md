# Automating Azure Deployments with Terraform

## Learning Objectives
- Understand how to integrate Terraform with CI/CD pipelines.
- Learn GitHub Actions workflow for Terraform automation.
- Master OIDC authentication for secure Azure access.
- Apply best practices for automated infrastructure deployments.

---

## 1. Storing Terraform State Securely (Storage Account Backend, resource Lock)

**Note:** This topic is covered in detail in Section 01 - State Management. This is a quick reference for CI/CD contexts.

For teams, store Terraform state remotely in Azure Storage Account for durability.

**Quick Reference:**
- Create Storage Account with versioning enabled
- Configure backend in `terraform` block (see Section 01 for details)
- Ensure permissions for Storage Account (Get/Put/List)

---

## 2. CI/CD Pipeline Setup with GitHub Actions

**Concept:**  
Automate Terraform workflows in CI/CD using GitHub Actions or Azure Devops Pipelines. Trigger on push/pull requests for `plan`, and on merge for `apply`. Use App Registrations for secure Azure access without long-lived credentials. (Federated credentials)

**Example: GitHub Actions Workflow (.github/workflows/terraform.yml)**

```yaml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC) 
        uses: azure/login@v2 
        with: 
          client-id: ${{ secrets.AZURE_CLIENT_ID }} 
          tenant-id: ${{ secrets.AZURE_TENANT_ID }} 
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.12.0

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        if: github.event_name == 'pull_request'
        run: terraform plan -out=plan.tfout

      - name: Terraform Apply
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
```

**Example: GitHub Actions Workflow (.github/workflows/terraform.yml)**

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  TF_VERSION: '1.12.0'

steps:
  - checkout: self

  # Authenticate to Azure using the Service Connection
  - task: AzureCLI@2
    displayName: "Azure Login"
    inputs:
      azureSubscription: "My-Azure-Service-Connection"   # Name of your service connection
      scriptType: bash
      scriptLocation: inlineScript: |
        echo "Logged into Azure"

  - task: TerraformInstaller@1
    displayName: "Install Terraform"
    inputs:
      terraformVersion: $(TF_VERSION)

  - script: terraform init
    displayName: "Terraform Init"

  - script: terraform validate
    displayName: "Terraform Validate"

  - script: terraform plan -out plan.tfout
    displayName: "Terraform Plan"
    condition: eq(variables['Build.Reason'], 'PullRequest')

  - script: terraform apply -auto-approve
    displayName: "Terraform Apply"
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
```


**Key Points:**  
- Use OIDC trust policy in Azure App Reg for GitHub.
- Plan on PRs for review; apply on main merges.  
- Store outputs/secrets securely in GitHub repo settings.

**Best Practice:**  
- Require PR approvals before merge.  
- Use workspaces for env separation (dev/prod).  
- Add notifications (e.g., Slack/MS Teams) on failures.

---

## 3. Lab Exercise

1. Create Storage Account in Azure.  
2. Configure Terraform backend in your .tf file.  
3. Run `terraform init` to migrate state to Azure Storage Account.  
4. Set up GitHub repo with the workflow YAML.  
5. Push changes and observe plan/apply in Actions tab.  
6. Clean up: `terraform destroy`.

---

## 4. Key Takeaways

- **Secure State:** Storage Account for storage, Resource group locksprevents corruption.  
- **Automation:** GitHub Actions for safe, automated deployments.  
- **Security:** Use OIDC over access keys.

---

## 5. Practice Questions

### Question 1
In a GitHub Actions workflow, what is the recommended method for authenticating to Azure?
A) Hardcode Azure credentials in the workflow file
B) Store credentials as GitHub Secrets
C) Use OIDC to assume an Azure role/App Reg
D) Use the Azure CLI default profile

<details>
<summary>Show Answer</summary>
Answer: **C** - OIDC (OpenID Connect) allows GitHub Actions to assume Azure roles without storing long-lived credentials. This is more secure than storing access keys as secrets.
</details>

---

### Question 2
What should a Terraform CI/CD pipeline do on pull requests?
A) Run `terraform apply` automatically
B) Run `terraform plan` to show what would change
C) Run `terraform destroy` to clean up
D) Skip Terraform validation

<details>
<summary>Show Answer</summary>
Answer: **B** - On pull requests, run `terraform plan` to preview changes without applying them. Apply should only happen on merge to main/master after review.
</details>
