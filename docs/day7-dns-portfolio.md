# Day 7 — Azure Private DNS + Portfolio Page Deployment

## Objective
Deploy a private DNS zone for internal name resolution across all VNets, verify DNS-based connectivity, and replace the placeholder webpage with a live professional portfolio page served through the full enterprise stack.

## Resources Created

### Private DNS Zone — `stewart-lab.internal`
| Property | Value |
|---|---|
| Location | Global |
| Resource group | `stewart-lab-RG` |
| Recordsets | 3 |
| Virtual Network Links | 3 |
| Links with Auto-registration | 2 (spoke1, spoke2) |

### Virtual Network Links
| Link Name | VNet | Auto-registration |
|---|---|---|
| `hub-vnet-link` | `hub-vnet` | No |
| `spoke1-vnet-link` | `spoke1-vnet` | Yes ← auto-registers VM hostnames |
| `spoke2-vnet-link` | `spoke2-vnet` | Yes ← auto-registers VM hostnames |

### DNS A Record
| Name | Type | TTL | Value |
|---|---|---|---|
| `web` | A | 300 | `10.1.1.4` |

Resolves as: `web.stewart-lab.internal` → `10.1.1.4`

## Portfolio Page Deployment

Replaced default Nginx page with a professional portfolio page deployed via `az vm run-command`:

```bash
az vm run-command invoke \
  --resource-group stewart-lab-RG \
  --name web-server-01 \
  --command-id RunShellScript \
  --scripts "cat > /var/www/html/index.html << 'HTMLEOF'
$(cat /home/yasir/index.html)
HTMLEOF
sudo systemctl restart nginx"
```

**Result:** `ProvisioningState: succeeded`

Portfolio page features:
- Dark NOC-themed design with pulsing status indicator
- 15+ years experience timeline in network topology style
- Skills panel showing monitoring tools (LogicMonitor, SolarWinds, ThousandEyes)
- Contact information and LinkedIn link
- Fully responsive mobile layout

## Verification

### DNS Resolution Test (from Bastion terminal)
```bash
nslookup web.stewart-lab.internal
# Server: 127.0.0.53
# Name: web.stewart-lab.internal
# Address: 10.1.1.4  ✅

curl http://web.stewart-lab.internal
# Returns full portfolio HTML page ✅
```

### Portfolio Page Test (from Cloud Shell)
```bash
curl -s http://20.233.190.55 | grep "<title>"
# <title>Yasir Munir — NOC &amp; Network Engineer</title>  ✅
```

## Complete Traffic Flow After Day 7

```
Internet
    ↓
appgw-pip (20.233.190.55)
    ↓
hub-appgw (AppGatewaySubnet)
    ↓ backend pool → 10.1.1.4
hub-firewall (10.0.1.4) ← inspects all spoke traffic
    ↓ UDR forces traffic here
web-server-01 (10.1.1.4)
    ↓ resolves privately as
web.stewart-lab.internal
    ↓ accessed securely via
Azure Bastion (hub-vnet) ← only admin path

DNS: stewart-lab.internal private zone → all 3 VNets linked
```

## Key Learnings

**Private DNS vs Public DNS**
| Feature | Private DNS | Public DNS |
|---|---|---|
| Resolvable from | Linked VNets only | Anywhere |
| Cost | Very low | Low |
| Use case | Internal service discovery | Public-facing endpoints |
| Auto-registration | Yes (VM hostnames) | No |

**Auto-registration behavior**
When enabled on a VNet link, Azure automatically creates A records for every VM in that VNet using the VM's hostname. This means `web-server-01.stewart-lab.internal` is auto-registered alongside the manual `web` record — no manual maintenance needed as VMs are added.

**Azure DNS resolver IP**
Azure VMs always use `168.63.129.16` as their DNS server (Azure's magic IP). When a private zone is linked to the VNet, this resolver automatically returns private zone records without any VM-level DNS configuration.

**az vm run-command**
Powerful tool for pushing files and running scripts on VMs without SSH/Bastion access. Useful for:
- Deploying configuration files
- Running maintenance scripts
- Pushing web content
Runs as root — no sudo needed inside the script.

## Screenshots
See [../screenshots/day7/](../screenshots/day7/)
