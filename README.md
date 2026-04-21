# Interview-Preparation for Azure

=================================
Subscriptions -
<img width="575" height="304" alt="image" src="https://github.com/user-attachments/assets/37cb102d-7e7c-4e3b-b897-e827bec27e89" />

<img width="538" height="266" alt="image" src="https://github.com/user-attachments/assets/b69df973-aaf9-47d2-af63-854dccc599f6" />

<img width="554" height="297" alt="image" src="https://github.com/user-attachments/assets/c5b05c41-3727-405b-8334-1d275508645e" />
<img width="566" height="297" alt="image" src="https://github.com/user-attachments/assets/b4f0faf5-6112-4fcc-9d97-6479b71ee430" />

<img width="563" height="294" alt="image" src="https://github.com/user-attachments/assets/ad6d638d-91c3-451f-88c2-984a8c5fe39b" />
<img width="560" height="295" alt="image" src="https://github.com/user-attachments/assets/f4390166-3f43-45e2-8935-73002b66fe0a" />
<img width="566" height="359" alt="image" src="https://github.com/user-attachments/assets/66953c9e-4469-4716-ba38-cd47404e2276" />
<img width="1538" height="810" alt="image" src="https://github.com/user-attachments/assets/e1404803-eee2-4487-8194-a187d7eeb36e" />
<img width="1571" height="913" alt="image" src="https://github.com/user-attachments/assets/aadd055d-1cda-4e13-ad92-f7c2f6889787" />
<img width="1257" height="928" alt="image" src="https://github.com/user-attachments/assets/60915379-b9ce-4da9-a052-26452ec92381" />
<img width="1672" height="959" alt="image" src="https://github.com/user-attachments/assets/9d97502e-2a6b-49b6-86bc-7e1471532991" />

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

<img width="809" height="338" alt="image" src="https://github.com/user-attachments/assets/f7aff3b4-3b9b-4729-a5f8-a10f795bba12" />
<img width="940" height="391" alt="image" src="https://github.com/user-attachments/assets/63f8830a-4d84-4421-a926-265526e61fc3" />
<img width="940" height="533" alt="image" src="https://github.com/user-attachments/assets/1cafb120-c17c-49f0-adae-052662421eb1" />
<img width="940" height="585" alt="image" src="https://github.com/user-attachments/assets/5dd42396-18e1-4ed2-a6f3-18dae25a8343" />
<img width="940" height="315" alt="image" src="https://github.com/user-attachments/assets/a64abe23-7044-4fea-8a08-121af1d8ac69" />
<img width="940" height="553" alt="image" src="https://github.com/user-attachments/assets/cf7a11fe-cd25-4dad-aa74-2fefe3b2c1a5" />


