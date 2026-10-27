# WireGuard VPN (wg-easy)

Self-hosted remote access to the cluster and LAN from anywhere, reached at `vpn.mosearc.eu`. Built on [wg-easy](https://github.com/wg-easy/wg-easy) **v15** for its web admin UI and QR-code client onboarding, running as Docker-in-LXC in **CT 100** (on the primary, HA-replicated to the second ZFS node).

Compose file: [`../configs/wg-easy/docker-compose.yml`](../configs/wg-easy/docker-compose.yml).

## Container setup

- **CT 100**, unprivileged, with the `nesting=1,keyctl=1` features (Docker-in-LXC needs both), static IP `10.77.0.100`.
- **TUN device access** added to `/etc/pve/lxc/100.conf` — see [`../configs/lxc/100-vpn.conf.example`](../configs/lxc/100-vpn.conf.example). Because `/etc/pve` is cluster-synced, this travels to the failover node automatically.
- The in-kernel `wireguard` module is loaded on **both** ZFS hosts (`/etc/modules-load.d`) so the tunnel works wherever the container lands.

## wg-easy v15 specifics (these tripped me up)

v15 is a rewrite — most older guides are wrong for it:

- The admin account and the public **host** (the endpoint written into every client config) are set in a **first-run web wizard**, not via `WG_HOST`/`PASSWORD_HASH` env vars (which no longer exist).
- `INSECURE=true` is required for the admin UI over plain HTTP — kept **LAN-only**; port 51821 is never forwarded.
- A `/lib/modules:ro` mount is required so the container can use the host's kernel WireGuard.
- The public **host must be `vpn.mosearc.eu`**, not the container's LAN IP — the LAN IP is unreachable from outside, and DDNS is what keeps the domain pointed home.

## Routing & exposure

- Router forwards **UDP 51820 → 10.77.0.100**. (The admin UI stays off the internet.)
- `vpn.mosearc.eu` is a **grey-cloud** Cloudflare record (UDP can't be proxied), kept current by the DDNS container.

## Clients

- **Phone:** add a client in the UI, scan the QR, done.
- **Debian laptop:** `apt install wireguard`, drop the config at `/etc/wireguard/wg0.conf`, `wg-quick up wg0`.
  - Hit the classic `resolvconf: command not found` on bring-up — fixed with `apt install openresolv` (the config carries a `DNS =` line that wg-quick applies via resolvconf).
  - For a GUI toggle: `nmcli connection import type wireguard file /etc/wireguard/wg0.conf` puts it in the NetworkManager applet. Don't let both NetworkManager and `wg-quick`/systemd manage the same interface.
- **Client DNS** is set per-client in the wg-easy admin panel (in v15 this is in the UI, and existing clients must be **re-issued** to pick up a change). Currently points at a public resolver; if an internal filtering resolver is added later, this is where VPN clients would be pointed at it.

## Full vs split tunnel

Set per-client via **AllowedIPs**: `0.0.0.0/0` routes everything through home (good on untrusted Wi-Fi); `10.77.0.0/24` only routes cluster/LAN traffic (snappier for trusted networks). Either reaches the cluster.

Once connected, the whole cluster is reachable as if local, e.g. `https://pvedell.lan:8006`.
