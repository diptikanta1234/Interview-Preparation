# Interview-Preparation for Azure

=================================

# Azure Enterprise Landing Zone for a Three-Tier Node.js Application

## 1. Overview

This document describes a proposed enterprise-grade Azure landing zone for a three-tier Node.js application hosted on Azure Virtual Machines.

### Design requirements

- **Primary region:** Central India
- **Environments:** Separate Dev and QA subscriptions
- **Application:** Three-tier Node.js application hosted on Azure VMs
- **Database:** Azure SQL Database or Azure SQL Managed Instance (final choice pending)
- **Connectivity:** Hub-and-spoke networking with Azure Firewall
- **Infrastructure as Code:** Terraform
- **Identity:** Microsoft Entra ID, managed identities, and workload identity federation for Azure DevOps
- **Governance:** Management groups, Azure Policy, RBAC, tagging, and budgets
- **Resilience:** Azure Backup, database recovery, and cross-region disaster recovery (DR)

> **Important:** This is a proposed design baseline. Confirm address-space overlap, Azure service availability, quotas, compliance requirements, recovery objectives, and cost before deployment.

---

## 2. Target Azure Architecture

```text
Microsoft Entra ID tenant
  |
  +-- Tenant Root Management Group
        |
        +-- Platform
        |     |
        |     +-- Connectivity subscription
        |     |     +-- Hub VNet
        |     |     +-- Azure Firewall
        |     |     +-- Azure Bastion
        |     |     +-- VPN / ExpressRoute gateway (if required)
        |     |     +-- Private DNS
        |     |
        |     +-- Management subscription (can be an existing subscription)
        |           +-- Azure Monitor
        |           +-- Log Analytics
        |           +-- Security operations and alerting
        |
        +-- Landing Zones
              |
              +-- Non-Production
              |     +-- Dev subscription
              |     |     +-- Dev spoke VNet
              |     |     +-- Web / ingress tier
              |     |     +-- Node.js application tier
              |     |     +-- Private data connectivity
              |     |
              |     +-- QA subscription
              |           +-- QA spoke VNet
              |           +-- Web / ingress tier
              |           +-- Node.js application tier
              |           +-- Private data connectivity
              |
              +-- Production (future)
                    +-- Production workload subscription
```

### Architecture principles

- Keep Dev and QA in separate subscriptions and separate spoke VNets.
- Do not share application VMs, databases, secrets, or environment-specific deployment identities between Dev and QA.
- Share centrally managed connectivity services where appropriate.
- Use a separate production subscription with stricter controls when Production is introduced.
- Place subscriptions in management groups according to governance and ownership requirements, not solely by environment name.
- The platform subscriptions may already exist; do not create duplicates without checking the organization's existing landing zone.

---

## 3. Management Groups and Subscriptions

Suggested initial organization:

| Scope | Suggested name | Purpose |
|---|---|---|
| Tenant root | Existing tenant root | Tenant-wide governance |
| Platform management group | `mg-platform` | Shared platform services |
| Landing zones management group | `mg-landingzones` | Application workload subscriptions |
| Non-production management group | `mg-app-nonprod` | Dev and QA governance |
| Connectivity subscription | `sub-platform-connectivity-001` | Hub networking, firewall, Bastion, DNS |
| Management subscription | `sub-platform-management-001` | Monitoring and operational services |
| Dev subscription | `sub-nodeapp-dev-001` | Dev application workload |
| QA subscription | `sub-nodeapp-qa-001` | QA application workload |
| Future Production subscription | `sub-nodeapp-prod-001` | Production workload |

These are suggested display names; subscription names and IDs should be confirmed in the Azure tenant. The exact management-group structure should match organizational governance requirements.

---

## 4. Naming Convention

### General pattern

```text
<resource-type>-<application>-<environment>-<region>-<instance>
```

Example:

```text
vm-nodeapp-dev-centralindia-001
```

Use these environment codes consistently:

- `dev` — Development
- `qa` — Quality Assurance
- `stg` — Staging
- `prod` — Production

