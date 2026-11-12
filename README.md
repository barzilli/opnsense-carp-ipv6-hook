# OPNsense CARP IPv6 hook

`80-wancarp` is a CARP syshook for an OPNsense HA pair that has a single upstream DHCPv6-PD WAN.

- **MASTER:** brings WAN up, switches LAN IPv6 to track WAN (identity association), and disables the floating LAN IPv6 gateway.
- **BACKUP / INIT:** takes WAN down, switches LAN IPv6 to SLAAC, and enables the floating LAN IPv6 gateway, so the default route goes via the master.

## Install

Copy the script to `/usr/local/etc/rc.syshook.d/carp/80-wancarp` on both nodes and run `chmod +x` on it. Check the settings block at the top first (interface keys and device names, and the floating gateway name).

## Firewall prerequisites

The hook expects the pair to be set up as follows. Items 1 to 5 come from the tested deployment. Items 6 to 9 follow from what the script itself does.

1. **Same WAN MAC address on both nodes.** Set the WAN interface (`vtnet1`) to one MAC address on both firewalls. The hook takes WAN down on the BACKUP, so that MAC address is never live on both nodes at once. The upstream gateway then sees a single client, with the same link-local address and the same delegated prefix, whichever node is MASTER. This is safe only because WAN is down on the BACKUP; it is not a general HA recommendation.
2. **Same DHCP Unique Identifier (DUID) on both nodes.** dhcp6c identifies itself to the upstream gateway by DUID, so with the same DUID (and the same MAC address, item 1) each node is seen as the same client and is delegated the same prefix after a failover.
3. **DHCPv6 prefix IAID set to 2** (WAN, DHCPv6 options). This was added when earlier versions of the hook saw the IPv6 prefix change across failovers. It may not be needed with the current version, but that hasn't been tested yet.
4. **Floating IPv4 default gateway `LAN_4` on each node:** priority 255, description "Floating Static Route", pointing to the other node's internal static IPv4 address. While a node is BACKUP and its WAN is down, this gives it an IPv4 default route via its peer.
5. **Floating IPv6 default gateway `LAN_6` on each node** (the `LAN_GW6` setting): Interface **LAN** (not WAN, which is down on the BACKUP, so a link-local gateway there never installs), priority 253, Upstream Gateway ticked, routing via the other node's LAN IPv6 link-local address. It needs priority 253 to win over the dynamic `LAN_SLAAC` gateway (priority 254) that OPNsense creates from the master's RA (via `fe80::c0a8:1`) while LAN is in SLAAC mode. It gives the BACKUP node an IPv6 default route via its peer. The hook enables it on BACKUP and disables it on MASTER. Each node's link-local address comes from its LAN MAC address, so it changes if that NIC's MAC changes. Required, and it must point at the peer's own link-local, not the CARP VIP. In testing without it, the BACKUP learned the master's RA, whose source is the LAN CARP link-local VIP (`fe80::c0a8:1`). That address is also configured on the BACKUP's own vtnet0 (vhid 1, inactive), so the RA-learned default route pointed at a local address and IPv6 egress failed with "No route to host".
6. **A CARP VIP on LAN** (`LAN_IF`). The hook ignores CARP events from every other interface.
7. **WAN IPv6 set to DHCPv6**, enabled on the MASTER and disabled on the BACKUP. The hook switches it at each transition.
8. **LAN IPv6 set to track WAN (identity association) on the MASTER, and to SLAAC on the BACKUP.** The hook switches it at each transition.
9. **The settings block matches each node:** `WAN_KEY`, `WAN_IF`, `LAN_KEY`, `LAN_IF`, `LAN_GW6`.

## Tuning `WAN_UP_DELAY_US`

Both nodes share one WAN MAC address, so the new MASTER must not bring vtnet1 up until the old MASTER's vtnet1 is down. The delay can't guarantee that, because it can't see the other node. It can only cover a gap you have measured. The BACKUP branch therefore takes vtnet1 down first, before the slower interface reset and reconfigure, and both transitions are logged with microsecond timestamps:

```
wancarp: vtnet1 down at 1762930000.123456   (old master)
wancarp: vtnet1 up at 1762930000.734567     (new master)
```

To tune it:

1. Keep both nodes NTP-synced.
2. Run about 20 graceful failovers each way (CARP maintenance mode, a clean VM shutdown or reboot, and pulling the LAN link). A crash needs no tuning, because the old master's WAN is already gone.
3. For each one, compute the margin as `up - down` from `grep wancarp /var/log/system/latest.log` on both nodes.
4. The smallest margin you saw is slack. You can lower `WAN_UP_DELAY_US` by up to that much, but keep a safety margin of at least 100 ms for clock skew and load. If any margin is negative, the links overlapped, so raise the delay instead.

## Other ISPs

The hook has only been tested with AT&T fiber, which delegates /64s out of a /60. The script itself doesn't depend on the ISP: it only switches WAN, LAN tracking and the floating gateways. What changes between ISPs is how their DHCPv6 server reacts when a second router takes over the same lease. Before relying on it with another ISP, check these, which are general guidance and not yet tested:

- **MAC locking.** Cable ISPs such as Spectrum usually have the modem learn the first MAC address it sees and serve only that until the modem reboots. That makes the shared WAN MAC (item 1) essential, not just helpful.
- **Lease binding.** Most ISPs tie the delegated prefix to the DUID and IAID, so keep both identical on the two nodes (items 2 and 3) to get the same prefix after a failover.
- **Release on WAN down.** If the old master sends a DHCPv6 RELEASE when the hook takes WAN down, some ISPs hand the new master a different prefix. Consider enabling "Do not allow PD/Address release" on WAN.
- **Prefix size.** Set the requested prefix-delegation size to what the ISP actually delegates (Spectrum commonly gives a /64, sometimes a /56 on request). The hook tracks with prefix ID 0, which works for either size.
- **Prefix-only leases.** If the ISP delegates a prefix but no WAN address, tick "Request only an IPv6 prefix". Otherwise OPNsense may not reconfigure the tracking LAN ([opnsense/core#7202](https://github.com/opnsense/core/issues/7202)).

After switching ISPs, run a failover and failback and confirm that LAN keeps the same prefix, that `ndp -p` shows `LO` (no `A`) for it on the master, and that the BACKUP has IPv4 and IPv6 default routes.

## Fix: LAN /64 route lost after failback

While a node is BACKUP, vtnet0 accepts RAs and learns the LAN prefix from the master with the autonomous (A) flag set. When the node returns to MASTER, the hook used to turn `accept_rtadv` off but left that RA-learned prefix in place. The address that dhcp6c then added reused it, so the A flag stuck. FreeBSD then detaches A-flagged prefixes that have no reachable advertising router as soon as the WAN router is seen as reachable ([nd6_rtr.c](https://github.com/freebsd/freebsd-src/blob/stable/14/sys/netinet6/nd6_rtr.c#L1846-L1855)). That removed the LAN /64 route, and traffic for LAN hosts looped between the firewall and the upstream gateway.

On the switch back to tracking, `flush_slaac_state()` now removes vtnet0's `autoconf` addresses and flushes the prefix and default-router lists (`ndp -P`, `ndp -R`). This runs only while WAN is still down, so the flushes hit nothing but the stale LAN entries.

After deploying, reapply LAN once to clear any prefix that still has the A flag set, then check that `ndp -p` shows `LO` (no `A`) for the LAN prefix after a failover and failback.
