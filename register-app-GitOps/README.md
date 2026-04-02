
# Automating a Release Process through CI/CD

---

## Architecture Overview

![CI/CD Flow](https://github.com/user-attachments/assets/b3d21086-f5e8-4550-bb87-22fc36a0899f)

Here the CI job ends and CD job gets triggered automatically.

Our CD job will first update the build number and release number in `deployment.yaml` in the GitOps repo on GitHub, then ArgoCD will pull the manifest file and deploy the resources on the EKS cluster.

Once the CD job is finished it will send the notification over Slack and send the email as well.

![CD Flow](https://github.com/user-attachments/assets/16d97e33-b98f-4a56-bab0-5e7a4f75eae4)

---

## Step 1 - Create a Jenkins Master

- Selected an EC2 instance with **Ubuntu OS** and **t2.small**
- Access it through **MobaXterm**
- Install Jenkins by following the official documentation steps

---

## Step 2 - Create a Jenkins Agent

- Named as `jenkins-agent` with **Ubuntu OS**
- Instance type → **t3.small**
- Storage → **15GB gp3 volume**
- Change the hostname to `jenkins-agent`
- Reboot the system:
```bash
sudo init 6
```

- Install **Java JDK 17**:
```bash
sudo apt install openjdk-17-jdk -y
```

- Install **Docker**:
```bash
sudo apt install docker.io -y
```

- Give full rights to the current user:
```bash
sudo usermod -aG docker $USER
```

---

## Step 3 - Configure SSH Between Master and Agent

- Open `/etc/ssh/sshd_config` on **jenkins-agent**
- Uncomment the following lines:
```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

![sshd config](https://github.com/user-attachments/assets/6856b696-ece6-4859-917c-08538317ad7c)

- Do the **same on jenkins-master** → then reload SSH on both:
```bash
sudo service sshd reload
```

- On **jenkins-master**, generate SSH key:
```bash
ssh-keygen
```

- Go to `.ssh/` → copy contents of **public key**
- On **jenkins-agent**, open `.ssh/authorized_keys` → paste the public key on a new line

![authorized keys](https://github.com/user-attachments/assets/d2104362-5f6d-4778-8b3b-346f61091ed3)

---

## Step 4 - Configure Jenkins Master UI

- Open Jenkins in browser:
```
http://<jenkins-master-public-ip>:8080
```

- Use the initial admin password from:
```bash
/var/lib/jenkins/secrets/initialAdminPassword
```

- Go to **Manage Jenkins** → **System Configuration** → **Nodes** → **built-in node** → **Configure**
- Set **Number of Executors = 0** → Save

> **Note:** Setting executors to 0 on master means no jobs run on master directly.
> All jobs get routed to agent nodes — keeps master stable and secure.

---

## Step 5 - Add Jenkins Agent Node

Go to **Manage Jenkins** → **Nodes** → **New Node** and configure:

| Field | Value |
|---|---|
| Name | `jenkins-agent` |
| Executors | `2` |
| Remote home directory | `/home/ubuntu` |
| Labels | `jenkins-agent` |
| Usage | Use this node as much as possible |
| Launch method | Launch agent via SSH |
| Host | Private IP (same VPC) or Public IP |

**Add Credentials:**

| Field | Value |
|---|---|
| Kind | SSH username with private key |
| Description | `jenkins-agent` |
| Username | `ubuntu` |
| Private Key | Paste private key of jenkins-master directly |

![Node Config 1](https://github.com/user-attachments/assets/e672086a-c52d-49d8-b38f-d50c02fe8cb2)

![Node Config 2](https://github.com/user-attachments/assets/30862907-d90e-448b-8970-e7452debb55b)

Agent added successfully ✅

---

## Step 6 - Test Agent Connectivity

- Go to Jenkins → Create a **Pipeline job** → name it `test`
- Use pipeline script → **Hello World**
- Save → click **Build Now**
- Check **Console Output** → verify build ran on `jenkins-agent`

> This confirms connectivity from agent to master is successful ✅

---

## Step 7 - Integrate GitHub with Jenkins

Go to **Manage Jenkins** → **Plugins** → Install:

- `Maven`
- `Pipeline Maven Integration`
- `Eclipse Temurin Installer`

---

## Step 8 - Configure Tools in Jenkins

Go to **Manage Jenkins** → **Tools**:

**Maven Installation:**

| Field | Value |
|---|---|
| Name | `maven3` *(use same name in pipeline script)* |
| Version | `3+` |
| Install automatically | ✅ |

**JDK Installation:**

| Field | Value |
|---|---|
| Name | `java17` *(use same name in pipeline script)* |
| Install automatically | ✅ |
| Installer | Install from adoptium.net |
| Version | `jdk-17.0.5+8` |

---

## Step 9 - Add GitHub Credentials in Jenkins

Go to **Manage Jenkins** → **Credentials** → **Add Credentials**:

| Field | Value |
|---|---|
| Username | GitHub username |
| Password | Personal Access Token (PAT) from GitHub Developer Settings |
| ID / Description | `github` *(use same name in pipeline script)* |

---

## Notes

**Check if a port is open in Linux:**
```bash
# Classic way
netstat -tuln | grep 8080

# Modern way
ss -tuln | grep 8080

# Check remote port connectivity
telnet 192.34.1.5 8080
```


Step 10: write the jenkins pipeline

Lets understand how can we write pipeline effectively in jenkins and understand details-
pipeline {
    agent any
    stages {
        stage('Hello') {
            steps {
                echo 'Hello, World!'
            }
        }
    }
}

agent any -> Tells Jenkins **where to run** this pipeline. `any` means "use whatever agent/node is available" — could be the master node or any connected worker. You could also specify `agent { label 'linux' }` to target a specific machine.

Injenkins whenever you are

NOTE:-
Workspace - workspace is the directory in agent/node where git check out (stotes) the source code, runs build command, stores artifacts and rus the pipeline for specific job. workspace is basically a 'working folder' for a job run.

when the we are running a pipeline, if its fetching the code from SCM/git, its mandatory to keep a local copy in jenkins/nodes/agents in workspace.

the default path in jenkins is var/lib/jenkins/workspace/job-name

**Key point:** Each agent creates its own isolated workspace directory. They do NOT share a filesystem automatically.

The Critical Problem This Creates in Production

Since Agent-1 and Agent-2 have **separate workspaces**, the `target/app.jar` built on Agent-1 is **NOT automatically available** on Agent-2.
You must explicitly transfer artifacts using one of these strategies:
```
Strategy 1 — stash/unstash (shown above)
  Agent-1 builds → stash to Jenkins master → Agent-2 unstash

Strategy 2 — Shared NFS/Network Mount
  Both agents mount the same network path as workspace

Strategy 3 — Artifact Repository (Production Best Practice)
  Agent-1 → publish to Nexus/Artifactory → Agent-2 pulls from it

Strategy 4 — Docker Registry
  Agent-1 builds image → pushes to ECR/DockerHub → Agent-2 pulls & deploys
```
### Workspace Lifecycle in Production
```
Job Triggered
     ↓
Agent Allocated
     ↓
Workspace Created (or reused if exists)
     ↓
SCM Checkout (git clone / pull into workspace)
     ↓
Steps Execute inside workspace
     ↓
Post-build actions run
     ↓
Workspace stays on disk (until cleaned)
```

create a jenkins file and store..its already stored in https://github.com/diptikanta1234/register-app.git
create these stages by goint to pipeline syntax

once done create a CI pipeline name as 'register-app-ci' -
- definition - pipeline script from scm
- scm - git
- repositories - give the repo url and add credentials
- branch - give the branch name or main
- repo browser - auto
- jenkins path - jenkinsfile ( makesure u have kept under repo)
- apply & save

step 11:- setup sonarqube
create a ec2 instance with name sonarQube -> ubuntu with t3.medium -> 
connect to the ec2 and install postgresql n sonarqube.
allow 9000 in SG..once everything is done reboot the ec2 once.
add a user sonar and group as sonar..give the perssion

https://www.youtube.com/watch?v=e42hIYkvxoQ&t=7821s

refer the video from 00:45 onwards to check the installation of sonarqube server, once done then access it via <publicIp>:9000
now integrate sonarQube with jenkins-
username/password - admin

generate a token in sonar qube.  
<img width="559" height="299" alt="image" src="https://github.com/user-attachments/assets/2784a73e-e64a-4842-b7ec-efb23e40e452" />

user the sonarqube generated token in jenkins -
- manage jenkins
- credentials
- add new credentials
- kind select as 'secret text'
- in secret pate the 'generated token context'
- id/description - give a name as 'jenkins-sonarqube-token'
- then create it

<img width="640" height="218" alt="image" src="https://github.com/user-attachments/assets/663b6366-1c37-46cf-9efd-6ebd83da6824" />

now install sonarqube plugins -> configure 'sonarqube server' and 'sonarqube scanner' -
- sonarqube scanner plugins
- sonar quality gates
- quality gates
- then istall those
- click restart checkbox to restart the jenkins server once the installation is done

for 'sonarqube server' - 
go to manage jenkins -> system -> sonarqube installation -> add sonarQube 
 - name - give a name as sonaqube-server ( use the same name in pipeline)
 - server url - http://<privateIp>:9000 //if the ec2 present in same vpc
 - server authentication token - 'jenkins-sonarQube-token' ( selet the token that was created by us in jenkins by using the token of sonarqube to connect sonarQube API/tool)
 - apply & save

for 'sonarqube scanner' -
go to manage jenkins -> global tool configuration -> sonarqube scanner installarion ->
- add sonarqube scanner
- install automatcally ( it will take the latest version of sonarqube scanner )
- apply n save

<img width="661" height="285" alt="image" src="https://github.com/user-attachments/assets/98f9d69e-1032-4d9d-9c00-f9208cd3d85c" />

while writing stages for 'sonarqube' get the code using pipeline synax -
 stage("SonarQube Analysis"){
           steps {
	           script {
		        withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
		        }
	           }	
           }
       }
how it works ->
sh "mvn sonar:sonar"

Executes the SonarQube Maven plugin goal.
Maven reads sonar-project.properties or pom.xml for project metadata (project key, name, version).
It compiles, analyzes the source code and sends the report to SonarQube server — covering:

Code smells, bugs, vulnerabilities
Code coverage (if Jacoco/Surefire reports exist)
Duplications, complexity metrics


At the end, SonarQube returns a task ID (stored as .scannerwork/report-task.txt) which Jenkins uses in the next stage.

stage("Quality Gate"){
    steps {
        script {
            waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
        }	
    }
}
```

**`waitForQualityGate`** is a blocking step that **polls SonarQube** for the analysis result.

**How it works internally:**
```
Jenkins  ──── polls ────►  SonarQube API
                           /api/qualitygates/project_status?analysisId=<id>
         ◄─── returns ───  { "status": "OK" | "ERROR" | "WARN" }
```

- Jenkins reads the **task ID** generated in Stage 1 from the workspace.
- It repeatedly hits the SonarQube webhook/API until the background analysis is **complete**.
- For this to work reliably, you should configure a **SonarQube Webhook** pointing back to Jenkins:
```
  SonarQube → Administration → Webhooks → Create
  URL: http://<jenkins-url>/sonarqube-webhook/
```

**`abortPipeline: false`**

This is the critical flag controlling behavior on Quality Gate failure:

| Value | Behavior |
|---|---|
| `abortPipeline: true` | Pipeline **fails & aborts** if Quality Gate is ERROR |
| `abortPipeline: false` | Pipeline **continues** even if Quality Gate fails — result is marked UNSTABLE |

In production pipelines, you'd typically set this to `true` to enforce standards.
```
add the above stage to jenkins file then run build-

<img width="707" height="339" alt="image" src="https://github.com/user-attachments/assets/9aadf77a-5368-42f5-9055-e920d079b45f" />

check the build that got generated from sonarQube-

step 11: configure docker with jenkins
install plugins -
- docker
- docker pipeline

now setup credentials of docker hub with jenkins to push/pull -
manage jenkins -> credentials -> add credentials 
- kind - username and password
- username - uname of dockerhub
- password - enter the token generated in dockerhub ( login to dockerhub - a/c setting - security - new token )
- id/description - dockerhub ( use same name in ci/cd)

now add the pipeline script for docker build n push, trivy scan, cleaning work space. also add some environmental variables.

step 12: create a EKS bootstrap server 
create a ec2 instance as 'eks-bootstrap-server' - ubuntu - t3.micro - 8gb ebs - launch it
install aws cli
install kubectl
install eksctl

create a iam role -> select service ec2 -> add administrative permission -> give role name ' eksctl-role'. then modify and apply iam role by selecting the ec2 instance.

create the eks cluster using eksctl command from boot stap server.
<img width="359" height="41" alt="image" src="https://github.com/user-attachments/assets/e182d589-9fe9-43bd-9758-dc8bccaf9960" />
kubectl get nodes

step 13 - configure argocd
in the same boot stap server, create a namespace as 'argocd' and install argocd -
<img width="592" height="100" alt="image" src="https://github.com/user-attachments/assets/dc93ce68-32aa-4928-a232-ca207bca944e" />

kubectl get pods -n argocd

to interact with api server , from argocd we need to install argocd cli. give it executable pemission.
now expose the argocd UI to external world through ALB.

<img width="836" height="350" alt="{82E54F25-F031-430D-A601-94CC0B58AC78}" src="https://github.com/user-attachments/assets/fc5bc43f-f8f4-4f7c-80bb-910ac7c727fd" />
curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64

chmod +x /usr/local/bin/argocd

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc -n argocd

get the password of argocd and decode it -
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml

echo U2Q2ZXFyN0dWazlUb3drTQ== | base64 --decode

access argocd ui throug LB and give username as admin and password that is  decoded.
change the password in aargocd.

now we need to pull the eks cluster info to argocd. for that login argocd from bootstrap server-
argocd login a661d50fa978944729aaa0788b166e52-1337611090.ap-south-1.elb.amazonaws.com --username admin

argocd cluster list

kubectl config get-contexts

argocd cluster add i-08b83e50d15eca49a@virtualtechbox-cluster.ap-south-1.eksctl.io --name virtualtechbox-eks-cluster

argocd cluster list
```

---

**Output & Explanation:**

**1st `argocd cluster list`** — Shows only the default in-cluster:
```
SERVER                       NAME        VERSION  STATUS   MESSAGE
https://kubernetes.default.svc  in-cluster           Unknown  Cluster has no applications and is not being monitored.
```

**`kubectl config get-contexts`** — Lists available kubeconfig contexts:
```
CURRENT   NAME                                                          CLUSTER                                              AUTHINFO
*         i-08b83e50d15eca49a@virtualtechbox-cluster.ap-south-1.eksctl.io   virtualtechbox-cluster.ap-south-1.eksctl.io   i-08b83e50d15eca49a@virtualtechbox-cluster.ap-south-1.eksctl.io
```

**`argocd cluster add`** — Registers the EKS cluster with ArgoCD:
```
WARNING: This will create a service account `argocd-manager` on the cluster referenced 
by context `i-08b83e50d15eca49a@virtualtechbox-cluster.ap-south-1.eksctl.io` with full 
cluster level privileges. Do you want to continue [y/N]? y

INFO[0005] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0005] ClusterRole "argocd-manager-role" created
INFO[0005] ClusterRoleBinding "argocd-manager-role-binding" created
INFO[0010] Created bearer token secret for ServiceAccount "argocd-manager"
Cluster 'https://F0BB33878900D8F0BEC9A1A260E394C6.gr7.ap-south-1.eks.amazonaws.com' added
```

**2nd `argocd cluster list`** — Now shows both clusters registered:
```
SERVER                                                                         NAME                      VERSION  STATUS   MESSAGE
https://F0BB33878900D8F0BEC9A1A260E394C6.gr7.ap-south-1.eks.amazonaws.com    virtualtechbox-eks-cluster          Unknown  Cluster has no applications and is not being monitored.
https://kubernetes.default.svc                                                  in-cluster                          Unknown  Cluster has no applications and is not being monitored.

<img width="840" height="354" alt="{44284A8B-157B-451B-862D-814960400FC8}" src="https://github.com/user-attachments/assets/23287756-b0f4-4480-b3a4-42420c5c56e7" />

now check in argocd - settings - cluster -> 2 clusters shoukd be visible








