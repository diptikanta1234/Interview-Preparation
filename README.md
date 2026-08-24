# Terraform Notes & Interview Questions

---

## Core Commands

| Command | Description |
|---|---|
| `terraform init` | Downloads provider plugins (AWS/Azure/GCP), initializes backend (local or remote like S3), sets up `.terraform/` directory |
| `terraform validate` | Checks for syntax errors in configuration |
| `terraform plan` | Dry-run — previews infrastructure changes |
| `terraform apply` | Deploys resources |
| `terraform show` | Displays current state |
| `terraform destroy` | Deletes infrastructure |

---
<img width="725" height="375" alt="image" src="https://github.com/user-attachments/assets/834f4005-5b9c-49e4-9496-a414c9f46a7d" />

## Q. how to set terraform remote in your project? Why do u store it in remote?
We are using azure centralized azure storage account. our terraform.ststefile gets stored inside blob container inside the storage account
```
Benefits
Centralized state — everyone uses the same terraform.state file across our project.
State locking — prevents two Terraform operations from modifying state simultaneously.
Team collaboration — developers and CI/CD can work with the same infrastructure.
Security — access can be controlled using Azure RBAC/Entra ID.
Durability/recovery — Azure Storage can provide versioning/soft-delete protections.
CI/CD friendly — pipelines don't need to maintain a local state file.

there will be no state files for child modules...its the frame that is stored in gihub in our org.
To design the landing zone for different requirement, we start calling the child resources -> once applied the states are stored in terraform.state file in different storage account
Landingzone_app_vss -> statefile in azure storage account storagevss2026
Landingzone_app_IM -> statefile in azure storage account storageIM2026 
```
in backend.tf -
```
#C:\Users\admin\Desktop\azureClass\azureTerraformPrograms\practice19thAug\environment\dev\terraform.tfstate.backup

terraform {
  backend "azurerm"{
    resource_group_name="example-resources"
    storage_account_name="remotestorage9999"
    container_name = "storagetfstatecont"
    key = "dev/terraform.tfstate"
  }
}
```
then do terraform init -> it will move the contents inside local.statefile to remote dev/terraform.tfstate
State locking features available automatically in remote statefile.

## Q. Lets say two developer are doing terraform plan at a time, your state file is in remote azure storage account, what will happen?
terraform starts acquiring lock for one session for developer-A, and do terraform plan by locking that session. it hold the lock until completion.

During that time it will give error to developer-B "Error: Error acquiring the state lock"( Error message: state blob is already locked )
State lock happens for both terraform plan and apply.

once  developer-A completes then developer-B will try again terraform plan to have it successful.
developer-A session:
<img width="979" height="218" alt="image" src="https://github.com/user-attachments/assets/c0b4a2eb-d13e-47f3-9bbf-61ae17125587" />
developer-B session:
<img width="1550" height="341" alt="image" src="https://github.com/user-attachments/assets/05bd06a8-e0ad-42fe-932a-10b2588efab0" />

## Q. How can you unlock a state lock ?
terraform force-unlock <lockId>
terraform force-unlock b04622b8-c4a1-6a39-341c-b5319fc104b7
<img width="1121" height="587" alt="image" src="https://github.com/user-attachments/assets/f632274e-71ce-4b4e-a316-7e6ea4ccc811" />

## Q. How will recover Terraform state file if its crashed/deleted ? 
First of all don't run terraform apply command when state file is deleted. it may create duplicate resources.

If your Terraform state file (terraform.tfstate) in an Azure Blob Storage backend is deleted, recovery depends primarily on what data-protection features were enabled on the Storage Account.

<img width="763" height="443" alt="image" src="https://github.com/user-attachments/assets/08749265-7d95-4d65-8450-34dc084ee6fc" />

Recovery options
Option A — Blob soft delete is enabled

This is the easiest recovery path. Azure Blob soft delete keeps deleted blobs for a configured retention period, up to 365 days.
<img width="1869" height="544" alt="image" src="https://github.com/user-attachments/assets/1c96dd9f-ec8f-40da-a582-fe24a27c3220" />

In the Azure Portal: Open the Storage Account -> Containers -> Open the container containing your Terraform state -> Enable Show deleted blobs -> Find the deleted terraform.tfstate -> Select it -> Choose Undelete.

After restoring it, verify that Terraform can see the state:
terraform init
terraform state list . You should see your existing resources.

Option B — Blob versioning is enabled

enable Blob Versioning, Azure maintains previous versions of the blob. When the current blob is deleted, the previous version can remain available.

Storage Account → Container → terraform.tfstate → Versions

