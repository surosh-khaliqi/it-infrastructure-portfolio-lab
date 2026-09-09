# Notable Issues

This document covers six troubleshooting cases from across the four project phases. They were selected because the problems were not obvious at first, affected important parts of the lab, or required several steps to identify and fix.

Each case includes the symptom, diagnosis path, root cause, fix, and the main lesson from the troubleshooting process.

---

### 1 — The recurring `ip_forward` resets

**Where:** RTR-SITE1 and RTR-SITE2, across Phases 1–3

**Symptom:** Inter-VLAN or cross-site TCP traffic would suddenly fail while ICMP continued to work. The failure often appeared after a reboot, a package install, or a `ufw reload`.

**Diagnosis path:**  
In Phase 1, the symptom first appeared during validation. Checking `sysctl net.ipv4.ip_forward` returned `0` even though the setting had been written earlier. Temporary fixes restored forwarding, but the same reset reappeared in Phase 2 after router work and again in Phase 3 after Linux agent and UFW changes. Each time, a temporary sysctl setting restored service until the next UFW operation set the value back to `0`.

**Root cause:**  
UFW maintains its own sysctl file at `/etc/ufw/sysctl.conf`. On enable or reload, it re-applies the settings in that file. The forwarding line was still commented out:

```ini
#net/ipv4/ip_forward=1
```

Edits made only to `/etc/sysctl.conf` or `/etc/sysctl.d/` were overwritten the next time UFW reloaded.

**Fix:**

```bash
sudo sed -i 's/#net\/ipv4\/ip_forward=1/net\/ipv4\/ip_forward=1/' /etc/ufw/sysctl.conf
sudo ufw reload
sysctl net.ipv4.ip_forward   # remains 1 after reboot
```

Applied on both routers.

**Lesson:** When a kernel setting keeps reverting after a tool that manages sysctl is involved, check that tool’s own configuration instead of only the standard sysctl locations. When forwarded traffic behaves inconsistently, checking `ip_forward` early can rule out a basic router configuration problem.

**Source:** [Phase 1 build log](../phase1-network/02-build-log.md) · [Phase 2 build log](../phase2-servers/02-build-log.md) · [Phase 3 build log](../phase3-monitoring/02-build-log.md)

---

### 2 — DHCP relay dropping replies as “bogus giaddr”

**Where:** RTR-SITE1 and RTR-SITE2, Phase 2

**Symptom:** Clients on the Users VLANs timed out on `ipconfig /renew`. Relay logs showed the request being forwarded, then the reply being discarded with the message `Dropping reply received on eth1 / BOOTREPLY giaddr: ... / Packet to bogus giaddr`.

**Root cause:**  
The default `isc-dhcp-relay` service unit applied a plain `-i` flag to every interface. That caused the relay to treat each interface as client-facing and enforce ownership of the `giaddr`. Legitimate replies arriving on the transit WAN interface failed that check and were dropped.

**Fix:**  
Classify the interfaces explicitly in `/etc/default/isc-dhcp-relay`:

```text
OPTIONS="-id eth0.10 -iu eth0.20 -iu eth1"
```

`-id` marks the client-facing interface, while `-iu` marks upstream or transit interfaces. After the change, DHCP replies flowed correctly on both sites.

**Lesson:** In a multi-hop DHCP relay design, the interface roles matter in both directions. If requests are being forwarded but replies are being dropped, check how the relay classifies the transit interfaces.

**Source:** [Phase 2 build log](../phase2-servers/02-build-log.md)

---

### 3 — Router ACLs broke Site 2 — missing UDP and the DHCP reply path

**Where:** Both routers, Phase 4

**Symptom:** Immediately after the first UFW allow-list and `default deny routed` policy were applied, Site 2 clients lost their DHCP leases and fell back to APIPA. They could also no longer locate the domain controller. Site 1 remained healthy, while ICMP and SSH from the monitoring host still worked.

**Diagnosis path:**  
1. Confirmed both routers were reachable and IP forwarding was still enabled.  
2. Found that the first allow-list contained only TCP rules for the AD-related ports.  
3. Added UDP rules for ports 88, 389, and 464. Domain controller location began working again with `nltest /dsgetdc:ans.local`, but DHCP renewal still failed.  
4. Checked `isc-dhcp-relay` behaviour and confirmed that replies were sent to the relay `giaddr`, which was the client-facing VLAN gateway rather than the WAN address. The original UFW rules did not allow that reply path.

**Root cause:**  
There were two separate gaps in the first rule set:

- required AD traffic was missing UDP rules
- DHCP relay replies were addressed to the VLAN gateway (`giaddr`), not the router WAN address

**Fix:**

