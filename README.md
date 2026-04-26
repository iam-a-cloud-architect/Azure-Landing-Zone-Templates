# 🏗️ Azure CAF Landing Zone — Terraform

> Industry-standard, CAF-aligned Azure Landing Zone with Management Groups, Policy Guardrails, and Hub-Spoke Networking.
> 
> Author: **Saravanaselvan Narayanamoorthy** — Cloud Presales Architect

---

## 📐 Architecture

```
Tenant Root Management Group
└── Contoso (Root Org MG)
    ├── Platform
    │   ├── Connectivity   ← Hub VNet, Firewall, Bastion, VPN GW
    │   ├── Management     ← Log Analytics, Automation Account
    │   └── Identity       ← AD DS, AAD Connect
    ├── Landing Zones
    │   ├── Corp           ← Internal workloads (spokes peered to hub)
    │   └── Online         ← Internet-facing workloads
    ├── Sandboxes          ← Dev/test (relaxed policies)
    └── Decommissioned     ← Subscriptions pending removal
```

```
Connectivity Subscription
┌────────────────────────────────────────────────┐
│  Hub VNet (10.0.0.0/16)                        │
│  ┌──────────────┐  ┌────────────────────────┐  │
│  │GatewaySubnet │  │AzureFirewallSubnet     │  │
│  │10.0.0.0/27   │  │10.0.1.0/26  [FW PIP]   │  │
│  └──────┬───────┘  └──────────┬─────────────┘  │
│         │ VPN GW             │ Azure Firewall  │
│  ┌──────┴─────┐  ┌──────────┴─────────────┐    │
│  │AzureBastion│  │snet-mgmt               │    │
│  │10.0.3.0/26 │  │10.0.2.0/27             │    │
│  └────────────┘  └────────────────────────┘    │
└──────────┬───────────────────┬─────────────────┘
           │  VNet Peering     │  VNet Peering
    ┌──────┴──────┐     ┌──────┴──────┐
    │  Corp Spoke │     │  DMZ Spoke  │
    │  10.1.0.0/16│     │  10.2.0.0/16│
    └─────────────┘     └─────────────┘
```

---

## 🔒 Policy Guardrails Deployed

| Policy | Scope | Effect |
|---|---|---|
| Allowed Locations | Root MG | Deny |
| Require Tags (CostCenter, Owner, Env, BU, ManagedBy) | Root MG | Deny |
| Microsoft Cloud Security Benchmark (MCSB) | Root MG | Audit |
| Deploy Diagnostic Settings → Log Analytics | Platform MG | DeployIfNotExists |
| Deny Public IP on NICs | Landing Zones MG | Deny |
| Require HTTPS on Storage Accounts | Landing Zones MG | Deny |

---

## 📁 Repository Structure

```
caf-landing-zone/
├── main.tf                          # Root module: wires all child modules
├── variables.tf                     # All input variables with validation
├── outputs.tf                       # Key output values
├── terraform.tfvars.example         # Example values (copy → terraform.tfvars)
├── README.md
└── modules/
    ├── management-group/
    │   ├── main.tf                  # Full MG hierarchy
    │   ├── variables.tf
    │   └── outputs.tf
    ├── policy/
    │   ├── main.tf                  # CAF policy assignments
    │   └── variables.tf
    └── hub-spoke/
        ├── main.tf                  # Hub VNet, Firewall, Bastion, Spokes, DNS
        └── variables_outputs.tf
```

---

## 🚀 Quick Start

### Prerequisites

| Tool | Minimum Version |
|---|---|
| Terraform | >= 1.5.0 |
| Azure CLI | >= 2.50 |
| Contributor + User Access Admin | At Tenant Root MG scope |

### Step 1 — Authenticate

```bash
az login
az account set --subscription "<connectivity-subscription-id>"
```

### Step 2 — Elevate permissions (first-time only)

```bash
# Elevate your account to manage Management Groups
az rest --method post \
  --url "/providers/Microsoft.Authorization/elevateAccess?api-version=2016-07-01"
```

### Step 3 — Configure variables

```bash
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your subscription IDs, org prefix, etc.
```

### Step 4 — Configure remote state backend

Edit the `backend "azurerm"` block in `main.tf`:

```bash
# Create state storage first
az group create -n rg-tfstate-prod -l eastus
az storage account create -n sttfstateprod001 -g rg-tfstate-prod --sku Standard_LRS
az storage container create -n tfstate --account-name sttfstateprod001
```

### Step 5 — Deploy

```bash
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

---

## 🧹 Destroy (use with caution)

```bash
# Remove policy assignments first to avoid dependency locks
terraform destroy -target=module.policies
terraform destroy
```

---

## 💡 Customisation Guide

| Goal | What to change |
|---|---|
| Add a new spoke VNet | Add entry to `spoke_vnets` map in `terraform.tfvars` |
| Add a new policy | Add resource in `modules/policy/main.tf` |
| Enable VPN Gateway | Set `enable_vpn_gateway = true` |
| Add Private DNS zone | Append to `local.private_dns_zones` in hub-spoke module |
| Change Firewall mode to Deny | Set `mode = "Deny"` in `azurerm_firewall_policy.hub` intrusion_detection |

---

## 📊 Cost Estimate (East US, monthly)

| Resource | SKU | Est. Cost |
|---|---|---|
| Azure Firewall Premium | Fixed + data | ~$1,740 |
| VPN Gateway (if enabled) | VpnGw2AZ | ~$365 |
| Azure Bastion Standard | Fixed + hours | ~$200 |
| Log Analytics Workspace | PerGB2018, 90d | Variable |
| DDoS Protection (if enabled) | Standard | ~$2,944 |

> FinOps tip: Use Reservations for Firewall (1yr = ~33% saving). Disable DDoS in non-prod.

---

## 📚 References

- [Microsoft Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)
- [Azure Landing Zones (Bicep/Terraform)](https://github.com/Azure/ALZ-Bicep)
- [CAF Terraform Landing Zones](https://github.com/Azure/caf-terraform-landingzones)
- [Azure Architecture Center — Hub-Spoke](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke)

---

*Built with ❤️ by Saravanaselvan — feedback welcome via Issues or LinkedIn.*
