# Phase 2 — Servers: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Directory services | Single AD domain `ans.local` on `SRV-S1-SERVERS` (sole Domain Controller) |
| DNS | AD-integrated forward zone + four reverse zones; both Windows clients and `SRV-S2-SERVERS` use `10.10.12.10` for DNS |
| DHCP | Two scopes (`DHCP-S1-USERS`, `DHCP-S2-USERS`); relay agents on both routers |
| Addressing | Users-VLAN PCs converted to DHCP; servers and routers remain static |
| Domain join | Both Windows clients + `SRV-S2-SERVERS` (Linux via realmd/SSSD) joined |
| Time sync | `SRV-S1-SERVERS` authoritative (Stratum 1); `SRV-S2-SERVERS` synced via chrony |
| File sharing | Native SMB share on Windows + Samba share on Linux; AD group ACLs enforced on both |
| AD objects | Site1/Site2 OUs, two security groups, two sample users |
| Documentation | [`01-decisions.md`](01-decisions.md), [`02-build-log.md`](02-build-log.md), this summary |

## Verified Working

Full test matrix and evidence are in [`02-build-log.md`](02-build-log.md), Section 9 (Validation). Summary:

- DNS full-mesh resolution, including DHCP-registered client records
- DHCP leases active on both Users VLANs
- Time hierarchy verified between `SRV-S1-SERVERS` and `SRV-S2-SERVERS`
- Kerberos tickets present for cross-platform access
- `dcdiag` passed except `DFSREvent`; `SysVolCheck` passed independently
- Group-based file-share enforcement confirmed on both platforms (ReadWrite write succeeds; ReadOnly write denied, read succeeds)

## Lab Scope & Constraints

- **Single Domain Controller.** `SRV-S1-SERVERS` carries AD DS, DNS, DHCP, and the Windows file share. There is no second DC or service redundancy in this lab.
- **Site 2 depends on the WAN link for central AD, DNS, and DHCP services.** If the WAN is unavailable, access to those central services is affected; existing leases or cached credentials may continue to work until renewal or fresh authentication is required.
- **SSSD dynamic DNS registration was not reliable in this build.** The A record for `SRV-S2-SERVERS` was added manually. See Issue 16 in the build log.
- **Windows Firewall remains disabled on the Windows end devices** (carried from Phase 1) as a lab convenience.
- **Host memory constrains simultaneous VM operation.** Validation was run in subsets when necessary.
- **Monitoring and security controls are outside Phase 2 scope.** Monitoring is added in Phase 3; firewall/ACL and SSH hardening work is added in Phase 4.

## What Phase 3 Inherits

- The routed VLAN/subnet structure from Phase 1, unchanged.
- A working AD domain with DNS, DHCP, domain-joined Windows clients, and a Linux domain member.
- Cross-platform file shares with AD group-based access control.
- Time synchronization between the Windows domain server and Linux member server.
- Static addressing on infrastructure devices and DHCP on the Users VLANs.
- AD/DNS/DHCP services, time sync, Samba, and routed connectivity ready to be monitored in Phase 3.

---

[← Main README](../README.md) · [01 — Design & Decisions](01-decisions.md) · [02 — Build Log](02-build-log.md)
