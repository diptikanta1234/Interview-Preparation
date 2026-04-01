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
i have named as 'jenkins-agnt' with ubuntu os
