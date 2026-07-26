# Day 1 — Hub-Spoke VNet Foundation

## Objective
Build a hub-spoke VNet topology in Azure UAE North that mirrors enterprise greenfield designs used in production environments.

## Resources Created

### Resource Group
- **Name:** `stewart-lab-RG`
- **Region:** UAE North

### Hub VNet — `hub-vnet`
| Property | Value |
|---|---|
| Address Space | `10.0.0.0/16` |
| Region | UAE North |

**Subnets:**
| Name | Range | Purpose |
|---|---|---|
| `AzureFirewallSubnet` | `10.0.1.0/24` | Required exact name for Azure Firewall |
| `AppGatewaySubnet` | `10.0.2.0/24` | Application Gateway |
| `AzureBastionSubnet` | `10.0.3.0/24` | Required exact name for Azure Bastion |

### Spoke 1 VNet — `spoke1-vnet`
| Property | Value |
|---|---|
| Address Space | `10.1.0.0/16` |
| Region | UAE North |

**Subnets:**
| Name | Range | Purpose |
|---|---|---|
| `web-subnet` | `10.1.1.0/24` | Web server VM |
| `mgmt-subnet` | `10.1.2.0/24` | Management traffic |

### Spoke 2 VNet — `spoke2-vnet`
| Property | Value |
|---|---|
| Address Space | `10.2.0.0/16` |
| Region | UAE North |

**Subnets:**
| Name | Range | Purpose |
|---|---|---|
| `app-subnet` | `10.2.1.0/24` | Application VM |
| `mgmt-subnet` | `10.2.2.0/24` | Management traffic |

### VNet Peering
| Peering | Direction | Forwarded Traffic | Gateway Transit |
|---|---|---|---|
| `hub-to-spoke1` / `spoke1-to-hub` | Bidirectional | ✅ Enabled | None (no gateway deployed) |
| `hub-to-spoke2` / `spoke2-to-hub` | Bidirectional | ✅ Enabled | None (no gateway deployed) |

## Key Learnings

**Gateway Transit requires an actual gateway**  
Enabling "Allow gateway transit" on VNet peering requires a VPN Gateway or Route Server to already exist in the hub VNet. Attempting to enable it without one throws an error. For this lab, hub-based traffic inspection is achieved via UDR (User Defined Routes) pointing to Azure Firewall instead — which is the more cost-effective pattern for labs.

**Subnet naming is mandatory for certain services**  
`AzureFirewallSubnet` and `AzureBastionSubnet` must be spelled exactly as shown. Azure will reject deployments of those services if the subnet name differs.

**Address space planning matters**  
Non-overlapping address spaces across all VNets is required for peering to work:
- Hub: `10.0.0.0/16`
- Spoke1: `10.1.0.0/16`
- Spoke2: `10.2.0.0/16`

## Verification Steps
- Navigate to `hub-vnet` → Peerings
- Confirm both peerings show status: **Connected**
- Confirm forwarded traffic is enabled on both

## Screenshots
See [../screenshots/day1/](../screenshots/day1/)
