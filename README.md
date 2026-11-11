# OPNsense CARP IPv6 hook

`80-wancarp` is a CARP syshook for an OPNsense HA pair that has a single upstream DHCPv6-PD WAN.

- **MASTER:** brings WAN up, switches LAN IPv6 to track WAN (identity association), and disables the floating LAN IPv6 gateway.
- **BACKUP / INIT:** takes WAN down, switches LAN IPv6 to SLAAC, and enables the floating LAN IPv6 gateway, so the default route goes via the master.

## Install

Copy the script to `/usr/local/etc/rc.syshook.d/carp/80-wancarp` on both nodes and run `chmod +x` on it. Check the settings block at the top first (interface keys and device names, and the floating gateway name).

## Firewall prerequisites

The hook expects the pair to be set up as follows. Items 1 to 4 come from the tested deployment. Items 5 to 8 follow from what the script itself does.

1. **Same WAN MAC address on both nodes.** Set the WAN interface (`vtnet1`) to one MAC address on both firewalls. The hook takes WAN down on the BACKUP, so that MAC address is never live on both nodes at once. The upstream gateway then sees a single client, with the same link-local address and the same delegated prefix, whichever node is MASTER. This is safe only because WAN is down on the BACKUP; it is not a general HA recommendation.
2. **DHCPv6 prefix IAID set to 2** (WAN, DHCPv6 options). This was added when earlier versions of the hook saw the IPv6 prefix change across failovers. It may not be needed with the current version, but that hasn't been tested yet.
3. **Floating IPv4 default gateway `LAN_4` on each node:** priority 255, description "Floating Static Route", pointing to the other node's internal static IPv4 address. While a node is BACKUP and its WAN is down, this gives it an IPv4 default route via its peer.
4. **Floating IPv6 default gateway `LAN_6` on each node** (the `LAN_GW6` setting), routing via the other node's LAN IPv6 link-local address. It gives the BACKUP node an IPv6 default route via its peer. The hook enables it on BACKUP and disables it on MASTER. Each node's link-local address comes from its LAN MAC address, so it changes if that NIC's MAC changes. This may be redundant: OPNsense sets `net.inet6.ip6.rfc6204w3=1`, so while it is BACKUP (LAN on SLAAC, accepting RAs) the node should learn the master's RA as its default router. That hasn't been tested without `LAN_6` yet, and the `route flush -inet6` in the BACKUP branch may remove the RA-learned route.
5. **A CARP VIP on LAN** (`LAN_IF`). The hook ignores CARP events from every other interface.
6. **WAN IPv6 set to DHCPv6**, enabled on the MASTER and disabled on the BACKUP. The hook switches it at each transition.
7. **LAN IPv6 set to track WAN (identity association) on the MASTER, and to SLAAC on the BACKUP.** The hook switches it at each transition.
8. **The settings block matches each node:** `WAN_KEY`, `WAN_IF`, `LAN_KEY`, `LAN_IF`, `LAN_GW6`.

## Fix: LAN /64 route lost after failback

While a node is BACKUP, vtnet0 accepts RAs and learns the LAN prefix from the master with the autonomous (A) flag set. When the node returns to MASTER, the hook used to turn `accept_rtadv` off but left that RA-learned prefix in place. The address that dhcp6c then added reused it, so the A flag stuck. FreeBSD then detaches A-flagged prefixes that have no reachable advertising router as soon as the WAN router is seen as reachable ([nd6_rtr.c](https://github.com/freebsd/freebsd-src/blob/stable/14/sys/netinet6/nd6_rtr.c#L1846-L1855)). That removed the LAN /64 route, and traffic for LAN hosts looped between the firewall and the upstream gateway.

On the switch back to tracking, `flush_slaac_state()` now removes vtnet0's `autoconf` addresses and flushes the prefix and default-router lists (`ndp -P`, `ndp -R`). This runs only while WAN is still down, so the flushes hit nothing but the stale LAN entries.

After deploying, reapply LAN once to clear any prefix that still has the A flag set, then check that `ndp -p` shows `LO` (no `A`) for the LAN prefix after a failover and failback.
