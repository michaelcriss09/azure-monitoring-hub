# Azure Monitoring Infrastructure

Terraform-deployed Azure infrastructure implementing a hub-spoke architecture with monitoring, high availability, and secure access via Bastion.

---

## Architecture

![Architecture](https://i.postimg.cc/Z5PDCfJk/Captura-de-pantalla-2026-03-25-091557.png)

### VNets

| VNet | CIDR | Description |
|------|------|-------------|
| workload_vnet | 10.0.0.0/16 | VMs and workloads |
| security_hub_vnet | 10.1.0.0/16 | Bastion and security |

### Subnets

| Subnet | CIDR | VNet |
|--------|------|------|
| workload_logic_subnet | 10.0.1.0/24 | workload_vnet |
| AzureBastionSubnet | 10.1.1.0/24 | security_hub_vnet |

---

## Deployed Resources

- 2 VNets with bidirectional VNet Peering
- 2 Subnets
- 2 Linux VMs Ubuntu 22.04 (Standard_D2s_v3)
- Internal Load Balancer with backend pool, health probe, and LB rule
- Azure Bastion (Standard SKU) with Native Client enabled
- NAT Gateway for outbound internet access
- NSG with security rules
- Log Analytics Workspace
- Azure Monitor Agent (AMA) on each VM
- Data Collection Rules (DCR) for Syslog and Performance
- Metric Alert for CPU > 80%
- Action Group with email notification

---

## NSG — Security Rules

### Inbound

| Priority | Name | Port | Source | Action |
|----------|------|------|--------|--------|
| 100 | allow-lb-health-probe | Any | AzureLoadBalancer | Allow |
| 110 | allow-bastion-ssh | 22 | VirtualNetwork | Allow |
| 120 | allow-intra-subnet-ssh | 22 | 10.0.1.0/24 | Allow |
| 4096 | deny-all-inbound | Any | Any | Deny |

### Outbound

| Priority | Name | Port | Destination | Action |
|----------|------|------|-------------|--------|
| 100 | allow-intra-subnet-outbound | 22 | 10.0.1.0/24 | Allow |
| 110 | allow-azure-monitor-outbound | 443 | AzureMonitor | Allow |
| 120 | allow-http-outbound | 80 | Internet | Allow |
| 130 | allow-https-outbound | 443 | Internet | Allow |
| 4096 | deny-all-outbound | Any | Any | Deny |

---

## Prerequisites

- Terraform >= 1.3
- Azure CLI configured (`az login`)
- Active Azure subscription

---

## Required Variables

`terraform.tfvars`

| Variable | Description |
|----------|-------------|
| `admin_password` | VM administrator password |
| `email_receiver_email` | Email address for monitoring alerts |

---

## Usage

```bash
# Initialize
terraform init

# Review the plan
terraform plan

# Deploy
terraform apply

# Destroy
terraform destroy
```

---

## Connecting to VMs

### From CLI

```bash
az network bastion ssh \
  --name <bastion_name> \
  --resource-group monitor-rg \
  --target-resource-id $(az vm show -g monitor-rg -n vm1 --query id -o tsv) \
  --auth-type password \
  --username vm1-admin
```

### From the Portal

Portal → `monitor-rg` → VM → Connect → Bastion

---

## Validation

### Verify Peering

```bash
az network vnet peering show \
  -g monitor-rg \
  --vnet-name workload_vnet \
  --name workload_vnet-peer \
  --query "{state:peeringState, access:allowVirtualNetworkAccess}" \
  -o table
```

### Verify VMs in Backend Pool

```bash
az network lb address-pool show \
  -g monitor-rg \
  --lb-name <ilb_name> \
  -n <backend_pool_name>
```

### Verify Metrics in Log Analytics

```kusto
Perf
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| where Computer in ("vm1-vm", "vm2-vm")
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 1m)
| order by TimeGenerated desc
| take 20
```

### Test CPU Alert

```bash
# From the VM via Bastion
stress --cpu 2 --timeout 300
```

---

## Notes

- `AzureBastionSubnet` does not require an NSG — Azure manages it internally.
- The NAT Gateway allows outbound internet access from private VMs without exposing them with a public IP.
- DCRs collect Syslog and Performance Counters every 60 seconds.
- The CPU alert evaluates every minute with a 5-minute window.
