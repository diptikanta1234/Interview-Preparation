# Interview-Preparation for Azure

=================================
<img width="2928" height="2884" alt="image" src="https://github.com/user-attachments/assets/2d20e010-96c6-463a-a331-184e22f08008" />

**PIM** :-
PIM in azure normally refers to MS entraId Privileged Identity Management .
It helps organisation to manage, control and monitor resources or users.

NOTE - Instead of giving the full access permanently, give access for limited time. works like STS in aws.

What PIM does

Instead of giving an administrator permanent access:
```
Just-in-time (JIT) access — users activate privileged roles only when needed.
Time-limited access — activated permissions can automatically expire.
Approval workflows — require someone to approve role activation.
MFA enforcement — require MFA when activating a privileged role.
Reason/ticket requirements — users can be required to provide justification or a ticket number.
Auditing — track who activated which role, when, and why.
Access reviews — periodically review whether users still need privileged access.
Example

Without PIM:

Alice → Owner role → Permanent access

With PIM:

Alice → Eligible for Owner → Requests activation → MFA/approval → Owner for 1 hour → Access expires

```



Subscriptions -


we can use 'Cost management' to check bills or invoice.

for high availability azure uses below -
<img width="834" height="371" alt="image" src="https://github.com/user-attachments/assets/823c6850-1e2f-48fc-aa04-f618a2dc5800" />
<img width="1150" height="622" alt="image" src="https://github.com/user-attachments/assets/bf5afaf4-46b7-422d-9a58-488e78bf5a37" />

Availability sets are present in a single Availability zone (AZ). If aby distaster comes in that AZ, VMs may get down-
<img width="1136" height="632" alt="image" src="https://github.com/user-attachments/assets/0d86b12c-d2a4-4351-ae21-6aa22aebc8dc" />

VMSS( virtual achine scale set)...similar to auto scaling in AWS


<img width="921" height="594" alt="image" src="https://github.com/user-attachments/assets/4ea8a1eb-142a-4a1a-85de-2f91c91cd83d" />

<img width="1214" height="599" alt="image" src="https://github.com/user-attachments/assets/fd7021d3-1200-4ff9-bc6e-c2511db25572" />

VNET ( Virtual Network ):-
its similar to VPC.
Azure Bastion is a paid service that provides secure RDP/SSH connectivity to your virtual machines over TLS. When you connect via Azure Bastion, your virtual machines do not need a public IP address

Azure Firewall
Azure Firewall is a managed cloud-based network security service that protects your Azure Virtual Network resources.

Custom data and cloud init
Pass a cloud-init script, configuration file, or other data into the virtual machine while it is being provisioned. The data will be saved on the VM in a known location

User data
Pass a script, configuration file, or other data that will be accessible to your applications throughout the lifetime of the virtual machine. Don't use user data for storing your secrets or passwords.


<img width="1269" height="671" alt="image" src="https://github.com/user-attachments/assets/0e799991-a445-488a-9683-516257cfc4de" />

i created vnet -> bastion -> firewall -> a vm in private subnet to install nginx in azure portal. now i allowed port 22,80,443 as inbound while creating the VM. now i connected to vm through bastion and updating package but The apt-get update is failing because the VM can't reach azure.archive.ubuntu.com:80 — it's timing out.

Root Cause
Even though you have a Firewall, traffic from the private subnet isn't being routed through it to reach the internet. You're missing two things:

-> A route table directing internet-bound traffic (0.0.0.0/0) → Azure Firewall
-> DNAT/Network rules on the Firewall allowing outbound HTTP/HTTPS

step 1 : Go to Route Tables → Create new 

Route name -> defaulti-Private-subnet-to-firewall

Address prefix -> 0.0.0.0/0

Next hop type -> Virtual appliance

Next hop address -> Private IP of your Azure Firewall (e.g. 10.0.0.4)

Now add rt defaulti-Private-subnet-to-firewall to priate subnet. ( in this way the private subnet will get internet when its not having public ip). Bastion is to connect through ssh. go to default subnet -> add the rt as defaulti-Private-subnet-to-firewall. then add the NS that is created while creting V to allow port - 22,80,443.
<img width="1050" height="451" alt="image" src="https://github.com/user-attachments/assets/bcadb918-e846-410a-a7d4-f86637059af3" />


Step 2 — Add a Network Rule to Your Azure Firewall

In Azure Portal → Firewall → Rules → Network Rule Collection → Add:

SourceYour private subnet CIDR (e.g. 10.0.1.0/24)

Destination - * 

Destination Ports - 80, 443



I will configure the policy in firewall so that if any body accessing to public Ip over a port ( <Public Ip of firewall>:4000 ), then my website should display which is inside vm.
<img width="1751" height="484" alt="image" src="https://github.com/user-attachments/assets/d8992b71-4c0c-4ed7-9b37-05fad6e82fbe" />

Azure interview question by Veeramala-

<img width="940" height="355" alt="image" src="https://github.com/user-attachments/assets/999689d2-a3ed-4711-8401-3334e24f97d0" />
<img width="940" height="371" alt="image" src="https://github.com/user-attachments/assets/3d19721c-7818-47ca-8da9-722d0cd11278" />
<img width="940" height="420" alt="image" src="https://github.com/user-attachments/assets/8e05a9af-726b-4822-8925-f68cb6a993a0" />
<img width="782" height="247" alt="image" src="https://github.com/user-attachments/assets/51c782e3-d33d-4ff5-a37f-789380cf1789" />

Aws user data is similar to azure custom data.
Azure user data is completely different.

Lets say we are creating a vm/ec2 through cli or ui or IAC, we need some dependency or application code to be installed during the creation of vm. So we are keeping those in custom data in azure (aws -> user data) as script.