Use the full Azure region name in resource names where practical. Some Azure resources have stricter length, character, or global uniqueness requirements, so validate each name against the relevant resource provider's rules.

### Application resource names

| Resource | Dev example | QA example |
|---|---|---|
| Subscription display name | `sub-nodeapp-dev-001` | `sub-nodeapp-qa-001` |
| Application resource group | `rg-nodeapp-dev-app-001` | `rg-nodeapp-qa-app-001` |
| Network resource group | `rg-nodeapp-dev-network-001` | `rg-nodeapp-qa-network-001` |
| Security resource group | `rg-nodeapp-dev-security-001` | `rg-nodeapp-qa-security-001` |
| Virtual network | `vnet-nodeapp-dev-centralindia-001` | `vnet-nodeapp-qa-centralindia-001` |
| Web subnet | `snet-nodeapp-dev-web-001` | `snet-nodeapp-qa-web-001` |
| Application subnet | `snet-nodeapp-dev-app-001` | `snet-nodeapp-qa-app-001` |
| Private endpoint subnet | `snet-nodeapp-dev-pe-001` | `snet-nodeapp-qa-pe-001` |
| Network security group | `nsg-nodeapp-dev-app-001` | `nsg-nodeapp-qa-app-001` |
| Route table | `rt-nodeapp-dev-app-001` | `rt-nodeapp-qa-app-001` |
| Web VM | `vm-nodeapp-dev-web-001` | `vm-nodeapp-qa-web-001` |
| Application VM | `vm-nodeapp-dev-app-001` | `vm-nodeapp-qa-app-001` |
| Key Vault | `kv-nodeapp-dev-001` | `kv-nodeapp-qa-001` |
| Managed identity | `id-nodeapp-dev-runtime-001` | `id-nodeapp-qa-runtime-001` |
| Log Analytics workspace | `log-nodeapp-dev-001` | `log-nodeapp-qa-001` |
| Recovery Services vault | `rsv-nodeapp-dev-001` | `rsv-nodeapp-qa-001` |
| Application Gateway | `agw-nodeapp-dev-001` | `agw-nodeapp-qa-001` |
| Internal load balancer | `lb-nodeapp-dev-internal-001` | `lb-nodeapp-qa-internal-001` |
| Azure DevOps service connection | `sc-azure-nodeapp-dev-wif` | `sc-azure-nodeapp-qa-wif` |
| Terraform state container | `tfstate` | `tfstate` (in separately scoped state storage/container or key) |

### Shared platform names

| Resource | Suggested name |
|---|---|
| Connectivity resource group | `rg-platform-connectivity-centralindia-001` |
| Hub VNet | `vnet-platform-hub-centralindia-001` |
| Azure Firewall | `azfw-platform-centralindia-001` |
| Azure Bastion | `bas-platform-centralindia-001` |
| Management resource group | `rg-platform-management-centralindia-001` |

### Naming exceptions

- **Storage accounts:** lowercase letters and digits only, 3–24 characters, globally unique. Example pattern: `stnodeappdev001`, if available.
- **Key Vault:** has its own length and character constraints; confirm global availability.
- **Private DNS zones:** use the required DNS namespace, such as `privatelink.vaultcore.azure.net` for the relevant Key Vault private endpoint.
- Do not put secrets, personal data, or sensitive business details in resource names or tags.

---

## 5. Network Design and CIDR Plan

The following CIDRs are a **proposal**, not an approved allocation. Before creating VNets, check for overlap with corporate networks, VPN, ExpressRoute, other Azure VNets, on-premises networks, and the planned DR region.

### Hub VNet — Connectivity subscription

**VNet:** `vnet-platform-hub-centralindia-001`  
**Address space:** `10.40.0.0/16`

