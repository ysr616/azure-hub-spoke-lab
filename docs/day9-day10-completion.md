# Day 9 & 10 — Lab Completion + AZ-104 Alignment

## Day 9 — Documentation + Architecture Review

### Final Resource Inventory

| Resource | Name | Type | Status |
|---|---|---|---|
| Resource Group | `stewart-lab-RG` | Resource Group | ✅ |
| Hub VNet | `hub-vnet` | Virtual Network | ✅ |
| Spoke 1 VNet | `spoke1-vnet` | Virtual Network | ✅ |
| Spoke 2 VNet | `spoke2-vnet` | Virtual Network | ✅ |
| NSG | `web-nsg` | Network Security Group | ✅ |
| NSG | `appgw-nsg` | Network Security Group | ✅ |
| Route Table | `spoke1-udr` | Route Table | ✅ |
| Route Table | `spoke2-udr` | Route Table | ✅ |
| VM | `web-server-02` | Virtual Machine | ✅ |
| Firewall | `hub-firewall` | Azure Firewall (Basic) | ✅ |
| App Gateway | `hub-appgw` | Application Gateway v2 | ✅ |
| Private DNS | `stewart-lab.internal` | Private DNS Zone | ✅ |
| Log Analytics | `hub-log-analytics` | Log Analytics Workspace | ✅ |
| Connection Monitor | `spoke-connectivity-monitor` | Network Watcher | ✅ |
| Alert Rule | `web-server-high-cpu` | Monitor Alert | ✅ |
| Action Group | `noc-alerts` | Action Group | ✅ |

### Subnets Built

| VNet | Subnet | Range | Purpose |
|---|---|---|---|
| hub-vnet | AzureFirewallSubnet | 10.0.1.0/24 | Azure Firewall data plane |
| hub-vnet | AppGatewaySubnet | 10.0.5.0/24 | Application Gateway |
| hub-vnet | AzureBastionSubnet | 10.0.3.0/24 | Azure Bastion |
| hub-vnet | AzureFirewallManagementSubnet | 10.0.4.0/24 | Firewall management NIC |
| spoke1-vnet | web-subnet | 10.1.1.0/24 | Web server VM |
| spoke1-vnet | mgmt-subnet | 10.1.2.0/24 | Management |
| spoke2-vnet | app-subnet | 10.2.1.0/24 | Application tier |
| spoke2-vnet | mgmt-subnet | 10.2.2.0/24 | Management |

### Complete Traffic Flow

```
[Public Internet]
       │
       ▼
[hub-appgw] — AppGatewaySubnet (10.0.5.0/24)
  appgw-pip: 20.233.190.55
       │
       │ backend pool → 10.1.x.x
       ▼
[hub-firewall] — AzureFirewallSubnet (10.0.1.0/24)
  Private IP: 10.0.1.4
  UDR forces ALL spoke traffic here
       │
       │ network + application rules applied
       ▼
[web-server-02] — web-subnet (10.1.1.0/24)
  Private IP only — no public IP
  Nginx serving portfolio page
  Resolves as: web.stewart-lab.internal

[Admin Access]
       │
       ▼
[hub-bastion] — AzureBastionSubnet (10.0.3.0/24)
  Browser-based terminal only
  No SSH client required
```

---

## Day 10 — AZ-104 Exam Alignment

This lab covers the following AZ-104 exam domains directly:

### Implement and Manage Virtual Networking (25-30%)

| Exam Topic | Lab Coverage |
|---|---|
| Create and configure VNets | Hub + 2 spoke VNets with correct address spaces |
| Configure VNet peering | hub↔spoke1, hub↔spoke2 with forwarded traffic |
| Create and configure subnets | 8 subnets across 3 VNets with specific naming |
| Configure private DNS zones | `stewart-lab.internal` with VNet links + auto-registration |
| Create and configure NSGs | `web-nsg` and `appgw-nsg` with layered rules |
| Configure Azure Firewall | Basic SKU with network + application rule collections |
| Configure UDR | `spoke1-udr` and `spoke2-udr` forcing traffic to hub |

