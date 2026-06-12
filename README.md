# Azure Firewall Network Security Lab

**Author:** Adebola Shopeju  
**Platform:** Microsoft Azure  
**Track:** Darey.io Cloud Engineering  

---

## Project Overview

This project demonstrates enterprise-grade network security using Azure Firewall — a fully managed, stateful Firewall-as-a-Service (FWaaS). The lab covers the full lifecycle of deploying and configuring a centralized firewall, including traffic filtering, NAT rules, threat intelligence, and monitoring.

---

## Architecture

```
Internet
    |
    v
[lab-firewall-pip] (Public IP: 20.124.207.251)
    |
[lab-firewall] (Azure Firewall Standard — Private IP: 10.0.0.4)
    |
[firewall-lab-vnet] (10.0.0.0/16)
    |-- AzureFirewallSubnet (10.0.0.0/26)  ← firewall lives here
    |-- App-Subnet (10.0.1.0/24)           ← workload VMs live here
            |
        [test-vm] (Ubuntu — 10.0.1.4)
```

All outbound traffic from App-Subnet is forced through the firewall via a User Defined Route (UDR). Inbound SSH access is available only through the DNAT rule on the firewall's public IP.

---

## Deployment Steps

### Phase 1 — Foundation

1. Created resource group `azure-firewall-lab-rg` in East US
2. Created VNet `firewall-lab-vnet` with address space `10.0.0.0/16`
3. Created subnet `AzureFirewallSubnet` (`10.0.0.0/26`) — required name for Azure Firewall
4. Created workload subnet `App-Subnet` (`10.0.1.0/24`)

### Phase 2 — Firewall Deployment & Routing

1. Deployed Azure Firewall (Standard SKU) as `lab-firewall` into `AzureFirewallSubnet`
   - Private IP: `10.0.0.4`
   - Public IP: `20.124.207.251` (resource: `lab-firewall-pip`)
2. Created route table `firewall-route-table`
3. Added UDR `to-firewall`: `0.0.0.0/0` → Virtual Appliance → `10.0.0.4`
4. Associated route table with `App-Subnet`
5. Deployed `test-vm` (Ubuntu, `10.0.1.4`) in App-Subnet
6. Verified effective routes on test-vm NIC confirm firewall as next hop

### Phase 3 — Traffic Filtering

**Network Rule Collection** (`allow-app-subnet-rules`, Priority 200, Allow)
- Rule `allow-dns`: UDP, source `10.0.1.0/24` → any destination, port 53

**Application Rule Collection** (`allow-app-subnet-fqdns`, Priority 200, Allow)
- Rule `allow-windows-update`: source `10.0.1.0/24`, FQDN tag = WindowsUpdate

**Blocking verified:** `curl -m 5 http://www.google.com` from test-vm returned:
```
Action: Deny. Reason: No rule matched. Proceeding with default action.
```

**DNAT Rule Collection** (`dnat-inbound-rules`, Priority 100)
- Rule `allow-ssh-inbound`: TCP, source `*`, destination `20.124.207.251:2222` → `10.0.1.4:22`

**DNAT verified:** `nc -zv 20.124.207.251 2222` returned:
```
Connection to 20.124.207.251 2222 port [tcp/*] succeeded!
```

### Phase 4 — Threat Intelligence

- Enabled Threat Intelligence mode: **Alert and deny**
- Firewall now blocks and alerts on traffic to/from known malicious IPs and domains using Microsoft's threat feed

### Phase 5 — Monitoring

1. Created Log Analytics workspace `firewall-log-workspace` (East US)
2. Enabled Diagnostic Settings (`firewall-diagnostics`) on lab-firewall:
   - Azure Firewall Network Rule logs
   - Azure Firewall Application Rule logs
   - Azure Firewall NAT Rule logs
   - Azure Firewall Threat Intelligence logs
3. Destination: `firewall-log-workspace`
4. Verified logs appear in Log Analytics using KQL query:
```kusto
AzureDiagnostics
| where Category contains "AzureFirewall"
| order by TimeGenerated desc
| take 20
```

---

## Screenshots

| File | Description |
|---|---|
| `resource-group-overview.png` | Resource group with all deployed resources |
| `vnet-and-subnets.png` | VNet with AzureFirewallSubnet and App-Subnet |
| `workload-subnet.png` | App-Subnet configuration |
| `firewall-overview.png` | Firewall public/private IP overview |
| `route-table-overview.png` | Route table overview |
| `route-table-udr.png` | UDR showing firewall as Next Hop |
| `subnet-association.png` | Route table associated with App-Subnet |
| `effective-routes.png` | Effective routes on test-vm NIC |
| `network-rules.png` | Network rule collection |
| `application-rules.png` | Application rule collection |
| `blocked-site.png` | Curl output showing firewall deny action |
| `nat-dnat-rules.png` | DNAT rule collection |
| `dnat-test.png` | Successful nc connection through DNAT rule |
| `threat-intelligence-config.png` | Threat intelligence set to Alert and deny |
| `diagnostic-settings.png` | Diagnostic settings configured |
| `monitoring-logs.png` | Log Analytics query results |

---

## Key Concepts Demonstrated

- **Defense in depth:** Layered rules (Network → Application → NAT) with default-deny behaviour
- **Forced tunneling:** UDR redirects all traffic through the firewall regardless of destination
- **Principle of least privilege:** Only explicitly allowed traffic passes; everything else is denied
- **Threat intelligence:** Microsoft's global feed proactively blocks known malicious actors
- **Centralized logging:** All firewall decisions captured in Log Analytics for audit and analysis
