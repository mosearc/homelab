# AdGuard Home — configuration notes

AdGuard's real config file (`AdGuardHome.yaml`) holds the admin **password hash**
and client details, so it is **not committed** (see repo `.gitignore`). These are
the settings that matter, recorded so the setup is reproducible.

Installed natively (no Docker) in **CT 102** on `pveacer` via the official script:

```
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

- **Container:** CT 102, unprivileged, static IP `10.77.0.102/24`, gateway `10.77.0.1`.
  Storage is `local` (ext4 node — no ZFS, so AdGuard is **not** HA. See the SPOF note below.)
- **Admin UI:** port 80 · **DNS:** port 53 (both on all interfaces).

## Upstream DNS (encrypted, parallel)

```
https://dns.cloudflare.com/dns-query
https://dns.quad9.net/dns-query
```

- Mode: **Parallel requests**
- **Bootstrap DNS:** `1.1.1.1`, `9.9.9.9` (needed to resolve the DoH hostnames)
- DNSSEC: enabled · Blocking mode: Default

## Local-name resolution (Filters -> DNS rewrites)

Lets every device (incl. VPN clients) reach the cluster by name instead of IP:

```
pvedell.lan  ->  10.77.0.11      # adjust to real node IPs
pveasus.lan  ->  10.77.0.12
pveacer.lan  ->  10.77.0.13
```
<!-- VERIFY node management IPs. -->

## Making the network use it

- **Router DHCP DNS:** primary `10.77.0.102`, secondary `1.1.1.1` (fallback).
  *The TIM router's DHCP-advertised DNS field is behind "show advanced"; if it
  can't be set, the fallback is to run AdGuard's own DHCP server and disable the
  router's.*
- **wg-easy client DNS:** set to `10.77.0.102` so VPN clients filter + resolve
  `.lan` names while away (set per-client in the wg-easy admin panel; existing
  clients must be re-issued).

## Single point of failure (accepted, mitigated)

AdGuard runs on the non-HA ext4 node. If it goes down, name resolution for the
network goes with it. Mitigations: the secondary DHCP resolver (`1.1.1.1`) keeps
devices resolving (unfiltered); a second AdGuard instance with config sync is a
future option. The Proxmox **hosts themselves are intentionally left on a public
resolver**, so a DNS-container hiccup can never affect the cluster.
