
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

add the above stage to jenkins file then run build-

<img width="707" height="339" alt="image" src="https://github.com/user-attachments/assets/9aadf77a-5368-42f5-9055-e920d079b45f" />

check the build that got generated from sonarQube-






