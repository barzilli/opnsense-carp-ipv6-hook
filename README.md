# OPNsense CARP IPv6 hook

`80-wancarp` is a CARP syshook for an OPNsense HA pair that has a single upstream DHCPv6-PD WAN.

- **MASTER:** brings WAN up, switches LAN IPv6 to track WAN (identity association), and disables the floating LAN IPv6 gateway.
- **BACKUP / INIT:** takes WAN down, switches LAN IPv6 to SLAAC, and enables the floating LAN IPv6 gateway, so the default route goes via the master.

## Install

Copy the script to `/usr/local/etc/rc.syshook.d/carp/80-wancarp` on both nodes and run `chmod +x` on it. Check the settings block at the top first (interface keys and device names, and the floating gateway name).

## Fix: LAN /64 route lost after failback

While a node is BACKUP, vtnet0 accepts RAs and learns the LAN prefix from the master with the autonomous (A) flag set. When the node returns to MASTER, the hook used to turn `accept_rtadv` off but left that RA-learned prefix in place. The address that dhcp6c then added reused it, so the A flag stuck. FreeBSD then detaches A-flagged prefixes that have no reachable advertising router as soon as the WAN router is seen as reachable ([nd6_rtr.c](https://github.com/freebsd/freebsd-src/blob/stable/14/sys/netinet6/nd6_rtr.c#L1846-L1855)). That removed the LAN /64 route, and traffic for LAN hosts looped between the firewall and the upstream gateway.

On the switch back to tracking, `flush_slaac_state()` now removes vtnet0's `autoconf` addresses and flushes the prefix and default-router lists (`ndp -P`, `ndp -R`). This runs only while WAN is still down, so the flushes hit nothing but the stale LAN entries.

After deploying, reapply LAN once to clear any prefix that still has the A flag set, then check that `ndp -p` shows `LO` (no `A`) for the LAN prefix after a failover and failback.
