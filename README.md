# Interview-Preparation
for interview questions and answers

JENKINS:-

Q.Shared library in Jenkins and how can you use it?
Jenkins Shared Library — concept
A Shared Library is a centralized repository of reusable Groovy code that multiple Jenkins pipelines can import. Instead of copy-pasting the same logic across dozens of Jenkinsfiles, you define it once and reference it everywhere.
Real-world scenario: Imagine 50 microservices, each needing the same deploy, test notification, and Docker build steps. Without a shared library, every Jenkinsfile duplicates hundreds of lines. With one, each pipeline imports a single method.

<img width="505" height="302" alt="image" src="https://github.com/user-attachments/assets/da851d17-310e-4e62-8b81-45aa70de7063" />

Real-time behaviour
Jenkins clones the library repo per build (or uses a cached version).
If you push a fix to the library, all new pipeline runs pick it up automatically.
You can pin a version tag to prevent unexpected changes.

my-shared-library/
├── vars/
│   ├── buildAndPush.groovy          ← callable as buildAndPush(...) in any pipeline
│   ├── deployToK8s.groovy
│   └── sendSlackNotification.groovy
│
├── src/
│   └── com/
│       └── myorg/
│           └── Utils.groovy         ← reusable Groovy classes (imported in vars/)
│
└── resources/
    └── com/myorg/
        └── deploy-template.yaml     ← non-code files (loaded via libraryResource())

vars/ — this is the most used folder. Each .groovy file becomes a global pipeline step with the same name. vars/buildAndPush.groovy → call buildAndPush() in any Jenkinsfile.
src/ — standard Groovy classes. Used for complex logic, utility functions, helper classes. Must follow Java-style package structure.
resources/ — static files (YAML templates, shell scripts, configs) that you pull at runtime using libraryResource().

Step 1 — Writing library code
vars/buildAndPush.groovy — simple step:
groovy// The 'call' method makes it usable as a pipeline step
def call(String imageName, String tag = 'latest') {
    echo "Building Docker image: ${imageName}:${tag}"
    sh "docker build -t ${imageName}:${tag} ."
    sh "docker push ${imageName}:${tag}"
}

Step 2 — Registering in Jenkins
Go to Manage Jenkins → System → Global Pipeline Libraries and fill in:
FieldValueNamemy-shared-libDefault versionmain (or a tag like v1.2)Retrieval methodModern SCM → GitSource code URLhttps://github.com/myorg/my-shared-library.gitLoad implicitly✅ (optional — auto-loads without @Library)

// --- Option A: explicit import (recommended) ---
@Library('my-shared-lib') _          // single underscore = import all vars
@Library('my-shared-lib@v1.2') _     // pin to a specific version/tag/branch

// --- Option B: import a specific class ---
@Library('my-shared-lib')
import com.myorg.Utils

// --- Option C: multiple libraries ---
@Library(['my-shared-lib', 'another-lib@main']) _


Use the library in pipeline to import the groovy code/function -
@Library('my-shared-lib') _
import com.myorg.Utils

pipeline {
    agent any

    environment {
        IMAGE = "myorg/my-app"
        TAG   = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Build & Push') {
            steps {
                // calls vars/buildAndPush.groovy → def call(...)
                buildAndPush("${IMAGE}", "${TAG}")
            }
        }

        stage('Deploy') {
            steps {
                // calls vars/deployToK8s.groovy → def call(Map config)
                deployToK8s(
                    name:      'my-app',
                    image:     "${IMAGE}:${TAG}",
                    namespace: 'production'
                )
            }
        }

        stage('Notify') {
            steps {
                script {
                    // using a src/ class directly
                    Utils.printEnv(this)
                    sendSlackNotification(channel: '#deployments', status: 'SUCCESS')
                }
            }
        }
    }
}

Real-time flow when a pipeline runs

