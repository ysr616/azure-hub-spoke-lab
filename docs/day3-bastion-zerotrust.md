# Day 3 — Azure Bastion + Zero-Trust Access

## Objective
Deploy Azure Bastion in the hub VNet, remove the public IP from the web server VM, and validate secure access through the hub — with zero direct internet exposure on the VM.

## Resources Created

### Azure Bastion — `hub-bastion`
| Property | Value |
|---|---|
| Tier | Basic |
| VNet | `hub-vnet` |
| Subnet | `AzureBastionSubnet` (10.0.3.0/24) |
| Public IP | `bastion-pip` |
| Region | UAE North |

> Bastion is the only resource with a public IP in this design. All VMs sit behind it with no direct internet exposure.

## Configuration Changes

### Public IP Removed from `web-server-01`
- Navigated to NIC → IP configurations → ipconfig1
- Unchecked "Associate public IP address"
- VM now has only private IP: `10.1.1.4`
- Public IP `20.174.160.32` disassociated and released

### VM Access Method Changed
| Before Day 3 | After Day 3 |
|---|---|
| SSH via public IP (port 22 open to internet) | Bastion browser terminal only |
| Public IP: 20.174.160.32 | No public IP |
| NSG Allow-SSH from Any | SSH no longer reachable from internet |

## Verification Steps

### 1. Public access confirmed blocked
Tested `http://20.174.160.32` from mobile browser after public IP removal — connection timed out (not refused). Full timeout confirms no route to VM from internet.

### 2. Bastion access confirmed working
- Navigated to `web-server-01` → Connect → Bastion
- Authenticated with username/password
- Browser-based terminal opened successfully on company laptop
- No SSH client required, no public IP required

### 3. Nginx still running inside VM
```bash
sudo systemctl status nginx
# Output: active (running)

curl http://localhost
# Output: Custom HTML page returned correctly

ip addr show
# Output: Only 10.1.1.4 — no public IP on interface
```

## Troubleshooting Encountered

### Issue: Bastion connection timeout on first attempt
**Symptom:** Bastion session showed "connection timeout" immediately after clicking Connect  
**Initial thought:** Bastion misconfiguration or NSG blocking  
**Self-diagnosed:** Checked VM status independently on mobile Azure app — found VM was in **deallocated state** from overnight auto-shutdown  
**Fix:** Started the VM from mobile Azure portal, returned to Bastion session on company laptop — connected successfully without re-entering credentials  

**Key insight:** Bastion connects to the VM's private IP via the hub VNet. If the VM is stopped/deallocated, the private IP is released and Bastion has nothing to connect to — same as SSH to a powered-off machine. Always verify VM is in **Running** state before initiating Bastion session.

> **Standard Day Start Checklist (learned from this):**
> 1. Start the VM first
> 2. Wait for status to show "Running"
> 3. Then initiate Bastion session

## Architecture After Day 3

```
Internet
    │
    │  ← Port 80/443 NO LONGER reachable directly
    │
    ▼
[Azure Bastion - hub-vnet]  ← Only entry point for admin access
    │
    │ Private VNet path only
    ▼
[web-server-01 - 10.1.1.4]  ← No public IP, Nginx still running
```

## Key Learnings

**Zero-trust access pattern**
Removing the public IP from a VM doesn't affect its ability to serve traffic internally or through a load balancer/App Gateway in front of it. It only removes direct internet reachability. The VM's private IP remains stable.

**Bastion vs Jump Box**
Azure Bastion is the managed alternative to a traditional jump box/bastion host. The key advantages:
- No VM to maintain or patch
- No public IP on the jump box VM itself (Bastion's public IP is managed by Azure)
- Browser-based — works from any device including mobile, no SSH client needed
- Session stays alive independently — starting VM on mobile and connecting on laptop in the same session is valid

**Auto-shutdown awareness**
Auto-shutdown deallocates the VM — it releases compute AND the dynamic private IP. Always verify Running state before dependent operations (Bastion, App Gateway health probes, monitoring agents).

## Cost Note
Azure Bastion Basic: ~$0.19/hr  
**Stop Bastion after each session** to preserve credits:  
Portal → `hub-bastion` → Stop

## Screenshots
See [../screenshots/day3/](../screenshots/day3/)