### Monitor and Maintain Azure Resources (10-15%)

| Exam Topic | Lab Coverage |
|---|---|
| Configure Azure Monitor | Log Analytics workspace + diagnostic settings |
| Create metric alerts | CPU alert with action group + email notification |
| Configure Network Watcher | Connection Monitor + Next Hop + Effective Security Rules |
| Analyze logs | KQL queries on firewall and App Gateway logs |

### Deploy and Manage Azure Compute Resources (20-25%)

| Exam Topic | Lab Coverage |
|---|---|
| Create and configure VMs | Ubuntu VM with correct size, VNet, subnet placement |
| Configure VM networking | Private IP only, NSG association, NIC configuration |
| Manage VM availability | Auto-shutdown, deallocate vs stop understanding |
| Configure VM extensions | az vm run-command for script execution |

### Implement and Manage Storage (15-20%)

| Exam Topic | Lab Coverage |
|---|---|
| Not directly covered | Recommend adding Azure Storage account in future lab |

### Manage Azure Identities and Governance (15-20%)

| Exam Topic | Lab Coverage |
|---|---|
| Not directly covered | Recommend adding RBAC roles in future lab |

---

## Key Troubleshooting Scenarios (Interview Ready)

### Scenario 1 — Connection Refused vs Timeout
- **Connection refused:** Packet reached VM, service not listening
- **Request timeout:** Packet never arrived — NSG, UDR, or peering issue
- **502 Bad Gateway:** App Gateway working, backend VM stopped
- **504 Gateway Timeout:** Backend overloaded or hanging

### Scenario 2 — NSG Layering
- NSG at subnet level = applies to all VMs in subnet
- NSG at NIC level = applies to specific VM only
- Both levels evaluated — most restrictive wins
- Network Watcher > Effective Security Rules = source of truth

### Scenario 3 — UDR + Firewall = Forced Inspection
- UDR alone: traffic directed to non-existent hop, dropped
- Firewall alone: traffic bypasses via system routes
- UDR + Firewall together: forced inspection of all spoke traffic

### Scenario 4 — App Gateway + UDR Conflict
- AppGatewaySubnet must have NO route table
- App Gateway v2 needs direct internet for management plane
- Adding UDR to AppGatewaySubnet breaks health probes

### Scenario 5 — Public IP Quota
- Free subscription default: 3 public IPs
- Basic Firewall consumes 2 PIPs (data + management)
- Bastion consumes 1 PIP
- App Gateway consumes 1 PIP
- Plan PIP usage: total minimum 4, exceeds free quota

### Scenario 6 — Private DNS Resolution Scope
- Private DNS only resolves from linked VNets
- Azure magic DNS IP: 168.63.129.16
- Auto-registration creates VM hostname records automatically
- External users must use public IP/FQDN, not private DNS names

---

## Real-World Connections — Stewart Title Architecture

This lab was intentionally designed to mirror production patterns:

| Lab Component | Production Equivalent |
|---|---|
| hub-vnet | Azure vWAN hub |
| Azure Firewall | Palo Alto inspection layer |
| Application Gateway | App GW + WAF for inbound |
| ExpressRoute (not built) | On-prem to Azure private connectivity |
| Private DNS | Internal service discovery |
| UDR | Forced tunneling to inspection |
| Bastion | Zero-trust VM access (no jump box) |

---

## Portfolio Value

This lab demonstrates:
- **Hub-spoke design** — industry standard Azure network topology
- **Defense in depth** — NSG + Firewall + App Gateway layered security
- **Zero-trust access** — no public IPs on workload VMs
- **Observability** — monitoring, logging, alerting, synthetic testing
- **Troubleshooting methodology** — 15+ real errors diagnosed and resolved
- **Documentation discipline** — GitHub repo with daily notes

**GitHub:** https://github.com/ysr616/azure-hub-spoke-lab
