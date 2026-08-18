# Interview-Preparation
for interview questions and answers
======================================
## GitHub Actions Variables

### 1. Repository / Organization Variables

For normal configuration values, use **GitHub Actions Variables**.

Go to:

**GitHub → Repository → Settings → Secrets and variables → Actions → Variables**

Add your variable, for example:

```text
API_URL = https://api.example.com
NODE_ENV = production

Use the variable in your GitHub Actions workflow:

jobs:
  deploy:
    runs-on: ubuntu-latest


    steps:
      - name: Show API URL
        run: echo "API URL is ${{ vars.API_URL }}"

Access variables using:

${{ vars.VARIABLE_NAME }}
```
### 2. Secrets

Use **Secrets** for sensitive or confidential values such as passwords, API keys, access tokens, database credentials, and private keys.

Go to:

**GitHub → Repository → Settings → Secrets and variables → Actions → Secrets**

Add or update your secret:

```text
DATABASE_PASSWORD = your-database-password
API_KEY = your-api-key
ACCESS_TOKEN = your-access-token

Use secrets in your workflow:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy application
        run: ./deploy.sh
        env:
          DATABASE_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
          API_KEY: ${{ secrets.API_KEY }}
          ACCESS_TOKEN: ${{ secrets.ACCESS_TOKEN }}


Use the following format to access a secret:

${{ secrets.SECRET_NAME }}
```
GitHub Actions OIDC Authentication with Azure, Terraform, and Azure Key Vault :: how resource is craeted through terraform by keeping secrets securely and the secrets will not be noted in terraform.statefile

This document explains how GitHub Actions authenticates with Microsoft Azure using OpenID Connect (OIDC) and how Terraform can then retrieve a secret from Azure Key Vault and use it when configuring an Azure Linux VM.

```
Authentication and Deployment Flow


Developer
   │
   │ git push
   ▼
GitHub Repository
   │
   │ triggers workflow . 
   ▼
GitHub Actions Runner ( "I am workflow X from repo Y")=
   │
   │ 1. github OIDC authentication ( GitHub OIDC Provider Gives GitHub a signed OIDC token(JWT) )
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
client secret -> GitHub gives Entra ID its OIDC token
   │
   ▼
Microsoft Entra ID ( Validates token
   │    and checks whether it matches
   │    the configured trust rules )
   │
   │ issues Azure access token
   ▼
Terraform
   │
   │ 2. requests secret using Azure identity
   ▼
Azure Key Vault
   │
   │ 3. returns secret value
   ▼
Terraform
   │
   │ 4. sends password to Azure VM API
   ▼
Azure Resource Manager
   │
   ▼
Azure Linux VM

```
  