Find the most recent good version and make it the current version.

Be careful here: with versioning enabled, simply undeleting the blob does not necessarily recreate the current blob. Microsoft recommends promoting/copying the desired previous version to become the current version.
<img width="927" height="314" alt="image" src="https://github.com/user-attachments/assets/f27d3d12-2029-462a-9515-6a6790a28df8" />
Option C - Azure Backup or other storage backup. we can take the backup of storage with GRS or ZRS

As our version is enabled, after the statefile is deleted we will check the different versions available and will check the appropriate versionId to 'Make current version'.
<img width="1879" height="912" alt="image" src="https://github.com/user-attachments/assets/4a66d840-f3b6-4ee7-b245-1c9bb5cd0051" />
### Q. what is "lifecycle" block in terraform?
Lifecycle block is a meta-argument in terraform which is used to control the creation, modification and deletion of resources.
Types -
```
-> create_before_destroy - creates a resource with same configuration before destroying it. Normally used to reduce downtime ( for any configuration changes to be reflected)
-> prevent_destroy - Used for critical resources to prevent deletion.( e.g -prod DB, vault, storage, vm etc)
-> ignore_changes - tells terraform to ignore the change of specific attributes (e.g- tags)
-> replace_triggered_by - replace the resource based on dependent resource/attribute changes.

1. create_before_destroy

It tells Terraform to create the replacement resource before destroying the existing resource.

resource "azurerm_network_security_group" "example" {
  name                = "example-nsg"
  location            = "East US"
  resource_group_name = "example-rg"

  tags = {
  Environment = "prod"
}


  lifecycle {
    create_before_destroy = true
    # prevent_destroy = true
  }

/*
#ignore_changes lifecycle
lifecycle {
    ignore_changes = [
      tags
    ]
*/

}

Azure interview example:

Suppose an Azure resource requires replacement because of a configuration change. Normally Terraform may destroy the old resource first and then create the new one.

4. replace_triggered_by -

resource "azurerm_linux_virtual_machine" "app" {
  name                = "app-vm"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location

  lifecycle {
    replace_triggered_by = [
      azurerm_network_interface.app
    ]
  }
}
```
### Q. What are meta-arguments in terraform?

Meta-arguments are special arguments that terraform controls to manage resources.
```
for example -
location, name -> are the arguments - defined by azure resources.
count, for_each, depends_on, lifecycle, provider -> meta-arguments -controls how terraform manages resources

count	-> Create multiple resource instances using indexes.[0], [1]
         Good for identical resources

         resource "azurerm_resource_group" "example" {
           count    = 5
           name     = "my-rg-${count.index}"
           location = "East US"
       }

for_each	-> Create multiple instances using keys.["dev"], ["prod"]
          Good for resources with different values
```
### Q. terraform --replace command vs create_before_destroy
```
terraform apply -replace = replace a resource for this particular Terraform run.
terraform apply -replace="azurerm_linux_virtual_machine.example"

create_before_destroy = change the replacement strategy for the resource whenever replacement is required
This is configured in the Terraform code.

```
### Q.  if-else condition? 
 if-els is usually implemented using the conditional expression:
```
condition ? value_if_true : value_if_false

e.g - create a vm if env is prod otherwise don't create.
count = var.environment == "prod" ? 1 : 0

vm_size = var.environment == "prod" ? "Standard_D2s_v5" : "Standard_B1s"

```
### Q. CI/CD Pipeline Secrets (GitHub Actions / Jenkins) how do you store?

Store credentials as encrypted secrets in your CI platform, then inject them as environment variables at pipeline runtime. You can store these in AWS KMS or Vault, then use them as environment variables in the pipeline.

**GitHub → Repo → Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | your access key |
| `AWS_SECRET_ACCESS_KEY` | your secret key |
| `AWS_DEFAULT_REGION` | e.g. `ap-south-1` |

