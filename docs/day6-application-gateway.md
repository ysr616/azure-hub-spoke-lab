# Day 6 — Application Gateway (Inbound Traffic Management)

## Objective
Deploy Azure Application Gateway v2 in the hub VNet to manage all inbound web traffic, routing requests to the Nginx web server via private IP — with no public IP on the VM.

## Resources Created

### Application Gateway — `hub-appgw`
| Property | Value |
|---|---|
| Tier | Standard V2 |
| Virtual network | `hub-vnet` |
| Subnet | `AppGatewaySubnet` (10.0.5.0/24) |
| Frontend public IP | `appgw-pip` (20.233.190.55) |
| Backend pool | `web-backend-pool` → `10.1.1.4` |
| Routing rule | `http-routing-rule` (priority 100) |
| Listener | `http-listener` — HTTP port 80 |
| Backend setting | `http-backend-setting` — HTTP port 80 |

### Additional Subnet Added to hub-vnet
| Name | Range | Purpose |
|---|---|---|
| `AppGatewaySubnet` | `10.0.5.0/24` | Required for App Gateway — NO UDR attached |

### NSG — `appgw-nsg`
Associated to `AppGatewaySubnet`

| Priority | Name | Port | Source | Action |
|---|---|---|---|---|
| 100 | `Allow-HTTP-Internet` | 80 | Any | Allow |
| 110 | `Allow-GatewayManager` | 65200-65535 | GatewayManager | Allow |
| 120 | `Allow-AzureLoadBalancer` | * | AzureLoadBalancer | Allow |

## Troubleshooting Encountered

### Issue 1 — Missing AppGatewaySubnet
**Error:** AppGatewaySubnet not found in hub-vnet dropdown  
**Cause:** Subnet was not pre-created on Day 1  
**Fix:** Added `AppGatewaySubnet` (10.0.5.0/24) to hub-vnet before deployment  
**Lesson:** Plan all hub subnets upfront when designing the VNet

### Issue 2 — Public IP Quota Exceeded
**Error:** `ResourceCountExceedsLimitDueToTemplate` — subscription quota of 3 public IPs reached  
**Existing PIPs:** `firewall-pip`, `firewall-mgmt-pip`, `bastion-pip`  
**Fix:** Deleted Azure Bastion and its PIP to free one quota slot for `appgw-pip`  
**Lesson:** Free tier subscriptions have a Public IP quota of 3. Plan PIP usage carefully across Bastion, Firewall, and App Gateway.

### Issue 3 — Mobile browser timeout on App Gateway IP
**Symptom:** `curl` from Cloud Shell returned `HTTP/1.1 200 OK` but mobile browser timed out  
**Diagnosis:**
- `AppGatewaySubnet` confirmed no UDR ✅
- `appgw-nsg` correctly associated ✅
- Backend health showing **Healthy — 200 status code** ✅
- Cloud Shell curl confirmed full end-to-end working ✅

**Root cause:** Mobile carrier/ISP blocking outbound port 80 — same pattern as Day 2 Stewart laptop issue  
**Lesson:** Always test from multiple networks. Cloud Shell (running inside Azure) is the definitive test for Azure-to-Azure connectivity.

## Verification

### Backend Health
```
Server: 10.1.1.4 (web-backend-pool)
Status: Healthy
Port: 80 (http-backend-setting)
Protocol: HTTP
Details: Success. Received 200 status code
```

### Cloud Shell curl test
```bash
curl -I http://20.233.190.55
# HTTP/1.1 200 OK
# Server: nginx/1.24.0 (Ubuntu)
# Content-Length: 372
# Connection: keep-alive
```

### Route table verification
```bash
az network route-table list --resource-group stewart-lab-RG \
  --query "[].{Name:name, Subnet:subnets[0].id}" -o table
# Result: Only spoke1-udr and spoke2-udr
# AppGatewaySubnet has NO route table ✅
```

## Traffic Flow After Day 6

```
Internet
    ↓
appgw-pip (20.233.190.55) — public entry point
    ↓
hub-appgw (AppGatewaySubnet 10.0.5.0/24)
    ↓ backend pool routing
web-server-01 (10.1.1.4) — private IP only, no public IP
    ↓
Nginx → HTTP 200 OK
```

## Key Learnings

**App Gateway subnet requirements**
- Must have its own dedicated subnet — cannot share with other resources
- Must NOT have a UDR/route table — App GW v2 requires direct internet access for management
- Requires NSG with GatewayManager (65200-65535) rule for Azure platform health probes

**App Gateway vs Load Balancer**
| Feature | App Gateway | Azure Load Balancer |
|---|---|---|
| Layer | L7 (HTTP/HTTPS) | L4 (TCP/UDP) |
| SSL termination | Yes | No |
| URL routing | Yes | No |
| WAF support | Yes (WAF SKU) | No |
| Cost | Higher | Lower |

**Public IP quota planning**
In free/trial subscriptions, plan PIP allocation carefully:
- Bastion: 1 PIP
- Firewall Basic: 2 PIPs (data + management)
- App Gateway: 1 PIP
- Total: 4 PIPs minimum — exceeds default quota of 3

**Workaround:** Delete Bastion PIP when not in active use, redeploy when needed. Bastion redeployment takes ~5 minutes.

## Cost Management
App Gateway Standard v2: ~$0.25/hr  
**Stop after each session:** Portal → `hub-appgw` → **Stop**

## Screenshots
See [../screenshots/day6/](../screenshots/day6/)
