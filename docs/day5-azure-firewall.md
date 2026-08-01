# Day 5 — Azure Firewall (Hub Inspection Layer)

## Objective
Deploy Azure Firewall Basic in the hub VNet, activate the UDR routing from Day 4, and verify all spoke traffic is inspected before reaching the internet.

## Resources Created

### Azure Firewall — `hub-firewall`
| Property | Value |
|---|---|
| SKU | Basic |
| Virtual network | `hub-vnet` |
| Subnet | `AzureFirewallSubnet` (10.0.1.0/24) |
| Private IP | `10.0.1.4` |
| Public IP | `firewall-pip` |
| Management subnet | `AzureFirewallManagementSubnet` (10.0.4.0/24) |
| Management public IP | `firewall-mgmt-pip` |
| Provisioning state | Succeeded |

### Additional Subnet Added to hub-vnet
| Name | Range | Purpose |
|---|---|---|
| `AzureFirewallManagementSubnet` | `10.0.4.0/24` | Required by Basic SKU for management NIC |

### Firewall Rules Created

**Network Rule Collection — `allow-spoke-traffic` (Priority 100, Action: Allow)**
| Rule | Source | Destination | Protocol | Ports |
|---|---|---|---|---|
| `spoke1-to-spoke2` | `10.1.0.0/16` | `10.2.0.0/16` | Any | * |
| `spokes-to-internet` | `10.1.0.0/16, 10.2.0.0/16` | `*` | Any | * |

**Application Rule Collection — `allow-web-outbound` (Priority 100, Action: Allow)**
| Rule | Source | Protocol:Port | Target FQDNs |
|---|---|---|---|
| `allow-http-https` | `10.1.0.0/16, 10.2.0.0/16` | http:80, https:443 | `*` |

## Troubleshooting Encountered

### Firewall Validation Failed — Missing Management Subnet
**Error:** *"Force Tunneling requires this virtual network have a subnet named AzureFirewallManagementSubnet"*

**Cause:** Azure Firewall Basic SKU requires a dedicated management subnet for its management NIC. This subnet was not created on Day 1 because the requirement is specific to the Basic SKU with Management NIC enabled.

**Fix:** Added `AzureFirewallManagementSubnet` (10.0.4.0/24) to `hub-vnet` before retrying firewall deployment.

**Lesson:** When planning hub VNet subnets for Azure Firewall Basic, always pre-create both:
- `AzureFirewallSubnet` — for data plane traffic
- `AzureFirewallManagementSubnet` — for management plane traffic (Basic SKU requirement)

## Verification

### Network Watcher — Next Hop (Before vs After)
| Test | Day 4 Result | Day 5 Result |
|---|---|---|
| Source: 10.1.1.4 → Dest: 8.8.8.8 | Next hop: **None** | Next hop: **VirtualAppliance** |
| Next hop IP | 10.0.1.4 | 10.0.1.4 |

The UDRs from Day 4 activated instantly when the firewall was deployed into `10.0.1.4`.

### Internet Traffic Test (via Bastion terminal)
```bash
curl -l http://google.com
# Result: 301 Moved — HTML response from Google returned successfully
# Traffic path: web-server-01 → UDR → hub-firewall → Google → response back
```

## Traffic Flow After Day 5

```
Internet
    │
    ▼
hub-firewall (10.0.1.4)  ← ALL outbound traffic inspected here
    │                        Application rules applied (allow http/https)
    │
    ├── UDR: 0.0.0.0/0 → 10.0.1.4 (spoke1-udr)
    │   └── web-server-01 (10.1.1.4)
    │
    └── UDR: 0.0.0.0/0 → 10.0.1.4 (spoke2-udr)
        └── app-subnet (10.2.1.0/24)

Admin access: Azure Bastion → hub-vnet → private IP only
```

## Key Learnings

**Default-deny firewall behavior**
Azure Firewall blocks all traffic by default. Without explicit allow rules, even DNS resolution fails. Network rules handle IP-based traffic, application rules handle FQDN-based traffic (with DNS proxy).

**UDR + Firewall = Forced Inspection**
The combination of Day 4 UDRs and Day 5 Firewall creates forced tunneling — all traffic must pass through the inspection layer regardless of destination. Neither alone is sufficient:
- UDR without Firewall: traffic directed to nothing, drops
- Firewall without UDR: traffic bypasses firewall via system routes

**Network vs Application Rules**
| Rule Type | Use For | Example |
|---|---|---|
| Network rules | IP/port based allow/deny | Spoke-to-spoke communication |
| Application rules | FQDN based with HTTP/S inspection | Allow *.microsoft.com outbound |
| DNAT rules | Inbound traffic translation | Inbound to internal VM (Day 6 App GW replaces this) |

**Basic SKU limitations (vs Standard)**
- No IDPS (Intrusion Detection)
- No TLS inspection
- No URL filtering beyond FQDN
- Sufficient for lab demonstration of hub inspection concept

## Cost Management
Azure Firewall Basic: ~$0.30/hr  
**Always stop after each session:**  
Portal → `hub-firewall` → **Stop**

**Day start checklist (updated):**
1. Start `hub-firewall` first (takes 3-4 min)
2. Start `web-server-01`
3. Then proceed with lab work

## Screenshots
See [../screenshots/day5/](../screenshots/day5/)