| Subnet / purpose | Proposed CIDR | Notes |
|---|---|---|
| Azure Firewall | `10.40.0.0/26` | Dedicated `AzureFirewallSubnet`; at least `/26` |
| Azure Bastion | `10.40.0.64/26` | Dedicated `AzureBastionSubnet`; at least `/26` |
| VPN / ExpressRoute gateway | `10.40.1.0/27` | Validate against selected gateway SKU and requirements |
| Reserved hub services | `10.40.2.0/24` | Allocate only to services that support a regular subnet |

### Dev spoke VNet

**VNet:** `vnet-nodeapp-dev-centralindia-001`  
**Address space:** `10.41.0.0/16`

| Subnet / purpose | Proposed CIDR | Notes |
|---|---|---|
| Web / ingress | `10.41.1.0/24` | Private application backends; public entry should be through WAF where required |
| Node.js application tier | `10.41.2.0/24` | App VMs, with only required inbound flows |
| Private endpoints | `10.41.3.0/24` | Private endpoints and associated DNS design |
| SQL Managed Instance (if selected) | `10.41.4.0/24` | Preliminary only; dedicated, delegated subnet and sizing must be validated |
| Reserved for growth | `10.41.5.0/24` onward | Keep unallocated until required |

### QA spoke VNet

**VNet:** `vnet-nodeapp-qa-centralindia-001`  
**Address space:** `10.42.0.0/16`

| Subnet / purpose | Proposed CIDR | Notes |
|---|---|---|
| Web / ingress | `10.42.1.0/24` | Private application backends; public entry should be through WAF where required |
| Node.js application tier | `10.42.2.0/24` | App VMs, with only required inbound flows |
| Private endpoints | `10.42.3.0/24` | Private endpoints and associated DNS design |
| SQL Managed Instance (if selected) | `10.42.4.0/24` | Preliminary only; dedicated, delegated subnet and sizing must be validated |
| Reserved for growth | `10.42.5.0/24` onward | Keep unallocated until required |

### Network implementation rules

- Use VNet peering between the hub and each spoke.
- Configure UDRs deliberately to direct appropriate outbound traffic through Azure Firewall.
- Validate routing symmetry, return paths, private endpoint behavior, and required Azure platform dependencies before forcing all traffic through the firewall.
- Use NSGs for subnet-level controls and Azure Firewall policies for centrally managed traffic rules.
- Do not assign public IPs to application or database VMs.
- Use Azure Bastion or an approved private administration path; do not expose SSH/RDP directly to the internet.
- Use Application Gateway with WAF for public HTTP/S entry where appropriate.
- Configure private DNS zones and VNet links for the private services used by the application.
- If using SQL Managed Instance, use a dedicated delegated subnet and validate supported subnet sizing, route requirements, and service constraints.
- If using Azure SQL Database with a private endpoint, validate private DNS resolution and disable public network access only after private connectivity is confirmed.
- Confirm CIDR allocation and IP capacity before deploying resources.

### High-level traffic flow

```text
Internet clients
    |
    v
Application Gateway + WAF (HTTPS)
    |
    v
Private web / ingress backends
    |
    v
Node.js application VMs
    |
    v
Private SQL connectivity
    |
    v
Azure SQL Database private endpoint
OR
Azure SQL Managed Instance in its delegated subnet

Controlled outbound traffic:
Workload subnet -> UDR -> Azure Firewall -> approved destination
```

---

## 6. Identity and Access Management

Follow Zero Trust, least privilege, and separation of duties.

### Human identities

- Use Microsoft Entra ID for authentication and centralized identity management.
- Enforce MFA and appropriate Conditional Access policies.
- Use Privileged Identity Management (PIM) for eligible privileged roles where licensed and applicable.
- Separate platform, network, security, database, and application operations roles.
- Avoid permanent subscription Owner permissions for routine work.
- Review access regularly and remove unused role assignments.

### Azure DevOps and Terraform identities

- Use an Azure Resource Manager service connection with workload identity federation (OIDC) where supported.
- Avoid long-lived client secrets for pipeline authentication when federation is available.
- Scope permissions to the minimum required subscription/resource scope.
- Separate platform provisioning permissions from Dev and QA workload deployment permissions.
- Separate plan/validation and apply/deployment permissions where appropriate.
- Protect production approvals and environment permissions when Production is introduced.
- Store Terraform state in a secured remote backend; state can contain sensitive values.

