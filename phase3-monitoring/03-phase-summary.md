# Phase 3 — Monitoring: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Monitoring host | `MON-SRV` (Ubuntu Server, `10.10.12.20`, VLAN 20) running Zabbix 7.0 LTS + MySQL |
| Agent coverage | Zabbix agent on all 7 lab VMs |
| Service checks | AD DS (NTDS/DNS/Netlogon), DHCP (service + lease counts), W32Time, chrony, Samba (`smbd`/`nmbd`) |
| Simple checks | DNS zone health (SOA/PTR), SSH reachability to both routers + `SRV-S2-SERVERS` |
| Host-down detection | ICMP Ping template linked to all 7 hosts |
| WAN monitoring | Latency + packet-loss items on dedicated `network-links` host object |
| Triggers | 8 custom service triggers (Disaster/High/Average) + ICMP auto-triggers |
| Alerting | Gmail media type + `Notify on High+ severity` action |
| ServiceNow | Personal Developer Instance with 3 assignment groups, custom categories, integration user, and REST incident creation/tracking |
| Documentation | [`01-decisions.md`](01-decisions.md), [`02-build-log.md`](02-build-log.md), this summary |

## Verified Working

Full test matrix and evidence are in [`02-build-log.md`](02-build-log.md), Section 6 (Validation). Summary:

- All 7 hosts green
- Core service items returning healthy values
- Custom triggers fire and resolve correctly
- Email notification path works for High/Disaster
- WAN link metrics live
- ServiceNow incident lifecycle (create → close) demonstrated via REST

## Lab Scope & Constraints

- **Single monitoring host, no high availability.** `MON-SRV` is the only Zabbix server in the lab.
- **TempNAT + split-default-route retained on three Windows hosts.** Left in place after agent install for continuity. Depends on the explicit `10.10.0.0/16` exception route remaining intact.
- **MON-SRV outbound internet limited to scoped `/32` routes.** Only the destinations required for email and the ServiceNow API are routed via NAT. ServiceNow developer-instance IPs can rotate, so the hardcoded route may need to be updated if the destination changes.
- **Host memory constrains simultaneous VM operation.** Dynamic Memory reduced pressure on the 16 GB host, but validation was still run in subsets when necessary.
- **Automated remediation and Group Policy were outside Phase 3 scope.**
- **Windows Firewall was re-enabled** on all three Windows hosts before agent installation, with rules for AD, DNS, DHCP, file sharing, ICMP, and Zabbix traffic. Broader hardening is handled in Phase 4.
- **Single Domain Controller and Site 2 WAN dependency** from Phase 2 remain unchanged.
- **The permanent `ip_forward` / UFW sysctl fix is in place** on both routers and was verified during Phase 3.

## What Phase 4 Inherits

- All 7 lab VMs under Zabbix monitoring
- Core Phase 2 services monitored with custom items and triggers
- Working email notification and WAN monitoring paths
- ServiceNow groups, categories, integration user, and REST incident integration
- The routed, domain-joined infrastructure from Phases 1 and 2
- A monitoring baseline that can show whether Phase 4 security changes affect host or service availability
- TempNAT adapters and scoped NAT routes on MON-SRV, left in place for continuity.

---

[← Main README](../README.md) · [01 — Design & Decisions](01-decisions.md) · [02 — Build Log](02-build-log.md)
