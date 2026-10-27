# Networking & DNS

## The constraints

- **Dynamic public IP** (IPv4 only — IPv6 disabled at the ISP router).
- **TIM Technicolor router**, consumer-grade and locked down: limited DHCP/DNS controls hidden behind an "advanced" toggle, no port-mirroring, no custom firmware.
- LAN: `10.77.0.0/24`, gateway `10.77.0.1`.

Early on I suspected CGNAT (which would rule out port-forwarding), but testing confirmed a real, routable — if dynamic — public IP. That single fact shaped the whole exposure strategy.

## Address plan

Containers use **static IPs set in Proxmox** (not DHCP reservations), chosen to mirror their CT IDs where possible:

| Service | CT | IP |
|---------|----|----|
| WireGuard (wg-easy) | 100 | `10.77.0.100` |
| Portfolio | 101 | `10.77.0.101` |

These sit outside the router's DHCP pool so nothing else is handed the same address.

## Exposure strategy: outbound-friendly by design

Two public services, two different records on Cloudflare — and the difference is the whole point:

- **Portfolio → proxied (orange cloud).** Public DNS points at Cloudflare's stable IPs; Cloudflare forwards to home. Free HTTPS, home IP hidden, **no inbound ports opened** on the router.
- **VPN → DNS only (grey cloud).** WireGuard is UDP and can't traverse Cloudflare's HTTP proxy, so `vpn.mosearc.eu` resolves straight to the home IP, and UDP 51820 is port-forwarded on the router.

## Dynamic DNS

A hardened [`cloudflare-ddns`](../configs/cloudflare-ddns/) container keeps `vpn.mosearc.eu` current as the IP changes (every 5 min), using a **scoped API token** (Edit zone DNS, `mosearc.eu` only). `PROXIED=false` keeps the record grey and the updater re-enforces that each run. A low record TTL shrinks the propagation gap after an IP change.

## DNS

Clients currently resolve through the router / public resolvers; there is no
internal filtering resolver in the cluster at the moment. A filtering + local-name
resolver (AdGuard Home or Pi-hole on the `pveacer` node) is a candidate future
addition — the trade-off to weigh there is that it would be a network-wide single
point of failure on the non-HA ext4 node, so it would need a fallback resolver in
DHCP and the Proxmox hosts kept on a separate public resolver.
