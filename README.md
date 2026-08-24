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
```
OIDC ( OpenId Connect) -> 
•	mainly used for authethentication ( WHO are u).   
•	Provides an ID Token (commonly a JWT)
•	Contains identity/claims about the authenticated user or workload

OAuth :-
•	mainly used for Authorization ("What are you allowed to access?").
•	Provides an Access Token
•	Used by a client/application to access an API/resource
•	Contains information/permissions (scopes, roles, etc.)
```
  
<img width="2928" height="2884" alt="image" src="https://github.com/user-attachments/assets/2d20e010-96c6-463a-a331-184e22f08008" />


## Build and deploy flask application. now deploy it using ci(build code, test, build image, tag the version, push to ACR) and cd ( deploy to aks ) pipeline using github action by creating a self hosted agent. in ci write multistage docker file to build. now write the aks/k8s yaml and install those through helm...once done install prometheus and grafana to check the health of cluster/node/pod.

app.py
```
import os
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({"status": "healthy", "message": "Flask App running on AKS!"})

@app.route('/health')
def health():
    return jsonify({"status": "UP"}), 200

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```
test_app.py

```
import pytest
from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_home(client):
    rv = client.get('/')
    assert rv.status_code == 200
    assert b"Flask App running on AKS!" in rv.data

def test_health(client):
    rv = client.get('/health')
    assert rv.status_code == 200
    assert b"UP" in rv.data
```

```
# Stage 1: Build stage
FROM python:3.11-slim AS builder

WORKDIR /app

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime stage
FROM python:3.11-slim AS runner

WORKDIR /app

COPY --from=builder /opt/venv /opt/venv
COPY app.py .

ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PORT=5000

# Non-root user for security
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

CMD ["python", "app.py"]
```
What is requirements.txt? why we use this?

requirements.txt - It contains different library, versions, dependencies that is required to run the application.
We use 'pip install -r requirment.txt' to install these list of dependencies with same version defined in this file.

-> Who gives or creates this file?
the developer create and maintain this file.When building a Python project locally, you install libraries using pip. Once your application works, you generate this file so other developers, CI/CD pipelines, and Docker containers know exactly what to install.
You typically generate it automatically from your local virtual environment by running:
```
pip freeze > requirements.txt
```
inside this file we have -
```
flask == 3.0.2
pytest == 8.0.0
gunicorn==21.2.0
requests>=2.31.0
psycopg2-binary==2.9.9
```
Helm:-
It is a kubernetes package manager like ( apt, yum, apt etc). Instead of using 'kubectl apply -f '
, we can use helm commands to install and apply.

supose we want to manage some different attributes based on environments ( like 2 replicas for dev, 3 for preprod, 5 for prod), we can define different values.yaml ( eg- value_dev.yaml etc) to achieve it.

Chart Directory Structure
```
charts/flask-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```
3. Helm Chart Structure
Create a directory charts/flask-app with the following files:

charts/flask-app/Chart.yaml
```
apiVersion: v2
name: flask-app
description: A Helm chart for deploying Flask application on AKS
type: application
version: 0.1.0
appVersion: "1.0.0"
```
charts/flask-app/values.yaml
```
replicaCount: 2

image:
  repository: myacr.azurecr.io/flask-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 5000

ingress:
  enabled: true
  host: flask.yourdomain.com
  clusterIssuer: letsencrypt-prod
  tlsSecretName: flask-app-tls-secret

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```
charts/flask-app/templates/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 10
```
charts/flask-app/templates/service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    app: {{ .Chart.Name }}
```
templates/ingress.yaml
```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
  annotations:
    cert-manager.io/cluster-issuer: {{ .Values.ingress.clusterIssuer | quote }}
    kubernetes.io/ingress.class: nginx
spec:
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.tlsSecretName }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}-{{ .Chart.Name }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

4. GitHub Actions CI/CD Pipeline (.github/workflows/deploy.yml)
Set up secrets in GitHub repository: AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID, ACR_NAME, RESOURCE_GROUP, AKS_CLUSTER_NAME.
```
name: CI/CD Pipeline to AKS

on:
  push:
    branches: [ "main" ]

permissions:
  id-token: write
  contents: read

jobs:
  ci-cd:
    runs-on: self-hosted # Running on your Self-Hosted Runner
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies & Run Tests
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt pytest
          pytest test_app.py

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Build, Tag, and Push Image to ACR
        run: |
          ACR_NAME=${{ secrets.ACR_NAME }}
          IMAGE_TAG=${{ github.sha }}

          az acr login --name $ACR_NAME

          docker build -t $ACR_NAME.azurecr.io/flask-app:$IMAGE_TAG -t $ACR_NAME.azurecr.io/flask-app:latest .
          docker push $ACR_NAME.azurecr.io/flask-app:$IMAGE_TAG
          docker push $ACR_NAME.azurecr.io/flask-app:latest

      - name: Set AKS Context
        uses: azure/aks-set-context@v3
        with:
          resource-group: ${{ secrets.RESOURCE_GROUP }}
          cluster-name: ${{ secrets.AKS_CLUSTER_NAME }}

      - name: Deploy via Helm
        run: |
          ACR_NAME=${{ secrets.ACR_NAME }}
          IMAGE_TAG=${{ github.sha }}

          helm upgrade --install flask-release ./charts/flask-app \
            --namespace default \
            --set image.repository=$ACR_NAME.azurecr.io/flask-app \
            --set image.tag=$IMAGE_TAG
```

Prometheus & Grafana Setup on AKSTo monitor the health of your nodes, clusters, and pods, install the kube-prometheus-stack via Helm from your self-hosted runner or local terminal logged into AKS:Bash
```
# 1. Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 2. Create namespace for monitoring
kubectl create namespace monitoring

# 3. Install kube-prometheus-stack (includes Prometheus, Grafana, Node Exporter, Kube State Metrics)
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring

# 4. Get Grafana 'admin' password
kubectl get secret --namespace monitoring prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo

# 5. Access Grafana Dashboard locally via Port-Forwarding
kubectl port-forward service/prometheus-stack-grafana 8080:80 -n monitoring
```
Open http://localhost:8080 in your browser:Username: adminPassword: (Retrieved from step 4)Out of the box, kube-prometheus-stack auto-configures pre-built Grafana dashboards for:Kubernetes / Compute Resources / ClusterKubernetes / Compute Resources / Node (Pods)Kubernetes / Compute Resources / Workload
