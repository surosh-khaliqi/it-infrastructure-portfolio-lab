# Phase 4 — Security: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Router ACLs | `ufw` route allow-list + `default deny routed` on both `RTR-SITE1` and `RTR-SITE2` |
| Allowed traffic | Required AD, DHCP relay, Samba, Zabbix, and management SSH paths |
| DC hardening | `wuauserv` restored to Automatic/Running; `LockoutThreshold` set to 5 |
| Service-account hygiene | Undocumented Zabbix sudoers `NOPASSWD` grant removed |
| SSH hardening | Key-only authentication active on `RTR-SITE1`; password authentication rejected |
| Service reduction | Unused MySQL X Protocol listener on `MON-SRV` disabled |
| Audit | Nmap and manual checks completed across all 7 hosts; findings remediated or documented where no change was required |
| ServiceNow documentation | 5 Incident-table records documenting significant findings, changes, work notes, root cause, and resolution |
| Documentation | [`01-decisions.md`](01-decisions.md), [`02-build-log.md`](02-build-log.md), this summary |

## Verified Working

Full test matrix and evidence are in [`02-build-log.md`](02-build-log.md), Section 6 (Validation). Summary:

- Site 2 DHCP and AD authentication restored after the two-layer ACL correction
- Critical inter-VLAN and inter-site service paths remain functional under the default-deny routed policy
- SSH key-only authentication confirmed on `RTR-SITE1`
- DC hardening settings verified after the ACL work
- Zabbix sudoers grant confirmed removed
- All five ServiceNow records completed with chronological work notes and resolution evidence

## Lab Scope & Constraints

- **Single Domain Controller and Site 2 dependency on the WAN link** remain unchanged from Phase 2. The router ACLs restrict the paths but do not add service redundancy.
- **SSH key-only authentication was applied only to `RTR-SITE1`.** Other Linux hosts retain their existing SSH authentication configuration.
- **Router ACLs are network- and port-based controls.** No identity-aware or application-layer filtering was implemented.
- **TempNAT adapters and scoped NAT routes on MON-SRV / Windows hosts** from Phase 3 are still present, left in place for continuity.
- **ServiceNow Personal Developer Instance uses the Incident table** for all five records, with type labels used where a record represents a problem or change.
- **Host memory remains constrained.** Dynamic Memory is enabled and selective power-on was used where necessary during validation.
- **Group Policy and automated remediation were outside Phase 4 scope.**

## Final Project State

Phase 4 closes the four-phase build with routed default-deny controls active on both routers, key-only SSH verified on `RTR-SITE1`, identified service and privilege findings remediated, and the critical Phase 2 and Phase 3 paths revalidated under the hardened configuration.

The final architecture, remaining lab constraints, verification guidance, and handover details are documented in the project [`README.md`](../README.md) and [`docs/final_handover.md`](../docs/final_handover.md).

---

[← Main README](../README.md) · [01 — Design & Decisions](01-decisions.md) · [02 — Build Log](02-build-log.md)