Suggested identity names:

| Purpose | Name |
|---|---|
| Dev runtime identity | `id-nodeapp-dev-runtime-001` |
| QA runtime identity | `id-nodeapp-qa-runtime-001` |
| Dev infrastructure deployment identity | `id-nodeapp-dev-iac-001` |
| QA infrastructure deployment identity | `id-nodeapp-qa-iac-001` |

### Application and VM identities

- Use system-assigned or user-assigned managed identities where appropriate.
- Give application identities only the required Key Vault and data-plane permissions.
- Prefer Microsoft Entra authentication for Azure SQL when supported by the selected configuration.
- Do not place credentials in source code, VM images, pipeline YAML, or ordinary Terraform variables.
- Run the Node.js process under a dedicated non-root operating-system account.

---

## 7. Security Baseline

| Layer | Recommended controls |
|---|---|
| Internet entry | Application Gateway WAF, HTTPS, secure TLS configuration |
| Network | Azure Firewall, NSGs, UDRs, private endpoints, private DNS |
| VM security | Trusted images, patching, disk encryption, endpoint protection |
| Administration | Azure Bastion, private access, least-privilege OS accounts |
| Secrets | Azure Key Vault, managed identities, restricted access |
| Database | Private connectivity, encryption in transit/at rest, auditing |
| Threat detection | Microsoft Defender for Cloud plans selected for workload risk |
| Monitoring | Azure Monitor, Log Analytics, diagnostic settings, alerts |
| Software supply chain | SAST, dependency scanning, artifact integrity, package/image scanning |
| Recovery | Protected backups, restricted deletion, tested restore procedures |

### Node.js application security

- Run the Node.js service as a non-root user.
- Keep Node.js, OS packages, and dependencies patched.
- Store secrets and environment-specific configuration outside source control.
- Use TLS for client-to-application and application-to-database traffic.
- Restrict inbound ports and disable unnecessary services.
- Use private VM access; avoid public IPs on individual workload VMs.
- Implement input validation, secure HTTP headers, appropriate rate limiting, and secure error handling.
- Scan source code and dependencies in CI and block releases according to agreed severity thresholds.

---

## 8. Governance and Compliance

### Azure Policy baseline

Apply policies at the appropriate management group or subscription scope. Begin with audit mode where needed, then move to deny or deploy-if-not-exists after validation.

Recommended policy areas:

- Allowed Azure regions: Central India for primary workload resources and explicitly approved regions for DR.
- Required tags: `Application`, `Environment`, `Owner`, `CostCenter`, `ManagedBy`.
- Approved VM SKUs and supported operating systems.
- Diagnostic settings and centralized logging.
- Encryption and secure network access.
- Restrictions on public IPs and public database access.
- Backup protection for in-scope VMs.
- Restrictions on unauthorized resource types and locations.
- Consistent security configuration and monitoring coverage.

Do not apply a tenant-root deny policy that blocks required platform services or approved DR resources.

### Required resource tags

| Tag | Dev example | QA example |
|---|---|---|
| `Application` | `NodeApp` | `NodeApp` |
| `Environment` | `Dev` | `QA` |
| `Owner` | `PlatformEngineering` | `PlatformEngineering` |
| `CostCenter` | Approved cost centre | Approved cost centre |
| `ManagedBy` | `Terraform` | `Terraform` |
| `Criticality` | Based on business impact | Based on business impact |
| `DataClassification` | Based on actual data | Based on actual data |
| `DRRequired` | Approved value | Approved value |
| `BackupPolicy` | Approved policy name | Approved policy name |

Use the organization's actual regulatory obligations to define compliance requirements. Azure Policy and Defender for Cloud help assess controls but do not, by themselves, guarantee compliance.

---

## 9. Database Architecture Decision

