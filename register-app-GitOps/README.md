Automating a release process through CI/CD -

<img width="1238" height="407" alt="image" src="https://github.com/user-attachments/assets/b3d21086-f5e8-4550-bb87-22fc36a0899f" />

Here the CI job ends and CD job gets triggered automatically.
Our CD job will first update the build number and relese number in deployment.yaml in the gitOps repo in the github, then ArgoCD will pull the manifest file and deploy the resources on the EKS cluster.

once the CD job is finished it will send the notification over slack and send the email as well.

<img width="1256" height="691" alt="image" src="https://github.com/user-attachments/assets/16d97e33-b98f-4a56-bab0-5e7a4f75eae4" />

step1 - Create a jenkins-master:
here i have selected a ec2 instance with ubuntu OS and t2.small. access it through mobaxterm.
then instal jenkins by following the documentation steps

Step2 - create a jenkins-agent:
i have named as 'jenkins-agnt' with ubuntu os -> take t3.small -> 15gb gp3 volume
change the hostname to 'jenkins-agent' -> then reboot the syster 'sudo init 6'

install java jdk17 in my server
install docker
give full rights to the current user ' sudo usermod -aG $USER '

open /etc/ssh/sshd_config
uncomment the pubKeyAuthentication and AuthorizedKeyFiles -
<img width="641" height="93" alt="image" src="https://github.com/user-attachments/assets/6856b696-ece6-4859-917c-08538317ad7c" />

Now go to 'jenkins-master' and change the same in /etc/ssh/sshd_config
then reload the ssh service 'sudo service sshd reload ' in bothjenkins master and agent.

from jenkins-master, generate ssh-key using 'ssh-keygen '
after generating, go to .ssh and then you will get public and private keys

Now open the public key under .ssh in master node -> copy the contents -> open authorized-key under .ssh in 'jenkins-agent' node then pathe the content after a line.
<img width="1133" height="181" alt="image" src="https://github.com/user-attachments/assets/d2104362-5f6d-4778-8b3b-346f61091ed3" />

open jenkins in web <jenkins-master publicip>8080
then use the password from /var/lib/jenkins/secret/initialadminpassword

go to manage jenkins-> user 'system configuration' go to 'Nodes' -> 'built-in ode' -> configure -> make number of executor 0 -> save.
Note - The number of executors defines how many jobs/tasks a Jenkins node can run simultaneously (in parallel).
Node: built-in node
Executors = 0

┌──────────────┐
│ NO Executors │ ── No jobs can run here ❌
└──────────────┘
Job A comes in → goes to agent node ✅

┌─────────────────┐
                    │  Jenkins Master  │
                    │  (built-in node) │
                    │  Executors = 0   │ ← only schedules & manages
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
      │ Agent Node 1 │ │ Agent Node 2 │ │ Agent Node 3 │
      │ Executors=2  │ │ Executors=2  │ │ Executors=2  │
      │ runs jobs ✅ │ │ runs jobs ✅ │ │ runs jobs ✅ │
      └──────────────┘ └──────────────┘ └──────────────┘

  then add 'new node' -> give name 'jenkins-agent' 
  set Executors=2 
  remote homedirectory - /home/ubuntu 
  labels - jenkins-agent
  usage - user this node ASAP
  launch method - launch agent via ssh
  host - give the PrivteIp of jenkins-agent ec2 if it resides in same vpc otherwise give publicIp
  credentials -> add jenkinkins 
                  -> kind - ssh username with private key
                  -> description, username - jenkins-agent
                  -> username - ubuntu
                  -> private-key -> enter directly -> paste the private Key of 'jenkins-master'
  
<img width="1131" height="566" alt="image" src="https://github.com/user-attachments/assets/e672086a-c52d-49d8-b38f-d50c02fe8cb2" />
<img width="1199" height="546" alt="image" src="https://github.com/user-attachments/assets/30862907-d90e-448b-8970-e7452debb55b" />

now one of my agnt is added successfully.

go to jenkins -> create a pipeline job as test -> pipeline script as 'hello world' -> save -> do build now.
check the console output and check if the build is done in jenkins-agent. which concludes the connectvity from agent to master is successful.

Note -
how to check if a port is open or not in linux -
netstat -tuln | grep 8080
tcp   0   0  0.0.0.0:8080   0.0.0.0:*   LISTEN   ← port is OPEN ✅

ss-tuln | grep 8080  -> modern replacement of netstat
telnet <ip of host 192.34.1.5> 8080 -> check if the connectivity is successful in that server over that port

Integrate github with jenkins:
go to manage jenkins -> plugins ->   
- maven
- pipeine maven integration
- eclipse tumrin installer
then install those plugins.

now configure those plugins-
go to agage jenkins - tools ->
maven installation -> add maven 
        - name - maven3 ( use the same name in pipeline script)
        - version 3+
        - apply autoatically n save
jdk installation -> add jdk 
        - name - java17 ( use the same name in pipeline script)
        - install automatically
        - add installer 
        - install from adaptium.net
        - select jdk version jdk17.0.5+8
        - apply n save
        
add the credentials in jenkins-
manage jenkins -> credentials -. add credentials - 
  username - give github username
  password - give the PersonalAccessToken(PAT) created from developer setting of github
  id, description - give a name as 'github' ( use the same name in pipeline script)

Now create a pipeline script or jenkins file: 30:36


        