![GitHub Secrets Setup](https://github.com/user-attachments/assets/d64ad8de-4113-47f9-9a37-15f60292b1c1)

Then reference them in your workflow file or Jenkinsfile:

```yaml
# .github/workflows/terraform.yml
name: Terraform Apply

on:
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    env:
      AWS_ACCESS_KEY_ID:     ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_DEFAULT_REGION:    ${{ secrets.AWS_DEFAULT_REGION }}

    steps:
      - uses: actions/checkout@v4
```
### Understanding `//` and `?ref=`

```
github.com/your-org/terraform-vss-module//ec2?ref=main
│                                        │    │    │
│                                        │    │    └── branch name (main)
│                                        │    └─────── subfolder inside repo (ec2/)
│                                        └──────────── double slash separates repo from subfolder
└───────────────────────────────────────────────────── GitHub repo path
```

> **Recommended for production** — use a tag instead of branch:
>
> ```bash
> git tag v1.0.0
> git push origin v1.0.0
> ```
## Q. In how many ways can a Terraform module be stored remotely and imported into other projects?

```
module "network" {
  source = "git::https://github.com/company/terraform-modules.git//network?ref=v1.0.0"
}
```
| Source | Example |
|---|---|
| GitHub / GitLab / Bitbucket | `github.com/your-org/terraform-modules//ec2?ref=v1.0.0` |
| Terraform Private Registry | Terraform Cloud / Enterprise |

![Module sources diagram](https://github.com/user-attachments/assets/a2576188-68ee-4d10-a941-7aa8ac5c6be7)
<img width="1191" height="167" alt="image" src="https://github.com/user-attachments/assets/013eb675-9fa7-4e11-99a6-d76fdab2c1c8" />



---

## Q. What is a dynamic block in Terraform?
## Q. lets say i am creating 3 ec2 instance with same type having count=3 using below code. Now i want to delete a specific instance. How can we do it in Terraform?
## Q. I have an EC2 created in AWS Console (not via Terraform). How do I bring it into IaC?



**######################  AWS  ########################**
## Q. I created 3 EC2 instances and 1 S3 bucket. EC2-3 has an invalid AMI ID and fails. Will the other resources still be created?

**Yes.** Terraform does NOT stop when EC2-3 fails. It continues creating other independent resources and shows a partial success summary at the end.

![Partial success output](https://github.com/user-attachments/assets/6d00a1d5-3101-4089-a353-147b913ea4fe)


## Q. How do you manage credentials when creating an EC2 in a QA account using `terraform apply`?

Go to the account/IAM user → Credentials → Generate secret key → Use CLI → Download the key.

There are **3 preferred ways** to handle this:

---

### Way 1 — AWS Credentials File (local/bastion host)

On the host where you run `terraform apply`, create `~/.aws/credentials` and set the access key, secret key, and profile name. Use the same profile in your config files.

```ini
[terraform-user]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region                = ap-south-1
```

**`provider.tf`** — points directly at `terraform-user`'s profile, no `assume_role` needed:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = "ap-south-1"
  profile = "terraform-user"
}
```

---



---

### Way 3 — IAM Role (`assume_role`) — Cross-Account (Recommended for Production)

The `assume_role` pattern is used when Terraform runs in **account A** but needs to create resources in **account B** (true multi-account setup). Your principal (developer machine, CI runner, or EC2 instance) has one identity, and Terraform temporarily assumes a role in the prod account.

**Step 1** — Create a role in the prod account with a trust policy allowing your source account to assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::SOURCE_ACCOUNT_ID:root" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "unique-external-id" }
    }
  }]
}
```

**Step 2** — Attach the role in `provider.tf`:

```hcl
provider "aws" {
  region = "ap-south-1"

  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformRole"
    session_name = "TerraformPRODSession"
    external_id  = "unique-external-id"
  }
}
```

When you run `terraform apply`, the AWS SDK calls `sts:AssumeRole`, gets temporary credentials (valid 1 hour by default), and uses those to provision resources in the prod account. Your source credentials never touch the PROD account directly.

---


---


**Using modules from GitHub in `project-a/main.tf`:**

```hcl
module "my_ec2" {
  source        = "github.com/your-org/terraform-modules//ec2?ref=v1.0.0"
  instance_type = "t2.micro"
  ami_id        = "ami-0abcdef1234567890"
}

module "my_vpc" {
  source     = "github.com/your-org/terraform-modules//vpc?ref=v1.0.0"
  cidr_block = "10.0.0.0/16"
}
```

---

## Q. Explain using a GitHub-hosted EC2 module in detail with code

### Step 1 — Create module repo `terraform-vss-module`

**`ec2/main.tf`**

```hcl
resource "aws_instance" "xx" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name        = var.instance_name
    Environment = var.environment
  }
}
```

**`ec2/variables.tf`**

```hcl
variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_name" {
  description = "Name tag for EC2"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

**`ec2/outputs.tf`**

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.xx.id
}

output "public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.xx.public_ip
}

output "private_ip" {
  description = "Private IP of the EC2 instance"
  value       = aws_instance.xx.private_ip
}
```

**Push code to GitHub:**

```bash
git init
git add .
git commit -m "added ec2 module"
git branch -M main
git remote add origin git@github.com:your-org/terraform-vss-module.git
git push -u origin main
```

Your GitHub repo structure:

```
terraform-vss-module/
└── ec2/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

---

### Step 2 — Create `project-a` and consume the module

**`project-a/main.tf`**

```hcl
provider "aws" {
  region = "eu-north-1"
}

module "my_ec2_instance" {
  source = "github.com/your-org/terraform-vss-module//ec2?ref=main"

  ami_id        = var.ami_id
  instance_type = var.instance_type
  instance_name = var.instance_name
  environment   = var.environment
}

output "ec2_instance_id" {
  value = module.my_ec2_instance.instance_id
}

output "ec2_public_ip" {
  value = module.my_ec2_instance.public_ip
}
```

**`project-a/variables.tf`**

```hcl
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "instance_name" {
  type = string
}

variable "environment" {
  type    = string
  default = "dev"
}
```

**`project-a/terraform.tfvars`**

```hcl
ami_id        = "ami-0abcdef1234567890"
instance_type = "t2.micro"
instance_name = "vss-ec2-instance"
environment   = "dev"
```

---

### Step 3 — Run in `project-a`

```bash
terraform init    # downloads the module from GitHub
terraform plan
terraform apply
```

---

>
> ```hcl
> module "my_ec2_instance" {
>   source = "github.com/your-org/terraform-vss-module//ec2?ref=v1.0.0"
> }
> ```

---

## Q. How do you delete a specific resource named `vss-instance` in Terraform?

```bash
terraform destroy -target="aws_instance.vss-instance"
```

---
## Q. lets say i am creating 3 ec2 instance with same type having count=3 using below code. Now i want to delete a specific instance. How can we do it in Terraform?
resource "aws_instance" "demo" {
    ami="ami-077d1b9f9a1902bbc"
    instance_type = "t3.micro"
    count = 3

    tags = {
        Name = "dev-vss-server-${count.index + 1}"
    }
}

Ans: we can delete the ec2 instance using its name i.e aws_instance.demo[0] or aws_instance.demo[1] or aws_instance.demo[0]. but not dev-vss-server-1 ( this is the tag).

how can we get which instanceId belong to demo[0]...lets break into

```bash
terraform state list
terraform state show demo[0] //check the instance-id that you want to delete
terraform destroy -target="aws_instance.demo[0]"
```
<img width="1013" height="551" alt="image" src="https://github.com/user-attachments/assets/a6a3bb50-76f4-4175-ab70-68640ec62b4f" />

terraform state list | grep aws_instance --> can see how many ec2 istance is there
---

Q.lets say somebody in my project did terraform destroy -target=aws_instance.demo[0] as its not needed as per requirement. but in statefile its still having...if someby adding some security group to a instance once we do terraform apply it will create again 2 resource...whats the ideal way to work in this scenario?

ans: This is a classic Terraform state drift + team workflow problem
Terraform works like this: State file = source of truth
Better to avild -target manually, if it identifies the changes it will sync as per .tf file or if someone modify other thing and does terraform.apply it will be creting another instnce.

better approach is -
<img width="1001" height="591" alt="image" src="https://github.com/user-attachments/assets/25140df3-e77d-4616-8cd9-83cd17bdd8b8" />


## Q. I have an EC2 created in AWS Console (not via Terraform). How do I bring it into IaC?

**Steps:**

1. Get Instance ID from AWS Console (e.g., `i-0abc123...`)
2. Write a resource block in your `.tf` file
3. Run `terraform import`
4. Run `terraform plan` → fix any drift
5. Run `terraform apply` → confirm no changes
6. Commit `.tf` files to version control

**Write the resource block:**

```hcl
resource "aws_instance" "vss_ec2" {
  ami           = "ami-xxxxxxxx"
  instance_type = "t2.micro"
}
```

**Run the import:**

```bash
terraform import aws_instance.vss_ec2 i-0a1b2c3d4e5f67890
```

---

## Q. I change an EC2 instance type from `t3.micro` to `t3.large`. Will it replace or update the instance?

It will **update in-place** — the same EC2 instance is modified without recreation.

```
terraform plan output:
  ~ update in-place   (stop → resize → start, ~1-2 min downtime)
```

![Plan output](https://github.com/user-attachments/assets/98b150e3-7cc2-44fd-95b7-15149142bd81)

| Symbol | Meaning | Risk |
|---|---|---|
| `-/+` | Destroy and recreate | ❌ Data loss risk |
| `~` | Update in-place | ✅ Safe, minor downtime |
| `+` | Create new | ✅ Safe |
| `-` | Destroy | ❌ Destructive |

---


> *(Answer to be added)*
