# Phase 1 — Network: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Sites | 2 — Site 1 and Site 2, connected through a routed WAN link |
| VLANs | 2 per site — Users (VLAN 10) and Servers (VLAN 20), carried through Hyper-V 802.1Q trunks |
| Routers | `RTR-SITE1`, `RTR-SITE2` — Ubuntu Server with VLAN sub-interfaces and static routing |
| End devices | 4 — two Windows 11 clients, one Windows Server, and one Ubuntu Server |
| IP addressing | `/24` per VLAN, `/30` on the WAN link, using the `{site}{vlan}` third-octet scheme |
| Documentation | [01-decisions.md](01-decisions.md), [02-build-log.md](02-build-log.md), and this summary |

Phase 1 created the routed network foundation used by the rest of the project. DNS, DHCP, Active Directory, monitoring, and security controls were intentionally left for later phases.

---

## Verified Working

The full validation sequence and screenshots are documented in [02-build-log.md](02-build-log.md), Section 5.

Phase 1 validation confirmed:

- local gateway reachability
- inter-VLAN routing within Site 1
- cross-site routing over the WAN link to both Site 2 VLANs
- the expected multi-hop path using `tracert`
- VLAN sub-interface behaviour by disabling and restoring the Site 1 VLAN 20 interface

All Phase 1 validation tests passed after the documented configuration issues were corrected.

---

## Lab Scope & Constraints

- **Static routing:** Static routes were used for the two-router, single-path topology. Dynamic routing was not required for this phase.
- **No network redundancy:** Each site has one router and there is one WAN path between the sites.
- **Windows Firewall during Phase 1:** Windows Firewall was disabled on the Windows guests during connectivity testing. It was re-enabled later in the project with scoped rules.
- **Host memory:** The 16 GB Hyper-V host created memory pressure during Phase 1, so some validation was completed with only the VMs required for each test powered on. Later phases moved to Dynamic Memory, and the final environment was validated with all seven VMs running.
- **No server services or monitoring yet:** Phase 1 focused only on the network foundation. AD DS, DNS, DHCP, and file services were added in Phase 2, monitoring in Phase 3, and security controls in Phase 4.

---

## What Phase 2 Inherits

Phase 2 starts with the routed network from Phase 1 already in place:

- the same two-site VLAN and subnet structure
- `RTR-SITE1` and `RTR-SITE2` providing inter-VLAN and inter-site routing
- `SRV-S1-SERVERS` already built and addressed, ready to take the AD DS, DNS, and DHCP roles
- `SRV-S2-SERVERS` available as the Linux server for domain-join and Samba work
- both Windows clients connected to their Users VLANs

The Windows clients use static `.10` addresses in Phase 1. Phase 2 introduces DHCP for the Users VLANs, while routers and servers keep static addressing.

---

[Back to Phase 1 design decisions](01-decisions.md) · [View Phase 1 build log](02-build-log.md) · [Return to project README](../README.md)