```bash
# UDP for AD-related traffic
sudo ufw route allow proto udp from 10.10.11.0/24 to 10.10.12.10 port 88
sudo ufw route allow proto udp from 10.10.21.0/24 to 10.10.12.10 port 88
# same pattern for 389 and 464

# DHCP relay reply path
sudo ufw route allow proto udp from 10.10.12.10 to 10.10.11.1 port 67
sudo ufw route allow proto udp from 10.10.12.10 to 10.10.21.1 port 67
sudo ufw reload
```

After the second set of rules was added, Site 2 clients obtained leases, located the domain controller, and could access file shares again.

**Lesson:** When filtering Active Directory traffic, check whether each required service uses TCP, UDP, or both instead of building the rule set from TCP ports alone. For relay traffic, verify from logs where the reply is actually addressed before finalizing the firewall rules.

**Source:** [Phase 4 build log](../phase4-security/02-build-log.md)

---

### 4 — cloud-init drop-in kept password authentication enabled

**Where:** RTR-SITE1, Phase 4

**Symptom:** After editing `/etc/ssh/sshd_config` to set `PasswordAuthentication no` and restarting SSH, password logins still succeeded.

**Diagnosis path:**  
`sshd -T | grep -i passwordauthentication` still returned `passwordauthentication yes`. The main configuration file had been edited correctly, so another configuration file was overriding it.

**Root cause:**  
Ubuntu cloud-init had created `/etc/ssh/sshd_config.d/50-cloud-init.conf` with `PasswordAuthentication yes`. The active SSH configuration was therefore different from what the main `sshd_config` file showed.

**Fix:**

```bash
sudo sed -i 's/^PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo systemctl restart ssh
sudo sshd -T | grep -i passwordauthentication   # now reports no
```

Final verification confirmed that key-based login succeeded and password login was rejected with `Permission denied (publickey)`.

**Lesson:** After changing SSH authentication settings, verify the effective configuration with `sshd -T` instead of relying only on the main configuration file. On cloud-init systems, the drop-in directory should also be checked.

**Source:** [Phase 4 build log](../phase4-security/02-build-log.md)

---

### 5 — NTFS inheritance granted write access to the ReadOnly group

**Where:** SRV-S1-SERVERS (`C:\TechShare`), Phase 2

**Symptom:** A user who belonged only to `SG-FileShare-ReadOnly` could still create files on the share, even though the share permissions and explicit NTFS entries were intended to allow read-only access.

**Root cause:**  
The folder had inherited ACEs from the parent `C:\` volume. Those inherited entries granted the local Users group Append Data / Write Data rights. Domain users are members of the local Users group by default, so the inherited permissions allowed writes despite the more restrictive explicit entries.

`icacls /remove:g` removes explicit entries, but it does not remove inherited ones.

**Fix:**

```powershell
icacls "C:\TechShare" /inheritance:r
icacls "C:\TechShare" /grant "SYSTEM:(OI)(CI)F"
icacls "C:\TechShare" /grant "BUILTIN\Administrators:(OI)(CI)F"
# explicit SG-FileShare-ReadWrite and SG-FileShare-ReadOnly grants remained
```

After inheritance was removed and the ACL was rebuilt, the ReadOnly group could read but could no longer write.

**Lesson:** When a share needs strict group-based permissions, check inherited parent-folder entries as well as the explicit ACL. Removing an explicit ACE does not remove an inherited permission.

**Source:** [Phase 2 build log](../phase2-servers/02-build-log.md)

---

### 6 — NAT default route on MON-SRV broke internal connectivity

**Where:** MON-SRV, Phase 3

**Symptom:** After a NAT adapter was added for outbound email and ServiceNow API access, the Zabbix GUI became briefly unreachable and several high-severity alerts fired. Internal pings to the Users VLANs failed, while traffic to the local Servers VLAN continued to work.

**Root cause:**  
The netplan configuration for the NAT interface installed a default route with metric `50`. The primary interface’s default route had metric `100`. Traffic without a more-specific route, including internal lab destinations, was therefore sent to the NAT interface and black-holed.

**Fix:**  
Removed the competing default route and replaced it with destination-specific `/32` routes for the external services that actually required internet access:

```bash
sudo ip route del default via 192.168.100.1 dev eth1
sudo ip route add 8.8.8.8/32 via 192.168.100.1 dev eth1
sudo ip route add <servicenow-resolved-ip>/32 via 192.168.100.1 dev eth1
```

Netplan was updated to match. Internal connectivity and monitoring recovered immediately.

**Lesson:** An additional internet-facing interface should not introduce a lower-metric default route that can override internal routing. In this lab, destination-specific routes provided the required external access without changing the path for internal traffic.

**Source:** [Phase 3 build log](../phase3-monitoring/02-build-log.md)

---

These cases are a selected record of the troubleshooting completed during the project. The full phase build logs contain the remaining issues, validation steps, and supporting evidence.

Return to the [project README](../README.md).
