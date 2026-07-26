# Day 2 — NSG + Nginx Web Server

## Objective
Deploy a publicly accessible Nginx web server in Spoke 1 with correctly scoped NSG rules. Validate end-to-end connectivity from the public internet.

## Resources Created

### Network Security Group — `web-nsg`
Associated with: `spoke1-vnet / web-subnet`

**Inbound Rules:**
| Priority | Name | Port | Protocol | Source | Action |
|---|---|---|---|---|---|
| 100 | Allow-HTTP | 80 | TCP | Any | Allow |
| 110 | Allow-HTTPS | 443 | TCP | Any | Allow |
| 120 | Allow-SSH | 22 | TCP | Any | Allow |

> **Note:** NSG was associated at both subnet level AND NIC level for full effectiveness. Subnet-level association is the recommended practice for shared rules across all VMs in a subnet.

### Virtual Machine — `web-server-01`
| Property | Value |
|---|---|
| Image | Ubuntu Server 24.04 LTS |
| Size | Standard_B1s |
| VNet | `spoke1-vnet` |
| Subnet | `web-subnet` (10.1.1.0/24) |
| Private IP | 10.1.1.4 (Dynamic) |
| Public IP | 20.174.160.32 (`web-server-pip`) |
| Auto-shutdown | Enabled — 11:00 PM PKT |

### Nginx Configuration
- Installed via `apt install nginx`
- Service enabled (survives reboots)
- Custom HTML page deployed at `/var/www/html/index.html`
- Listening on `0.0.0.0:80`

## Troubleshooting Encountered

### Issue: "Connection refused" on public IP
**Symptom:** Browser showed "failed to connect — connection refused"  
**Initial suspect:** NSG misconfiguration  
**Actual cause:** NSG was not associated at the NIC level, only subnet level

**Diagnostic steps used:**
1. Checked `systemctl status nginx` — service was running ✅
2. Checked `curl http://localhost` — returned correct HTML ✅
3. Checked `curl -4 icanhazip.com` — confirmed correct public IP ✅
4. Used Azure Portal Network Watcher > Effective Security Rules — identified NIC had no NSG
5. Associated `web-nsg` to NIC directly — resolved the portal warning

**Final root cause:** Company endpoint security policy on the Windows laptop was blocking outbound port 80. Webpage loaded correctly when tested from a mobile device on a different network.

### Key Diagnostic Commands Used
```bash
# Check Nginx status
sudo systemctl status nginx

# Test Nginx locally
curl http://localhost

# Confirm VM's outbound public IP
curl -4 icanhazip.com

# Check what is listening on which ports
sudo ss -tlnp | grep nginx

# Check local firewall status
sudo ufw status
```

## Key Learnings

**Connection Refused vs Request Timed Out**
| Error | Meaning | Where to investigate |
|---|---|---|
| `Connection refused` | Packet reached VM, port not open or service down | VM-level: check service, UFW |
| `Request timed out` | Packet never reached VM | NSG, Route Table, Peering |

**Azure ICMP is blocked by default**  
Ping (ICMP) does not work to Azure VMs by default unless you add an NSG rule allowing it. This is expected behavior — do not use ping as a connectivity test in Azure.

**Always test from multiple networks**  
Enterprise endpoint security, corporate proxies, and ISP-level filtering can all block specific ports. A webpage that fails from a company laptop may work fine from a mobile device. This is a common real-world troubleshooting scenario.

**Network Watcher > Effective Security Rules**  
This is the correct Azure tool to verify exactly what security rules are being applied to a VM's NIC at runtime — more reliable than reading individual NSG configs manually.

## Verification
- Nginx serving page at `http://20.174.160.32` ✅
- Confirmed from mobile browser on separate network ✅
- Auto-shutdown active ✅

## Screenshots
See [../screenshots/day2/](../screenshots/day2/)
