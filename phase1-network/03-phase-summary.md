# Phase 1 — Network: Phase Summary

## What Was Delivered

| Component | Detail |
|---|---|
| Sites | 2 (Site 1, Site 2), connected via a routed WAN link |
| VLANs | 2 per site (Users / Servers), trunked via Hyper-V vSwitch, 802.1Q |
| Routers | `RTR-SITE1`, `RTR-SITE2` — Ubuntu Server, static routing, VLAN sub-interfaces |
| End devices | 4 — Windows 11 clients (Users VLAN, both sites), Windows Server + Ubuntu Server (Servers VLAN, one per site) |
| IP addressing | `/24` per VLAN, `/30` WAN link, third-octet `{site}{vlan}` scheme (see `01-decisions.md`) |
| Documentation | `01-decisions.md`, `02-build-log.md`, this summary |

## Verified Working

Full test matrix and evidence are in `02-build-log.md`, Section 5 (Validation). Summary:
local gateway reachability, inter-VLAN routing within a site, cross-site routing over the
WAN link to both VLANs, path verification via `tracert`, and VLAN segmentation
enforcement (disabling a sub-interface correctly blocks routed traffic to that segment) all
passed. No unresolved failures — see `02-build-log.md`, Section 7, for issues encountered
and how each was fixed.

## Lab Limitations

- **Static routing only.** Dynamic routing (FRRouting/OSPF) was scoped as a stretch goal
  and not implemented. Two routers with a single path between them don't create a
  scenario where dynamic routing would add measurable value — static routes are the
  correct, standard choice at this scale, not a shortcut.
- **No router or WAN link redundancy.** Each site has a single router and there is one WAN
  path between sites. A production network would have redundant links/routers; this is a
  single point of failure by design, acceptable for a lab meant to prove routing and VLAN
  fundamentals rather than high availability.
- **Windows Firewall disabled entirely on end devices**, rather than a scoped ICMP-only
  rule. A targeted rule was attempted first and failed on a display-name mismatch; fully
  disabling the firewall was accepted as a lab-only convenience to unblock validation
  testing, not a production-appropriate practice.
- **Host memory constrained validation sequencing.** The 16 GB nested-virtualization host
  can't run all 6 guest VMs simultaneously without hitting memory errors. Tests were run
  in pairs/subsets, starting and stopping VMs as needed — this affected build convenience
  only, not the validity of any test result.
- **No services, ACLs, or monitoring yet.** By design — DNS/DHCP/AD lands in Phase 2,
  monitoring in Phase 3, and firewalls/ACLs at VLAN boundaries in Phase 4. Nothing in
  Phase 1 blocks any of that work.

## What Phase 2 Inherits

- The routed VLAN/subnet structure built here, unchanged.
- `SRV-S1-SERVERS` (Windows Server 2022 Standard) as the target for AD DS, DNS, and DHCP
  — already domain-joinable, no rebuild needed.
- `SRV-S2-SERVERS` (Ubuntu Server) as the cross-platform Linux server target.
- Static IP addressing on all current devices will need to coexist with or transition to
  DHCP-issued addressing once Phase 2's DHCP server is live — a decision for Phase 2's
  `01-decisions.md`, not resolved here.