Jenkins sees @Library('my-shared-lib') and checks its internal cache.
If not cached (or load implicitly = false), it clones the library repo at the specified branch/tag.
All vars/*.groovy files become available as first-class pipeline steps — no imports needed for them.
src/ classes must be explicitly imported.
Any change pushed to the library's main branch is picked up by the next build automatically unless you've pinned a version.

Q.What is a Multibranch Pipeline?
A Multibranch Pipeline is a Jenkins job type that automatically discovers, creates, and manages separate pipelines for every branch in your Git repo. You don't manually create a job per branch — Jenkins scans the repo, finds a Jenkinsfile in each branch, and spins up a dedicated pipeline for it automatically.

Master vs Slave — who does what
The master (controller) never runs build logic. It only orchestrates: it scans branches, manages the build queue, stores logs, serves the UI, and sends jobs to agents. This keeps the master lightweight and stable.
The slaves (agents) do all the actual work — compiling, running tests, building Docker images, deploying to Kubernetes. Each agent registers with the master using a label (like linux, docker, windows), and the master routes jobs to agents matching the label you specify in your Jenkinsfile.

pipeline {
    // master dispatches this job to any agent labeled 'linux'
    agent { label 'linux' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm  // Jenkins auto-injects the right branch
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
                // or: sh 'docker build -t myapp:${env.BUILD_NUMBER} .'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Integration Tests') {
            // Only run on develop and main — skip for feature branches
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                sh 'mvn verify -Pintegration-tests'
            }
        }

        stage('Code Quality') {
            when { branch 'main' }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }

    post {
        failure {
            slackSend channel: '#alerts',
                      message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} on branch ${env.BRANCH_NAME}"
        }
        success {
            echo "Branch ${env.BRANCH_NAME} built successfully"
        }
    }
}

Q. How you connect a master to agent in jenkins?
<img width="575" height="329" alt="image" src="https://github.com/user-attachments/assets/9540edf6-ac15-4b6a-bff6-31c1543c1f13" />

On Jenkins master, generate a keypair:
bashssh-keygen -t rsa -b 4096 -f jenkins-agent-key -C "jenkins-agent"
# produces: jenkins-agent-key (private) and jenkins-agent-key.pub (public)

# Copy public key to agent
ssh-copy-id -i jenkins-agent-key.pub jenkins@<AGENT_IP>
In Jenkins UI:

Go to Manage Jenkins → Credentials → Global → Add Credentials
Kind: SSH Username with private key
Username: jenkins, paste the private key content
Give it an ID like linux-agent-ssh-key

Then go to Manage Jenkins → Nodes → New Node:
FieldValueNode namelinux-agent-1TypePermanent AgentRemote root directory/home/jenkins/workspaceLabelslinux dockerLaunch methodLaunch via SSHHost192.168.1.50 (agent IP)Credentialslinux-agent-ssh-keyHost Key VerificationKnown hosts or Non-verifying
Click Save → Launch agent — Jenkins SSHes in, copies agent.jar, and the node goes online.

Q. how can you check if a port is opened and the agent is reachable over a port ?

telnet <AGENT_IP> <PORT>

# command 1
telnet 192.168.1.50 22      # SSH agent
telnet 192.168.1.50 50000   # JNLP agent port
telnet 192.168.1.50 8080    # Jenkins master UI

# Success output:
# Trying 192.168.1.50...
# Connected to 192.168.1.50.    ← port is OPEN

# Failure output:
# Trying 192.168.1.50...
# telnet: connect to address 192.168.1.50: Connection refused   ← port CLOSED
# or just hangs with no response  ← firewall BLOCKING

# command 2
nc (netcat) — fastest, works everywhere

nc -zv <AGENT_IP> <PORT>
# -z = scan only (don't send data)
# -v = verbose
nc -zv 192.168.1.50 22

# command 3
ss / netstat — check FROM the agent side (is anything listening?)
# Run ON the agent machine itself
ss -tlnp | grep 50000       # is JNLP port listening?
ss -tlnp | grep 22          # is SSH listening?
netstat -tlnp | grep 50000  # older systems

# Output when listening:
# LISTEN  0  128  0.0.0.0:50000  0.0.0.0:*  users:(("java",pid=1234))

Q. how can we define parameters in jenkins...what's their use

Parameters in Jenkins let you pass dynamic values into a pipeline at runtime — instead of hardcoding things like environment names, image tags, or branch names, you make them inputs so the same pipeline can behave differently each time it runs.

<img width="506" height="320" alt="image" src="https://github.com/user-attachments/assets/dd26fa12-182b-4ec5-b84b-40b68ab21685" />

pipeline {
    agent any

    parameters {
        // Free text — single line
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'Docker image tag to deploy'
        )

        // Checkbox
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run test suite before deploying?'
        )
    }

    stage('Docker Build & Push') {
    steps {
        sh """
            docker build -t myorg/myapp:${params.IMAGE_TAG} .
            docker push myorg/myapp:${params.IMAGE_TAG}
        """
    }
}

stage('Deploy to K8s') {
    steps {
        sh """
            kubectl set image deployment/myapp \
              app=myorg/myapp:${params.IMAGE_TAG} \
              -n ${params.ENVIRONMENT}
        """
    }
}

}
