# Interview-Preparation
for interview questions and answers
======================================
'''
Refer ro respective branches for notes

Repository/organization variables

For normal configuration values, use Variables:

GitHub → Repository → Settings → Secrets and variables → Actions → Variables

For example, create:

API_URL = https://api.example.com
NODE_ENV = production

Then use it in your workflow:

jobs:
  deploy:
    runs-on: ubuntu-latest


    steps:
      - run: echo "API URL is ${{ vars.API_URL }}"
2. Secrets

For passwords, API keys, tokens, etc., use Secrets instead:

Settings → Secrets and variables → Actions → Secrets

For example:

DATABASE_PASSWORD = my-secret-password

Use it like:

steps:
  - run: ./deploy.sh
    env:
      DATABASE_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
'''