For a new Node.js application, **Azure SQL Database is the preferred starting point if the application does not require SQL Server instance-level features** that necessitate Managed Instance.

| Consideration | Azure SQL Database | Azure SQL Managed Instance |
|---|---|---|
| Management overhead | Lower | Higher |
| Dedicated VNet subnet | Not required for the database service itself | Required, dedicated and delegated |
| Private access | Private endpoint supported | VNet-local connectivity; additional private endpoint options depend on requirements |
| SQL Server instance-level compatibility | More limited | Broader compatibility |
| Typical fit | New cloud-native applications | Applications needing broader SQL Server compatibility |
| DR options | Failover groups, geo-replication, geo-restore | Managed Instance failover groups and supported recovery options |

Before choosing, assess:

- Required SQL Server features and compatibility.
- Performance, availability, and scaling requirements.
- Private networking and DNS requirements.
- Backup retention and regional recovery requirements.
- Cost and licensing.
- Regional availability, quotas, and supported service configuration.

For Azure SQL Database, disable public network access only after the private endpoint and DNS path have been tested. For Managed Instance, validate its dedicated subnet, address capacity, routing, and deployment requirements before creating the VNet resources.

---

## 10. Backup and Disaster Recovery

Define and obtain business approval for **RPO (Recovery Point Objective)** and **RTO (Recovery Time Objective)** before selecting final service configurations. The actual achievable RPO/RTO depends on service tier, configuration, outage scenario, and tested procedures.

| Component | Recommended approach |
|---|---|
| Azure VMs | Azure Backup with approved retention and backup policy |
| VM regional recovery | Evaluate Azure Site Recovery for supported VM replication and failover |
| Azure SQL Database | Automated backups; evaluate zone redundancy, failover groups, and geo-restore |
| Azure SQL Managed Instance | Automated backups and supported cross-region failover/recovery design |
| Terraform state | Restricted remote state, recovery/versioning protections where supported |
| Key Vault | Soft delete, purge protection, and tested recovery procedure |
| Application artifacts | Versioned, access-controlled releases in the artifact repository |
| DR environment | Documented network, DNS, identity, compute, and database recovery plan |

### DR design considerations

- Select a secondary region only after checking service availability, region pairing, quotas, VM SKUs, database support, and data residency obligations.
- Ensure the DR region has a planned IP address range that does not overlap with primary or corporate networks.
- Document DNS changes, traffic redirection, and application configuration changes required during failover.
- Protect backups against unauthorized deletion and credential compromise.
- Test restore operations regularly.
- Conduct planned failover and failback exercises and record measured recovery performance.
- Treat backup and replication as separate controls: backups support recovery from deletion/corruption; replication can support service recovery after a regional outage.
- Ensure DR resources and dependencies are actually provisioned or can be deployed within the approved RTO.

---

## 11. Terraform Repository Structure

```text
azure-landing-zone/
├── bootstrap/
│   └── terraform-state/
├── modules/
│   ├── management-groups/
│   ├── policy-assignments/
│   ├── hub-network/
│   ├── spoke-network/
│   ├── firewall/
│   ├── bastion/
│   ├── app-gateway-waf/
│   ├── linux-vm/
│   ├── managed-identity/
│   ├── key-vault/
│   ├── azure-sql-database/
│   ├── sql-managed-instance/
│   ├── monitoring/
│   └── backup/
├── platform/
│   └── centralindia/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── qa/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
└── policies/
    ├── allowed-regions/
    ├── required-tags/
    ├── secure-networking/
    └── diagnostics/
```

### Terraform implementation rules

- Pin Terraform and AzureRM provider versions to tested versions.
- Use reusable modules with explicit inputs and outputs.
- Keep environment-specific values in environment configuration; do not hard-code credentials.
- Use separate state keys and preferably separate containers or storage accounts with tightly scoped access for Dev, QA, and platform.
- Bootstrap the state backend separately from the resources managed by that backend.
- Secure state storage with least-privilege access, private connectivity where practical, and supported recovery/versioning protections.
- Use the AzureRM backend's supported state locking behavior.
- Protect pipeline plan/apply stages and review changes before applying.
- Do not assume that Terraform state is free of secrets; avoid persisting secrets where possible and restrict access to state.
- Validate names, CIDRs, provider constraints, service availability, and policy effects before deployment.

