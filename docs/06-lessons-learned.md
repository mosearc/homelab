# Lessons learned & debugging notes

The parts that didn't go to plan — and what fixed them. These are the most interesting bits of the build.

## Joining a cluster overwrites a node's storage config

**Symptom:** creating a container on the ext4 node, the disk dropdown offered only `local-zfs` showing **0 B / 0 B**, and `pvesm status` threw `cannot open 'rpool': no such pool`.

**Cause:** joining the cluster replaced the node's `/etc/pve` storage config with the cluster's (authored by a ZFS node). The ext4 node lost its own `local-lvm` definition and inherited `local-zfs`, which points at an `rpool` that doesn't exist on that machine.

**Fix:** the install's directory storage `local` was still active with real free space — enabled **Content → Container** on it (Datacenter → Storage → local → Edit) and used that. Tidy-up: scope the `local-zfs` storage to only the nodes that actually have a ZFS pool, so the phantom stops erroring.

**Takeaway:** in a mixed-storage cluster, storage definitions are cluster-wide; a storage can *appear* on a node that can't back it. Always confirm with `pvesm status` on the node itself.

## IP conflict from reusing a placeholder

Two containers briefly ended up on the same address because a suggested placeholder IP got reused. Showed up as flaky "works sometimes" behaviour. Fixed by giving each a distinct static IP (mirroring CT IDs) while nothing yet pointed at them. **Takeaway:** assign and track static IPs deliberately; port-forwards and records must target the container's *actual* IP, not its CT ID.

## wg-easy v15 is a rewrite — old guides mislead

Lost time to guides written for v13/v14. In v15: no `WG_HOST`/`PASSWORD_HASH` env vars (setup is a web wizard), `INSECURE=true` needed for plain-HTTP UI, a `/lib/modules` mount required, and image pinned to a major tag because the on-disk format changes across majors. **Takeaway:** for fast-moving self-hosted projects, check the *current* version's docs, not the top search result.

## WireGuard in an unprivileged LXC

`wg-quick` needs `/dev/net/tun`, which an unprivileged container doesn't get by default. Fixed with two `lxc.cgroup2.devices.allow` / `lxc.mount.entry` lines in the container config (cgroup2, not the older cgroup form). Bonus: since `/etc/pve` is cluster-synced, the device access and the host-loaded kernel module both survive an HA failover with no per-node work.

## `resolvenf: command not found` on the Debian client

`wg-quick up` created the interface then tore it down because the config's `DNS =` line is applied via `resolvconf`, absent on the box. Fixed with `apt install openresolv` (or drop the `DNS =` line). **Takeaway:** the tunnel actually came up fine — the failure was the last post-up step, not WireGuard.

## "Is it powered?" isn't a thing on desktops

Wanted to see AC vs battery per node; on desktops there's no battery sensor — `/sys/class/power_supply/` has no `BAT*`, and `upower` reports nothing. For desktops, "powered" == "running" == the node status dot / `pvecm status`. The right tool for "did a node drop?" is Proxmox email alerts, not a power sensor. (Battery state via `upower` only exists on laptop-class nodes.)

## HA "not failing back" is the feature, not a bug

After a node recovered, the VPN and portfolio stayed on the fallback node. That's **Failback disabled** working as designed — deliberate on flaky hardware. Move them back manually with a fast delta migration once the primary is confirmed healthy.
