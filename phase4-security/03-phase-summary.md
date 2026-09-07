# Phase 4 — Security & Automation: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Router ACLs | `ufw` route allow-list + default deny routed on both RTR-SITE1 and RTR-SITE2 |
| Allowed traffic | AD-core (TCP+UDP), Samba, Zabbix (both directions), SSH from MON-SRV, DHCP-relay reply path |
| DC hardening | `wuauserv` restored to Automatic/Running; LockoutThreshold set to 5 |
| Service-account hygiene | Undocumented zabbix sudoers NOPASSWD grant removed |
| SSH hardening | Key-only authentication live on RTR-SITE1 (password rejected) |
| Audit | Full Nmap + manual checks on all 7 hosts; findings resolved or documented as intentional |
| ITIL documentation | 5 ServiceNow tickets (Incident / Problem / Change analogues) with full Work Notes and Resolution |
| Documentation | `01-decisions.md`, `02-build-log.md`, this summary |

## Verified Working

Full test matrix and evidence are in `02-build-log.md`, Section 6 (Validation). Summary:

- Site 2 DHCP and AD authentication restored after the two-layer ACL fix
- Inter-VLAN and inter-site services still function under the allow-list
- SSH key-only authentication confirmed on RTR-SITE1
- DC hardening settings persist
- zabbix sudoers grant removed
- All five ServiceNow tickets closed with chronological evidence

No unresolved failures.

## Lab Limitations

- **Single Domain Controller and Site-2 dependency on the WAN link** remain unchanged from Phase 2. The new ACLs protect the paths but do not add redundancy.
- **SSH key-only authentication applied only to RTR-SITE1.** The other Linux hosts still accept password authentication. Full rollout is left as a natural next step.
- **Router ACLs are host/port allow-lists, not identity-aware or application-layer controls.** They enforce least privilege for the services this lab actually uses; they are not a substitute for a full Zero-Trust design.
- **TempNAT adapters and scoped NAT routes on MON-SRV / Windows hosts** from Phase 3 are still present. They were left in place for continuity and remain scheduled for cleanup.
- **ServiceNow Personal Developer Instance lacks Problem and Change modules.** All five tickets live in the Incident table with bracketed type prefixes to preserve the intended classification.
- **Host memory still constrains simultaneous VM operation.** Dynamic Memory and selective power-on continue to be required.
- **No Group Policy objects or automated remediation scripts** were created. The phase demonstrates manual hardening and ticketed response, not continuous compliance automation.

## Project Close

Phase 4 is the final phase. The full as-built architecture, consolidated limitations, and handover framing will live in `docs/final-handover.md` and the project `README.md` once those cross-cutting documents are written.