---

## 12. Monitoring and Operations

Centralize monitoring across Dev and QA while preserving appropriate environment access boundaries.

Recommended monitoring:

- VM availability, CPU, memory (where guest monitoring is configured), disk, and service health.
- Node.js process health, HTTP error rate, latency, and application logs.
- Application Gateway WAF detections and backend health.
- Azure Firewall traffic and threat logs.
- SQL availability, performance, connectivity, and auditing.
- Backup job status and recovery point availability.
- Azure Activity Log and relevant resource diagnostic logs.
- Security posture, vulnerability findings, and identity changes.

Create actionable alert rules and Azure Monitor action groups. Define ownership, severity, escalation, and response runbooks for each critical alert.

---

## 13. Implementation Sequence

1. **Foundation:** Confirm tenant, subscriptions, management groups, billing ownership, region policy, and Terraform state bootstrap.
2. **Address planning:** Validate CIDRs against corporate networks, VPN, ExpressRoute, other VNets, and the DR region.
3. **Connectivity:** Deploy hub VNet, Azure Firewall, Bastion, routing, DNS, and required gateway connectivity.
4. **Spokes:** Deploy Dev and QA VNets, subnets, NSGs, route tables, and hub peering.
5. **Governance and identity:** Apply policy assignments, RBAC, managed identities, workload federation, budgets, and logging.
6. **Application entry:** Deploy Application Gateway WAF and configure HTTPS, certificates, and backend health probes.
7. **Compute:** Deploy private web/application VMs with hardened images, patching, monitoring, and managed identities.
8. **Data:** Select Azure SQL Database or SQL Managed Instance, configure private connectivity, authentication, auditing, and backup.
9. **Delivery pipeline:** Build and scan the application, publish versioned artifacts, deploy via Azure DevOps, and implement approvals and rollback.
10. **Resilience:** Configure backup and the approved DR architecture; test restores, failover, and failback.
11. **Operational readiness:** Validate alerts, runbooks, ownership, security posture, cost controls, and compliance evidence before sign-off.

---

## 14. Pre-Deployment Checklist

- [ ] Dev and QA subscriptions are available and access is approved.
- [ ] Management group placement and policy inheritance are agreed.
- [ ] CIDR ranges are checked for overlap.
- [ ] Primary and DR regions are approved.
- [ ] Azure service availability and quotas are verified.
- [ ] SQL service choice is confirmed.
- [ ] RPO/RTO and data classification are approved.
- [ ] Terraform remote state is secured and bootstrapped.
- [ ] Azure DevOps workload identity federation is configured and tested.
- [ ] RBAC follows least privilege.
- [ ] Firewall, NSGs, UDRs, and private DNS are tested.
- [ ] Public exposure is minimized and approved.
- [ ] Diagnostic settings and security monitoring are enabled.
- [ ] Backup policies and restore tests are in place.
- [ ] DR failover/failback procedures are documented and tested.
- [ ] Naming, tags, budgets, and policy compliance are validated.

---

## 15. References

- [Azure landing zone design guidance](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure landing zone resource organization](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org)
- [Azure hub-and-spoke network topology](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke)
- [Azure SQL private endpoints](https://learn.microsoft.com/en-us/azure/azure-sql/database/private-endpoint-overview)
- [Azure Backup documentation](https://learn.microsoft.com/en-us/azure/backup/backup-overview)
- [Azure Site Recovery documentation](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview)
- [Azure Policy documentation](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [AzureRM Terraform backend](https://developer.hashicorp.com/terraform/language/backend/azurerm)

---

**Document status:** Proposed architecture baseline — validate and approve before production deployment.

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

