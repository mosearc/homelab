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
| AdGuard Home | 102 | `10.77.0.102` |

These sit outside the router's DHCP pool so nothing else is handed the same address.

## Exposure strategy: outbound-friendly by design

Two public services, two different records on Cloudflare — and the difference is the whole point:

- **Portfolio → proxied (orange cloud).** Public DNS points at Cloudflare's stable IPs; Cloudflare forwards to home. Free HTTPS, home IP hidden, **no inbound ports opened** on the router.
- **VPN → DNS only (grey cloud).** WireGuard is UDP and can't traverse Cloudflare's HTTP proxy, so `vpn.mosearc.eu` resolves straight to the home IP, and UDP 51820 is port-forwarded on the router.

## Dynamic DNS

A hardened [`cloudflare-ddns`](../configs/cloudflare-ddns/) container keeps `vpn.mosearc.eu` current as the IP changes (every 5 min), using a **scoped API token** (Edit zone DNS, `mosearc.eu` only). `PROXIED=false` keeps the record grey and the updater re-enforces that each run. A low record TTL shrinks the propagation gap after an IP change.

## AdGuard Home — filtering DNS + local names

Full config notes in [`../configs/adguard/README.md`](../configs/adguard/README.md). The two things it buys this setup:

- **Network-wide ad/tracker blocking** at the DNS layer — covers every device, including ones that can't run a browser extension (TVs, IoT, mobile apps).
- **Local-name resolution** (`pvedell.lan` → node IP) via DNS rewrites, so the cluster is reachable by name — including over the VPN, where the client DNS is pointed at AdGuard.

It's complementary to a browser ad-blocker, not a replacement: DNS filtering blocks by domain, so same-domain ads (e.g. YouTube) still need uBlock Origin.

## Honest limitation

AdGuard is the one network-wide SPOF — it lives on the non-HA ext4 node. Mitigated with a secondary DHCP resolver (`1.1.1.1`) for fallback, and the Proxmox hosts themselves are kept on a public resolver so a DNS-container outage can never touch the cluster.
