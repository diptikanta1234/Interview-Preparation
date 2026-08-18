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
