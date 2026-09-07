# Notable Issues

This document is a curated selection of the most instructive troubleshooting stories from the four-phase lab. It is not a complete catalogue of every problem encountered. Each entry is written to stand alone so a reader can follow the diagnostic path without opening the phase build logs.

The six stories below were chosen because they either spanned multiple phases, had non-obvious root causes, or demonstrated transferable diagnostic habits.

---

### 1 — The recurring `ip_forward` resets

**Where:** RTR-SITE1 and RTR-SITE2, across Phases 1–3

**Symptom:** Inter-VLAN or cross-site TCP traffic would suddenly fail while ICMP continued to work. The failure often appeared after a reboot, a package install, or a `ufw reload`.

**Diagnosis path:**  
In Phase 1 the symptom first appeared during validation. Checking `sysctl net.ipv4.ip_forward` returned `0` even though the setting had been written earlier. Temporary fixes restored forwarding, but the same reset reappeared in Phase 2 after router work and again in Phase 3 immediately after Linux agent and ufw changes. Each time the temporary sysctl.d drop-in restored service, only for the next ufw operation to set the value back to 0.

**Root cause:**  
ufw maintains its own sysctl file at `/etc/ufw/sysctl.conf`. On every enable or reload it re-applies the settings in that file. The line that enables forwarding was still commented out:

```ini
#net/ipv4/ip_forward=1
```

Edits made only to `/etc/sysctl.conf` or `/etc/sysctl.d/` were overwritten the next time ufw ran.

**Fix:**
```bash
sudo sed -i 's/#net\/ipv4\/ip_forward=1/net\/ipv4\/ip_forward=1/' /etc/ufw/sysctl.conf
sudo ufw reload
sysctl net.ipv4.ip_forward   # remains 1 after reboot
```
Applied on both routers.

**Lesson:** When a kernel setting keeps reverting after a tool that manages sysctl is involved, inspect that tool’s own configuration files, not only the usual sysctl locations. The observable symptom (ICMP works, TCP does not) is a reliable pointer to check `ip_forward` early.

---

### 2 — DHCP relay dropping replies as “bogus giaddr”

**Where:** RTR-SITE1 and RTR-SITE2, Phase 2

**Symptom:** Clients on the Users VLANs timed out on `ipconfig /renew`. Relay logs showed the request being forwarded, then the reply being discarded with the message `Dropping reply received on eth1 / BOOTREPLY giaddr: \ldots / Packet to bogus giaddr`.

**Root cause:**  
The default isc-dhcp-relay service unit applied a plain `-i` flag to every interface. That flag tells the relay to treat the interface as strictly client-facing and to enforce ownership of the giaddr. Legitimate replies that arrived on a transit (WAN) interface failed the ownership check and were dropped.

**Fix:**  
Classify interfaces explicitly in `/etc/default/isc-dhcp-relay`:

```text
OPTIONS="-id eth0.10 -iu eth0.20 -iu eth1"
```

`-id` marks the client-facing interface; `-iu` marks upstream or transit interfaces that must not enforce the giaddr ownership check. After the change, replies flowed correctly on both sites.

**Lesson:** In any multi-hop DHCP relay topology, every interface a reply could transit must be marked upstream. The distinction is poorly documented in the man page and is easy to miss when the relay appears to be “working” because it is forwarding the requests.

---

### 3 — Router ACLs broke Site 2 — missing UDP and the DHCP reply path

**Where:** Both routers, Phase 4

**Symptom:** Immediately after the first ACL allow-list and `default deny routed` were applied, Site 2 clients lost their DHCP leases (APIPA) and could no longer locate the domain controller. Site 1 remained healthy. ICMP and SSH from the monitoring host continued to work.

**Diagnosis path:**  
1. Confirmed the routers themselves were still reachable and that IP forwarding was enabled.  
2. Realised the original allow-list contained only TCP rules for the AD-core ports.  
3. Added matching UDP rules for 88, 389 and 464. Domain controller location began succeeding (`nltest /dsgetdc:ans.local`), but DHCP renewal still failed.  
4. Examined isc-dhcp-relay behaviour: the relay sets `giaddr` to its own client-facing interface address (the VLAN gateway), not the WAN address. Replies were therefore destined to 10.10.11.1 / 10.10.21.1 — a path the original ACL never permitted.

**Root cause:**  
Two independent gaps in the first allow-list:  
- AD-core services require UDP as well as TCP (CLDAP, Kerberos pre-authentication).  
- DHCP relay replies target the giaddr (VLAN gateway), not the router’s WAN address.

