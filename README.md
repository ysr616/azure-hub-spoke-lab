# Azure Hub-Spoke Network Lab
### Enterprise Hybrid Networking Portfolio — UAE North Region

**Author:** Yasir Munir | NOC Engineer II | Stewart Title  
**Skills Demonstrated:** Azure Networking | SD-WAN | Meraki | Hub-Spoke Design | Zero-Trust Access  
**Lab Duration:** 10 Days | **Region:** UAE North | **Status:** 🟡 In Progress

[![LinkedIn](https://img.shields.io/badge/LinkedIn-yasirmunir-blue)](https://www.linkedin.com/in/yasirmunir)

---

## 📋 Overview

This lab mirrors a real enterprise Azure greenfield deployment — modeled after production hybrid networking architectures used at Stewart Title. It demonstrates core Azure networking concepts aligned with the **AZ-104** exam and real-world enterprise requirements including:

- Hub-spoke VNet topology
- Centralized security inspection (Azure Firewall)
- Inbound traffic management (Application Gateway)
- Zero-trust VM access (Azure Bastion)
- DNS failover and monitoring

---

## 🏗️ Architecture

```
                    ┌─────────────────────────────────┐
                    │           Hub VNet               │
                    │         10.0.0.0/16              │
                    │                                  │
                    │  ┌──────────────────────────┐   │
                    │  │   Azure Firewall Basic   │   │
                    │  │    AzureFirewallSubnet    │   │
                    │  │      10.0.1.0/24         │   │
                    │  └──────────────────────────┘   │
                    │  ┌──────────────────────────┐   │
                    │  │   Application Gateway    │   │
                    │  │    AppGatewaySubnet       │   │
                    │  │      10.0.2.0/24         │   │
                    │  └──────────────────────────┘   │
                    │  ┌──────────────────────────┐   │
                    │  │     Azure Bastion        │   │
                    │  │  AzureBastionSubnet       │   │
                    │  │      10.0.3.0/24         │   │
                    │  └──────────────────────────┘   │
                    └──────────┬──────────────────────┘
                               │ VNet Peering
               ┌───────────────┴───────────────┐
               │                               │
  ┌────────────▼──────────┐      ┌─────────────▼─────────┐
  │    Spoke 1 VNet       │      │    Spoke 2 VNet        │
  │    10.1.0.0/16        │      │    10.2.0.0/16         │
  │                       │      │                        │
  │  web-subnet           │      │  app-subnet            │
  │  10.1.1.0/24          │      │  10.2.1.0/24           │
  │  [web-server-01]      │      │  [app-server-01]       │
  │  Nginx Web Server     │      │  TBD Day 5+            │
  │                       │      │                        │
  │  mgmt-subnet          │      │  mgmt-subnet           │
  │  10.1.2.0/24          │      │  10.2.2.0/24           │
  └───────────────────────┘      └────────────────────────┘
```

---

## 📦 Resources Deployed

| Resource | Name | Subnet / Location | Status |
|---|---|---|---|
| Resource Group | `stewart-lab-RG` | UAE North | ✅ Done |
| Hub VNet | `hub-vnet` | `10.0.0.0/16` | ✅ Done |
| Spoke 1 VNet | `spoke1-vnet` | `10.1.0.0/16` | ✅ Done |
| Spoke 2 VNet | `spoke2-vnet` | `10.2.0.0/16` | ✅ Done |
| VNet Peering | hub↔spoke1, hub↔spoke2 | — | ✅ Done |
| NSG | `web-nsg` | `web-subnet` | ✅ Done |
| Linux VM | `web-server-01` | `10.1.1.0/24` | ✅ Done |
| Nginx Web Server | — | `web-server-01` | ✅ Done |
| Azure Bastion | `hub-bastion` | `AzureBastionSubnet` | ✅ Done  |
| Route Tables (UDR) | `spoke-udr` | Both spokes | 🔜 Day 4 |
| Azure Firewall | `hub-firewall` | `AzureFirewallSubnet` | 🔜 Day 5 |
| Application Gateway | `hub-appgw` | `AppGatewaySubnet` | 🔜 Day 6 |
| Azure DNS | `lab-private-dns` | Private Zone | 🔜 Day 7 |
| Azure Monitor | `lab-monitor` | Log Analytics | 🔜 Day 8 |

---

## 📅 10-Day Build Plan

| Day | Topic | Status |
|---|---|---|
| Day 1 | Hub-Spoke VNet foundation + VNet Peering | ✅ Complete |
| Day 2 | NSG + Linux VM + Nginx Web Server | ✅ Complete |
| Day 3 | Azure Bastion + Zero-Trust Access (remove public IPs) | ✅ Complete |
| Day 4 | Route Tables (UDR) + Forced Tunneling | 🔜 Pending |
| Day 5 | Azure Firewall Basic — spoke-to-spoke inspection | 🔜 Pending |
| Day 6 | Application Gateway — inbound traffic management | 🔜 Pending |
| Day 7 | Azure DNS Private Zone + Traffic Manager | 🔜 Pending |
| Day 8 | Azure Monitor + Network Watcher + Alerts | 🔜 Pending |
| Day 9 | GitHub documentation + architecture diagrams | 🔜 Pending |
| Day 10 | AZ-104 alignment review + buffer | 🔜 Pending |

---

## 📖 Daily Lab Notes

### ✅ Day 1 — Hub-Spoke Foundation
**Goal:** Build the VNet topology that mirrors Stewart Title's Azure Greenfield design.

**What was built:**
- Resource Group `stewart-lab-RG` in UAE North
- Hub VNet `10.0.0.0/16` with 3 dedicated subnets (Firewall, AppGW, Bastion)
- Spoke 1 VNet `10.1.0.0/16` with web and mgmt subnets
- Spoke 2 VNet `10.2.0.0/16` with app and mgmt subnets
- VNet Peering hub↔spoke1 and hub↔spoke2 with forwarded traffic enabled

**Key learning:**  
Gateway transit requires an actual VPN Gateway or Route Server to exist — it cannot be enabled on empty VNets. UDR-based routing via Azure Firewall achieves the same hub inspection goal without a gateway, and is the correct pattern for this design.

**Screenshots:** [Day 1 folder](./screenshots/day1/)

---

### ✅ Day 2 — NSG + Web Server
**Goal:** Deploy a working Nginx web server with correctly scoped NSG rules.

**What was built:**
- NSG `web-nsg` with Allow-HTTP (80), Allow-HTTPS (443), Allow-SSH (22) rules
- NSG associated at **both subnet and NIC level** for full effectiveness
- VM `web-server-01` (Ubuntu 24.04 LTS, Standard_B1s) in `web-subnet`
- Nginx installed, enabled, serving custom HTML page
- Auto-shutdown configured to protect Azure credits

**Key learnings:**
- `Connection refused` = packet reaches VM but nothing is listening (Nginx issue) vs `Request timed out` = packet never arrives (NSG/routing issue)
- Azure blocks ICMP by default — ping is not a reliable reachability test
- Company endpoint security policies can block outbound port 80 — always test from multiple devices/networks
- Network Watcher > Effective Security Rules is the correct tool to verify what Azure is actually applying

**Webpage:** Publicly accessible at `http://20.174.160.32`

**Screenshots:** [Day 2 folder](./screenshots/day2/)

---


### ✅ Day 3 — Azure Bastion + Zero-Trust Access

**Goal:** Deploy Bastion in the hub, remove VM public IP, validate secure browser-based access with no internet exposure on the VM.

**What was built:**
- Azure Bastion Basic deployed in `AzureBastionSubnet` of `hub-vnet`
- Public IP removed from `web-server-01` — VM now private-only (`10.1.1.4`)
- Bastion browser terminal confirmed working from company laptop
- Nginx confirmed still running internally via `curl http://localhost`

**Key learnings:**
- Removing public IP from VM does not affect internal service operation — Nginx keeps running, only direct internet access is blocked
- Bastion connects via private VNet path through the hub — VM must be in **Running** state or connection times out
- Self-diagnosed VM deallocated state by checking Azure mobile app independently — started VM from mobile, Bastion session on laptop connected without re-entering credentials
- Always verify VM Running state before starting Bastion session (auto-shutdown awareness)

**Screenshots:** [Day 3 folder](./screenshots/day3/)

---
## 💡 Key Concepts Demonstrated

### NSG — Two Levels of Enforcement
Azure NSGs can be applied at two levels — subnet and NIC. Subnet-level protects all VMs in the subnet. NIC-level gives per-VM granular control. Best practice is subnet-level for broad rules, NIC-level only when a specific VM needs different treatment.

### VNet Peering — Forwarded Traffic Setting
Enabling "Allow forwarded traffic" on VNet peering is critical for hub-spoke designs. Without it, traffic that arrives at the hub from one spoke cannot be forwarded to another spoke — it gets silently dropped. This maps directly to the packet loss issue documented in Meraki vMX Azure deployments.

### Connection Refused vs Request Timed Out
| Error | Meaning | Where to look |
|---|---|---|
| Connection refused | Packet arrived, port not open | VM — check service running, UFW |
| Request timed out | Packet never arrived | NSG, Route Table, peering |
| Bad Gateway | Proxy/LB received but backend failed | App Gateway, Load Balancer |

---

## 💰 Cost Management

All VMs configured with **auto-shutdown** at 11 PM PKT.  
Bastion, Firewall, and App Gateway stopped manually after each session.

| Resource | Hourly Cost | Daily Est. (3hr session) |
|---|---|---|
| Azure Firewall Basic | ~$0.30/hr | ~$0.90 |
| Application Gateway v2 | ~$0.25/hr | ~$0.75 |
| Azure Bastion Basic | ~$0.19/hr | ~$0.57 |
| VM Standard_B1s | ~$0.01/hr | ~$0.08 |

---

## 🔗 Related Projects

- [Original Azure Networking Lab](https://github.com/yasirmunir/azure-networking-labs) — foundational VNet and NSG concepts
- AZ-104 Certification — In Progress

---

## 📬 Contact

**Yasir Munir**  
NOC Engineer II — Stewart Title  
[LinkedIn](https://www.linkedin.com/in/yasirmunir) | yasirmunir616@gmail.com
