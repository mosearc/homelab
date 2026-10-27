# Portfolio hosting

The public site at [mosearc.eu](https://mosearc.eu), served from **CT 101** (on the primary, HA-replicated to the second ZFS node). Static site, deployed by CI/CD, fully self-hosted.

## Exposure: direct, with Caddy + Let's Encrypt

Chosen over a Cloudflare tunnel/proxy on purpose — see [`02-networking-dns.md`](02-networking-dns.md) for the reasoning (self-hosted end to end, and edge traffic stays inspectable for a future IDS/IPS).

- `mosearc.eu` / `www.mosearc.eu` are **grey-cloud** A records, kept on the home IP by the DDNS updater.
- Router forwards **TCP 80 and 443** → `10.77.0.101`.
- **Caddy** serves the static files and obtains/renews a **Let's Encrypt** certificate automatically — no certbot, no manual renewal. HTTPS is terminated on the box, by us.

Caddyfile is dead simple — naming the domains is what triggers automatic TLS:

```
{
    email <account-email>
}
mosearc.eu, www.mosearc.eu {
    root * /var/www/portfolio
    file_server
    encode gzip zstd
}
```

Fallback if a residential ISP blocks inbound 80/443: switch Caddy to the ACME **DNS-01** challenge via the Cloudflare token (certs with no inbound ports). Not needed here — 80/443 were open.

## CI/CD: self-hosted GitHub Actions runner

A dynamic IP with no inbound access can't be *pushed to*, so deployment is **pull-based**: a self-hosted runner inside CT 101 pulls jobs from GitHub and deploys locally.

```
git push origin main
   → runner (in CT 101) checks out the repo
   → rsyncs public/ into /var/www/portfolio  (--delete = exact mirror)
   → Caddy serves it immediately
```

- Runner runs as a non-root `deploy` user that owns `/var/www/portfolio` (installed with `./svc.sh install deploy` — root installs the systemd unit, the runner process itself runs as `deploy`).
- Workflow triggers on **push to `main` only** (never on pull requests) — defense in depth, since the runner executes repo code inside the home network.
- **The site repo is private** for the same reason; the public showcase is this separate `homelab` repo. (The site repo is `mosearc.github.io` — going private disables free GitHub Pages at that URL, which is fine since the site is self-hosted at `mosearc.eu`.)

Workflow and Caddyfile live in the site repo (not committed here).

## Status

- [x] Container created, HA-replicated
- [x] Caddy serving static files
- [x] DNS (grey) + port-forward 80/443
- [x] Let's Encrypt TLS, live at `https://mosearc.eu`
- [x] Self-hosted runner registered + running as `deploy`
- [x] Deploy workflow (`push` → runner → `rsync` to web root)
- [x] Live — real site deployed from the existing `mosearc.github.io` source on every push to `main`