**Fix:**
```bash
# UDP for AD-core
sudo ufw route allow proto udp from 10.10.11.0/24 to 10.10.12.10 port 88
sudo ufw route allow proto udp from 10.10.21.0/24 to 10.10.12.10 port 88
# \ldots same pattern for 389 and 464

# DHCP relay reply path
sudo ufw route allow proto udp from 10.10.12.10 to 10.10.11.1 port 67
sudo ufw route allow proto udp from 10.10.12.10 to 10.10.21.1 port 67
sudo ufw reload
```

After the second layer of rules, Site 2 clients obtained leases, located the DC, and could access file shares again.

**Lesson:** When writing ACLs for Active Directory, always include the UDP counterparts of the well-known ports. For any relay protocol, verify with live logs where the reply is actually addressed before declaring the rule set complete.

---

### 4 — cloud-init drop-in kept password authentication enabled

**Where:** RTR-SITE1, Phase 4

**Symptom:** After editing `/etc/ssh/sshd_config` to set `PasswordAuthentication no` and restarting ssh, password logins still succeeded.

**Diagnosis path:**  
`sshd -T | grep -i passwordauthentication` still reported `passwordauthentication yes`. The main configuration file had been changed correctly; something else was overriding it.

**Root cause:**  
Ubuntu cloud-init places a drop-in at `/etc/ssh/sshd_config.d/50-cloud-init.conf` that explicitly sets `PasswordAuthentication yes`. Under first-match-wins evaluation order, the drop-in took precedence over the main file.

**Fix:**
```bash
sudo sed -i 's/^PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo systemctl restart ssh
sudo sshd -T | grep -i passwordauthentication   # now reports no
```

Final verification confirmed key-only login succeeded and password login was rejected with “Permission denied (publickey)”.

**Lesson:** On cloud-init-provisioned Ubuntu hosts, never trust the main `sshd_config` alone. Always verify the effective running configuration with `sshd -T` after any authentication change, and inspect the drop-in directory.

---

### 5 — NTFS inheritance silently granted write access to the ReadOnly group

**Where:** SRV-S1-SERVERS (`C:\TechShare`), Phase 2

**Symptom:** A user who was a member only of `SG-FileShare-ReadOnly` could successfully create files on the share, despite the share permissions and explicit NTFS ACEs that should have denied write access.

**Root cause:**  
The folder had inherited ACEs from the parent `C:\` volume. Those inherited entries granted the local Users group Append Data / Write Data rights. Domain users are members of the local Users group by default, so the inherited grant overrode the more restrictive explicit ACE.  
`icacls /remove:g` only removes explicit entries; it leaves inherited ones untouched.

**Fix:**
```powershell
icacls "C:\TechShare" /inheritance:r
icacls "C:\TechShare" /grant "SYSTEM:(OI)(CI)F"
icacls "C:\TechShare" /grant "BUILTIN\Administrators:(OI)(CI)F"
# explicit SG-FileShare-ReadWrite and SG-FileShare-ReadOnly grants remained
```

After inheritance was broken and the ACL rebuilt, the ReadOnly group could read but no longer write.

**Lesson:** When a share needs specific group-based restrictions, always check for and explicitly break inherited parent-folder permissions. Inherited entries will not be caught by a simple removal of explicit ACEs.

---

### 6 — Permanent NAT default route on MON-SRV broke internal connectivity

**Where:** MON-SRV, Phase 3

**Symptom:** After a permanent NAT adapter was added for outbound email and ServiceNow API access, the Zabbix GUI became briefly unreachable and multiple High-severity alerts fired. Internal pings to the Users VLAN failed while traffic to the local Servers VLAN continued to work.

**Root cause:**  
The netplan configuration for the NAT interface installed a default route with metric 50. The primary interface’s default route had metric 100. All traffic without a more-specific route — including destinations on 10.10.11.0/24 and the WAN link — was therefore pulled onto the NAT interface and black-holed.

**Fix:**  
Removed the competing default route and replaced it with scoped `/32` routes only for the external destinations that actually required internet access:

```bash
sudo ip route del default via 192.168.100.1 dev eth1
sudo ip route add 8.8.8.8/32 via 192.168.100.1 dev eth1
sudo ip route add <servicenow-resolved-ip>/32 via 192.168.100.1 dev eth1
```

Netplan was updated to match. Internal connectivity and monitoring recovered immediately.

**Lesson:** A permanent “internet” adapter on a monitoring host must never install a competing default route. Scope the routes to the exact external destinations required; otherwise internal traffic will be hijacked.

---

These six stories capture the diagnostic habits that mattered most across the project: verify the live state before trusting assumptions, treat “it should work” as a hypothesis rather than a fact, and when a setting keeps reverting, look for the tool that is re-applying it.
