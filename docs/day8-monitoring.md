# Day 8 — Azure Monitor + Network Watcher

## Objective
Deploy observability across the entire hub-spoke lab — firewall logging, App Gateway access logs, VM alerting, and continuous connectivity monitoring via Connection Monitor.

## Resources Created

### Log Analytics Workspace — `hub-log-analytics`
| Property | Value |
|---|---|
| Region | UAE North |
| Resource group | `stewart-lab-RG` |
| Purpose | Central log collection for all lab resources |

### Diagnostic Settings

**Azure Firewall — `firewall-diagnostics`**
| Log Category | Enabled |
|---|---|
| AzureFirewallNetworkRule | ✅ |
| AzureFirewallApplicationRule | ✅ |
| Destination | `hub-log-analytics` |

**Application Gateway — `appgw-diagnostics`**
| Category | Enabled |
|---|---|
| ApplicationGatewayAccessLog | ✅ |
| ApplicationGatewayFirewallLog | ✅ |
| AllMetrics | ✅ |
| Destination | `hub-log-analytics` |

### Connection Monitor — `spoke-connectivity-monitor`
| Property | Value |
|---|---|
| Source | `web-server-02` (spoke1-vnet) |
| Test configuration | HTTP, port 80, every 30 seconds |
| Destinations | `google-dns` (8.8.8.8), `www.google.com:80` |
| Workspace | `hub-log-analytics` |

### VM Alert — `web-server-high-cpu`
| Property | Value |
|---|---|
| Metric | Percentage CPU |
| Threshold | > 80% |
| Aggregation | Average |
| Check every | 1 minute |
| Lookback | 5 minutes |
| Severity | 2 — Warning |
| Action group | `noc-alerts` → email: yasirmunir616@gmail.com |

## Troubleshooting Encountered

### Connection Monitor — 100% Checks Failed
**Initial test:** HTTP port 80 to `8.8.8.8`
**Result:** 100% failed

**Diagnosis:**
- `8.8.8.8` is Google's DNS server — listens on port 53, not port 80
- Azure Firewall application rules match by FQDN — raw IP HTTP requests have no FQDN to match against
- Firewall correctly blocked the HTTP-to-IP request

**Fix:** Added `www.google.com:80` as second destination

**Final results:**
| Destination | Tests Failed |
|---|---|
| `google-dns` (8.8.8.8:80) | 100% — expected, wrong protocol |
| `http://www.google.com:80` | 0% — passing through firewall ✅ |

**Key insight:** Connection Monitor correctly detected a real failure and distinguished between two different test paths. This is exactly how ThousandEyes works at enterprise scale — same concept, Azure-native implementation.

## Log Queries Used

### Firewall Network Rule Logs
```kusto
AzureDiagnostics
| where ResourceType == "AZUREFIREWALLS"
| where TimeGenerated > ago(1h)
| project TimeGenerated, msg_s
| order by TimeGenerated desc
| take 20
```

### App Gateway Access Logs
```kusto
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where TimeGenerated > ago(1h)
| project TimeGenerated, requestUri_s, httpStatus_d, clientIP_s
| order by TimeGenerated desc
| take 20
```

## Error Encountered — VM Generalized

### Issue: web-server-01 permanently unusable
**Error:** `Operation 'Start VM' is not allowed on VM 'web-server-01' since the VM is generalized`

**Cause:** The `waagent -deprovision` command was run on the VM (during backup/image capture attempt), permanently generalizing it. A generalized VM cannot be started again.

**Fix:** Deployed `web-server-02` as replacement
- `Standard_B1s` not available in UAE North (capacity restriction)
- Deployed with `Standard_B2s` instead
- Nginx installed, portfolio page redeployed
- App Gateway backend pool updated to new VM IP
- Private DNS A record updated to new IP

**Error codes learned:**
| Code | Meaning |
|---|---|
| 409 Conflict | Operation attempted while previous operation in progress |
| 502 Bad Gateway | App Gateway reached but backend VM unreachable |
| SkuNotAvailable | VM size has no capacity in selected region |

## NOC Parallel — Azure vs Stewart Tools

| Stewart Tool | Azure Equivalent | Purpose |
|---|---|---|
| SolarWinds | Azure Monitor + Log Analytics | Infrastructure monitoring |
| ThousandEyes | Network Watcher Connection Monitor | Synthetic connectivity testing |
| LogicMonitor | Azure Monitor Alerts + Action Groups | Threshold alerting + notifications |
| Manual packet capture | Network Watcher Packet Capture | Deep packet inspection |
| Topology diagrams | Network Watcher Topology | Auto-generated resource maps |

## Screenshots
See [../screenshots/day8/](../screenshots/day8/)
