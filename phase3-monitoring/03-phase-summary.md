# Phase 3 — Monitoring: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Monitoring host | `MON-SRV` (Ubuntu Server, 10.10.12.20, VLAN 20) running Zabbix 7.0 LTS + MySQL |
| Agent coverage | Zabbix agent on all 6 existing devices + MON-SRV itself |
| Service checks | AD DS (NTDS/DNS/Netlogon), DHCP (service + lease counts), W32Time, chrony, Samba (smbd/nmbd) |
| Simple checks | DNS zone health (SOA/PTR), SSH reachability to routers + SRV-S2-SERVERS |
| Host-down detection | ICMP Ping template linked to all 7 hosts |
| WAN monitoring | Latency + packet-loss items on dedicated `network-links` host object |
| Triggers | 8 custom service triggers (Disaster/High/Average) + ICMP auto-triggers |
| Alerting | Gmail media type + “Notify on High+ severity” action |
| ServiceNow | Personal Developer Instance with 3 assignment groups, custom categories, integration user |
| Documentation | `01-decisions.md`, `02-build-log.md`, this summary |

## Verified Working

Full test matrix and evidence are in `02-build-log.md`, Section 6 (Validation). Summary:

- All 7 hosts green
- Core service items returning healthy values
- Custom triggers fire and resolve correctly
- Email notification path works for High/Disaster
- WAN link metrics live
- ServiceNow incident lifecycle (create → close) demonstrated via REST

No unresolved failures.

## Lab Limitations

- **Single monitoring host, no high availability.** `MON-SRV` is a single point of failure for visibility. Acceptable for a lab meant to prove the monitoring stack works, not production resilience.
- **TempNAT + split-default-route retained on three Windows hosts.** Left in place after agent install for convenience; scheduled for removal (adapter + persistent routes) at project close. Depends on the explicit `10.10.0.0/16` exception route remaining intact.
- **MON-SRV outbound internet limited to scoped `/32` routes.** Only the destinations required for email and the ServiceNow API are routed via NAT. ServiceNow developer-instance IPs can rotate; the hardcoded route may need occasional manual refresh.
- **Host memory still constrains simultaneous VM operation.** Dynamic Memory helped, but tests continue to be run in subsets. Does not affect the validity of any result.
- **No automated remediation or Group Policy.** By design — those belong to Phase 4.
- **Windows Firewall, disabled entirely since Phase 1, was re-enabled** on all three Windows hosts before agent install, with rules scoped to AD/DNS/DHCP/file-sharing/ICMP. Full hardening beyond these rules is still deferred to Phase 4.
- Single Domain Controller and Site-2 dependency on the WAN link (from Phase 2) remain unchanged.
- The permanent `ip_forward` / ufw sysctl fix is in place on both routers, but should still be spot-checked after any future router reboot.

## What Phase 4 Inherits

- A live, fully instrumented environment: every host and the core Phase 2 services are already under Zabbix.
- Working trigger and notification paths that Phase 4 automation can replace or extend.
- ServiceNow structure (groups, categories, integration user) ready for any scripted ticket creation.
- The same routed, domain-joined foundation from Phases 1–2, unchanged.
- Known temporary artifacts (TempNAT adapters, scoped NAT routes on MON-SRV) that should be cleaned up or hardened as part of Phase 4 security work.
