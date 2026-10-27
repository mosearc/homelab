# Portfolio hosting

The public site at [mosearc.eu](https://mosearc.eu), served from **CT 101** (on the primary, HA-replicated to the second ZFS node).

## Exposure: Cloudflare proxy, zero open ports

Given a dynamic IP behind a locked-down ISP router, the site is exposed through Cloudflare's **proxied (orange-cloud)** record rather than port-forwarding:

- Public DNS points at Cloudflare's stable IPs → Cloudflare forwards to the home origin.
- **No inbound ports** opened on the router for the web side.
- Free, auto-renewing **HTTPS** at the edge; the home IP stays hidden.
- DDNS keeps the origin reachable as the IP changes.

Two viable ways to connect Cloudflare to the origin:

1. **Cloudflare Tunnel** (`cloudflared` in the container) — dials *out* to Cloudflare, so nothing at all is opened on the router. Cleanest, and sidesteps the ISP router entirely. **Preferred.**
2. **Proxied A record + forward 80/443** — traditional; works, but opens two ports on the router.

## Web server

<!-- TODO: fill in once the site stack is finalised. -->
A lightweight server in CT 101 (Caddy or nginx for a static build; a Node/other runtime proxied if the site needs one). The container keeps a **static LAN IP** so that, if the non-tunnel option is ever used, the router's forward survives an HA failover.

## Status

- [x] Container created, HA-replicated
- [ ] Web server + site content deployed
- [ ] Cloudflare Tunnel wired to `mosearc.eu`
- [ ] HTTPS verified end-to-end
