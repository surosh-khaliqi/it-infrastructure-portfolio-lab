# Phase 2 — Servers: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Directory services | Single AD domain `ans.local` on `SRV-S1-SERVERS` (sole Domain Controller) |
| DNS | AD-integrated forward zone + four reverse zones; all devices re-pointed to 10.10.12.10 |
| DHCP | Two scopes (`DHCP-S1-USERS`, `DHCP-S2-USERS`); relay agents on both routers |
| Addressing | Users-VLAN PCs converted to DHCP; servers/routers remain static |
| Domain join | Both Windows clients + `SRV-S2-SERVERS` (Linux via realmd/SSSD) joined |
| Time sync | `SRV-S1-SERVERS` authoritative (Stratum 1); `SRV-S2-SERVERS` synced via chrony |
| File sharing | Native SMB share on Windows + Samba share on Linux; AD group ACLs enforced on both |
| AD objects | Site1/Site2 OUs, two security groups, two sample users |
| Documentation | `01-decisions.md`, `02-build-log.md`, this summary |

## Verified Working

Full test matrix and evidence are in `02-build-log.md`, Section 9 (Validation). Summary:

- DNS full-mesh resolution (including DHCP-registered records)
- DHCP leases active on both Users VLANs
- Time hierarchy healthy
- Kerberos tickets present for cross-platform access
- AD health diagnostic clean (single known low-risk exception: DFSREvent on a single-DC forest)
- Group-based file-share enforcement confirmed on both platforms (ReadWrite write succeeds; ReadOnly write denied, read succeeds)

No unresolved failures.

## Lab Limitations

- **Single Domain Controller.** `SRV-S1-SERVERS` carries AD DS, DNS, DHCP, and the file server simultaneously. A production design would separate these roles or add a second DC for redundancy. Acceptable for a lab meant to prove the services work, not high availability.
- **Site 2 depends on the WAN link for directory, DNS, and DHCP.** With no local DC or DHCP server at Site 2, loss of the WAN link takes authentication, name resolution, and address renewal offline for Site 2 devices. This is realistic branch-office behaviour and is stated explicitly rather than left as a surprise.
- **SSSD dynamic DNS registration does not work reliably.** The A record for `SRV-S2-SERVERS` was added manually. Documented as a known, accepted limitation (see Issue 17 in the build log).
- **Windows Firewall remains fully disabled on end devices** (carried from Phase 1). A lab convenience only; not production practice.
- **Host memory still constrains simultaneous VM operation.** Tests continue to be run in subsets. Does not affect the validity of any result.
- **No Group Policy, monitoring, or firewall/ACL work yet.** By design — those belong to Phases 3 and 4.

## What Phase 3 Inherits

- The complete routed VLAN/subnet structure from Phase 1, unchanged.
- A live AD domain with DNS, DHCP, domain-joined clients, and a Linux member server.
- Working cross-platform file shares with AD group enforcement.
- Time synchronization in place.
- Static addressing on infrastructure devices; DHCP on the Users VLANs.
- All of the above are ready to be monitored (DNS queries, DHCP leases, authentication events, file-share access) in Phase 3.
