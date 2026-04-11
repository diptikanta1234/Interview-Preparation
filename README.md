# Interview-Preparation
for interview questions and answers

# Interview-Preparation
for interview questions and answers

# Terraform Notes n interview questions:-
========================================
terraform init - It initializes the process by downloading provider plugins for cloud provider (Aws)/azure/gcp. initializes the backend (local or remote like in S3), and sets up the .terraform/ directory.
terraform validate	-> Check if any syntax error in configuration.
terraform plan	-> Its the dry-run. Preview infrastructure changes/
terraform apply ->	Deploy resources
terraform show	-> Display current state
terraform destroy ->	deletes infrastructure

Q.lets say i want to create ec2 in qa account. how do you manage the credential to create ec2 in that account while i apply terraform apply.
the account/IAMuser for which we are creating a resource, go to credentials -> generate the secret key -> use - CLI -> download the key
There are 3 ways we should prefer doing this -
WAY 1: - in the host/bastion from where you are doing terraform apply, create a '~/.aws/credentials' then set the aws_access_key_id , aws_secret_access_key with profile name. same profile you can use in configuration files to create resource
[terraform-user]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region                = ap-south-1

provider.tf — points directly at terraform-user's profile, no assume_role needed:
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

WAY 2:- will help during setting of CI/CD infra pipeline -

Store credentials as encrypted secrets in your CI platform, then inject them as env vars at pipeline runtime. Click your platform: ( GitHub Actions / Jenkins ). we can store these in AWS KMS or vault then use it in environmental variable in pipeline.

GitHub → Repo → Settings → Secrets and variables → Actions → New repository secret
Secret name - AWS_ACCESS_KEY_ID 
Secret name - AWS_SECRET_ACCESS_KEY
Secret name - AWS_DEFAULT_REGION
<img width="928" height="950" alt="image" src="https://github.com/user-attachments/assets/d64ad8de-4113-47f9-9a37-15f60292b1c1" />

Then reference them in your workflow file or jenkinsfile :
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

WAY 3:- through IAM role ( assume_role )
The assume_role pattern is only needed when Terraform runs in account A but needs to create resources in account B (true multi-account). mainly cross-account.

This is the proper production pattern. Your principal (developer machine, CI runner, or EC2 instance) has one identity, and Terraform temporarily assumes a role in the Prod account. here the ec2 in CI runner and production infra a/c mostlyare different.
Step 1 — create a role in the prod account with a trust policy allowing your source account to assume it:
json{
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
Step 2 — attach the role in provider.tf:
hclprovider "aws" {
  region = "ap-south-1"

  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformRole"
    session_name = "TerraformPRODSession"
    external_id  = "unique-external-id"
  }
}
When you run terraform apply, the AWS SDK calls sts:AssumeRole, gets a temporary set of credentials (valid 1 hour by default), and uses those to provision resources in the prod account. Your source credentials never touch the PROD account directly.

Q. lets say i have created 3 ec2 instance then one s3 bucket. For 2 ec2 intsance all parameter is fine to create, 1 ec2 not having valid ami-id so that it could not created. after that s3 bucket can be created as per configuration. 
how terraform will behave ? will it create all other resources except 3rd ec2 ?

Terraform does NOT stop when EC2-3 fails.
It continues creating other independent resources.
At the end it shows a partial success summary. the o/p looks as below-
<img width="934" height="594" alt="image" src="https://github.com/user-attachments/assets/6d00a1d5-3101-4089-a353-147b913ea4fe" />

Q. Have you used modules in one project by importing from other projects. in how many ways the module can be stored remotely and can be imported and used in other project in ur organisation.

in our organisation we are storing the source code and modules in github-

Supose the ec2 module and vpc module are stored in github in below path. now we are using those in project-a.

# project-a/main.tf
module "my_ec2" {
  source        = "github.com/your-org/terraform-modules//ec2?ref=v1.0.0"
  instance_type = "t2.micro"
  ami_id        = "ami-0abcdef1234567890"
}

module "my_vpc" {
  source       = "github.com/your-org/terraform-modules//vpc?ref=v1.0.0"
  cidr_block   = "10.0.0.0/16"
}

Q. Have you used modules in one project by importing from other projects. in how many ways the module can be stored remotely and can be imported and used in other project in ur organisation ?
through github ( using in our currect project) oe gitlab or bitbucket
through S3
Terraform Private Registry (Terraform Cloud / Enterprise)
terraform public registry

<img width="751" height="439" alt="image" src="https://github.com/user-attachments/assets/a2576188-68ee-4d10-a941-7aa8ac5c6be7" />
lets say i created a ec2 instance xx -> stored in github in terraform-vss-module repo in main branch. now use the same ec2 module in project-a...explain in detail with code -

Create Module Repo terraform-vss-module
ec2/main.tf
hclresource "aws_instance" "xx" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name        = var.instance_name
    Environment = var.environment
  }
}

ec2/variables.tf
hclvariable "ami_id" {
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

ec2/outputs.tf
hcloutput "instance_id" {
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

Push code to GitHub:
bash# In your local terraform-vss-module folder
git init
git add .
git commit -m "added ec2 module"
git branch -M main
git remote add origin git@github.com:your-org/terraform-vss-module.git
git push -u origin main
Your GitHub repo looks like this:
terraform-vss-module/
└── ec2/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf

STEP 2: Create project-a and consume the module
project-a/main.tf
hclprovider "aws" {
  region = "eu-north-1"
}

# Calling the EC2 module from terraform-vss-module repo
module "my_ec2_instance" {
  source = "github.com/your-org/terraform-vss-module//ec2?ref=main"

  # Passing values to module variables
  ami_id        = var.ami_id
  instance_type = var.instance_type
  instance_name = var.instance_name
  environment   = var.environment
}

# Use the outputs from module
output "ec2_instance_id" {
  value = module.my_ec2_instance.instance_id
}

output "ec2_public_ip" {
  value = module.my_ec2_instance.public_ip
}


project-a/variables.tf
hclvariable "ami_id" {
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

project-a/terraform.tfvars
hclami_id        = "ami-0abcdef1234567890"
instance_type = "t2.micro"
instance_name = "vss-ec2-instance"
environment   = "dev"

STEP 3: Run in project-a
bash# initialise - this DOWNLOADS the module from github

bashterraform plan
terraform apply

What // and ?ref= mean:
github.com/your-org/terraform-vss-module//ec2?ref=main
│                                        │    │       │
│                                        │    │       └── branch name (main)
│                                        │    └────────── subfolder inside repo (ec2/)
│                                        └─────────────── double slash separates repo from subfolder
└──────────────────────────────────────────────────────── github repo path

Using a Tag instead of branch (recommended for production):
bash# In terraform-vss-module repo, create a tag
git tag v1.0.0
git push origin v1.0.0
hcl# In project-a, use tag instead of main branch
module "my_ec2_instance" {
  source = "github.com/your-org/terraform-vss-module//ec2?ref=v1.0.0"
  ...
}

Q. How can we delete a specific resource named as vss-instance in terraform?

terraform destroy -target="aws_instance.vss-instance"

Q. i have one ec2 instance created in aws console but not through terraform. how can we  get the ec2 into IAC.
