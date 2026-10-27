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

## Exposure strategy: direct, self-hosted

Both public services resolve (via Cloudflare DNS) straight to the home IP as **grey-cloud** records — nothing is proxied, so traffic reaches the cluster directly:

- **Portfolio → grey-cloud A record + port-forward 80/443.** Caddy in CT 101 obtains its own Let's Encrypt certificate. Chosen over a Cloudflare tunnel/proxy deliberately: it keeps the path fully self-hosted and leaves inbound web traffic inspectable at the edge for a future IDS/IPS (a tunnel would encapsulate it). Trade-off accepted: the home IP is public.
- **VPN → grey-cloud A record + port-forward UDP 51820.** WireGuard is UDP and couldn't be proxied anyway, so `vpn.mosearc.eu` resolves straight to home.

## Dynamic DNS

A hardened [`cloudflare-ddns`](../configs/cloudflare-ddns/) container keeps `mosearc.eu`, `www.mosearc.eu`, and `vpn.mosearc.eu` current as the IP changes (every 5 min), using a **scoped API token** (Edit zone DNS, `mosearc.eu` only). `PROXIED=false` keeps all three records grey and the updater re-enforces that each run. One updater can mix proxied/unproxied later via a `PROXIED` boolean expression.

On this **IPv4-only** network the default IP-detection method (`cloudflare.trace`) failed: it resolves a hostname that returns AAAA records, then times out trying to reach it over absent IPv6. Fixed with `IP4_PROVIDER=cloudflare.doh` (talks to `1.1.1.1` by raw IP) and `IP6_PROVIDER=none`. See [`06-lessons-learned.md`](06-lessons-learned.md).

## DNS

Clients currently resolve through the router / public resolvers; there is no
internal filtering resolver in the cluster at the moment. A filtering + local-name
resolver (AdGuard Home or Pi-hole on the `pveacer` node) is a candidate future
addition — the trade-off to weigh there is that it would be a network-wide single
point of failure on the non-HA ext4 node, so it would need a fallback resolver in
DHCP and the Proxmox hosts kept on a separate public resolver.
