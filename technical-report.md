# Technical Report — Azure Firewall Lab

**Author:** Adebola Shopeju  
**Date:** June 2026  
**Platform:** Microsoft Azure  
**Track:** Darey.io Cloud Engineering  

---

## Overview

This report reflects on the key technical challenges encountered during the Azure Firewall lab, the decisions made to resolve them, and the lessons learned that will carry forward into real-world cloud security work.

---

## Challenge 1: Asymmetric Routing — Direct SSH Breaks After UDR Association

### What happened

After associating the route table with App-Subnet in Phase 2, all direct SSH connections to test-vm's own public IP began timing out — even though the VM was running and the SSH port was open.

### Root cause

This is a classic **asymmetric routing** problem. When a UDR forces all outbound traffic through the firewall (`0.0.0.0/0 → 10.0.0.4`), the reply packets from the VM travel back via the firewall rather than directly to the client. The firewall sees these reply packets as unsolicited inbound traffic with no matching session, and silently drops them. The connection never completes.

```
Client → VM public IP (inbound, bypasses firewall)
VM → Client (outbound, forced through firewall → DROPPED)
```

The firewall requires symmetric traffic flow: both directions of a connection must pass through it.

### Resolution

Two workarounds were explored:

1. **Azure Serial Console** — attempted but not available without prior password setup on the VM
2. **Azure Run Command (RunShellScript)** — succeeded. This feature executes commands directly on the VM via the Azure control plane, completely bypassing the network path. It became the primary method for all subsequent VM testing.

### Lesson learned

Always plan for VM access *before* locking down routing. In production environments, a jump box (bastion host) or Azure Bastion service should be deployed before applying restrictive UDRs. The DNAT rule created in Phase 3 — forwarding `public-IP:2222 → VM:22` — is the correct architectural solution: inbound SSH now flows symmetrically through the firewall in both directions.

---

## Challenge 2: Discovering Run Command as a Testing Tool

### What happened

Losing direct SSH access could have been a significant blocker. Instead, discovering Azure Run Command turned it into a learning opportunity.

### What Run Command does

Run Command executes shell scripts on a VM through the Azure Resource Manager API — not through the network. It does not use SSH, does not traverse the VNet, and is not affected by NSGs, route tables, or firewall rules. This makes it ideal for:

- Testing whether the firewall is blocking traffic (the test runs *from* inside the network)
- Diagnosing routing issues without needing network access
- Running commands on locked-down VMs where no inbound ports are open

### How it was used in this lab

| Command | Purpose |
|---|---|
| `curl -m 5 http://www.google.com` | Confirm firewall blocks non-allowed outbound traffic |
| `nc -zv 20.124.207.251 2222` | Confirm DNAT rule forwards inbound connection correctly |
| `curl -m 5 http://www.google.com; nc -zv 20.124.207.251 2222` | Generate both allowed and blocked traffic for Log Analytics |

### Lesson learned

Cloud environments offer control-plane tools (Run Command, Serial Console, Azure Bastion) that operate independently of the data plane. Understanding the difference between control-plane and data-plane access is essential for troubleshooting locked-down environments — and is a skill directly applicable to real incident response scenarios.

---

## Reflections on the Firewall Rule Model

Working through the three rule types — Network, Application, and NAT — made the layered security model concrete in a way that documentation alone cannot.

**Network rules** operate at Layer 4 (IP + port). They are fast and precise but cannot distinguish between, say, legitimate Windows Update traffic and a malicious connection on the same port.

**Application rules** operate at Layer 7 (FQDN). They can allow `*.windowsupdate.microsoft.com` while blocking everything else on port 443 — including sites that look legitimate. This is far more granular than anything achievable with NSGs alone.

**NAT rules** invert the model: instead of filtering outbound traffic, they selectively expose internal resources inbound. The implicit network rule Azure adds when a DNAT rule matches is an important detail — it means the return traffic is automatically permitted without a separate rule.

The default-deny posture throughout — where *anything not explicitly allowed is blocked* — is the correct mental model for enterprise network security.

---

## Summary

| Phase | Key Outcome |
|---|---|
| Foundation | VNet and subnet architecture established with correct addressing |
| Firewall + Routing | UDR successfully forces all App-Subnet traffic through firewall |
| Traffic Filtering | All three rule types (Network, Application, DNAT) deployed and verified |
| Threat Intelligence | Alert and deny mode active; Microsoft feed integrated |
| Monitoring | Diagnostic logs flowing to Log Analytics; KQL queries operational |

The most durable takeaway from this lab: a firewall is only as effective as the routing that forces traffic through it. The UDR is not an optional add-on — it is the mechanism that makes the firewall meaningful.
