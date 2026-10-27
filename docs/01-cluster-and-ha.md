# Cluster & High Availability

## Why a 3-node cluster

Three nodes give a proper **quorum majority** — the cluster stays operational with one node down, and the weakest machine earns its place just by being the tiebreaker (a 2-node cluster has no clean majority and the survivor goes read-only). Management is unified in one web UI, and guests can migrate between nodes.

What I deliberately **did not** build: a Ceph + HA-storage cluster. Ceph needs RAM the 4 GB node doesn't have and a fast storage network this setup lacks. None of these services need sub-minute automatic recovery anyway.

## Install choices that mattered

- **Proxmox VE 9.2** on all three nodes.
- **Filesystem per node:** ZFS on the two capable machines (enables replication + snapshots), **ext4/LVM on the 4 GB node** (ZFS ARC would only steal RAM from the workload).
- Hostnames and static IPs locked in **before** clustering — both are painful to change once a node has joined.
- `/etc/hosts` on every node lists all three so they resolve each other by name regardless of DNS.

## Clustering

Created the cluster on the primary (`pvedell`), then joined the others **one at a time** (a joining node must be empty of guests). Expect the joining node's web UI to momentarily break as it adopts the cluster's certificates — hard-refresh and log back in. Verified with `pvecm status`: three nodes, `Quorate: Yes`, total votes 3 / quorum 2.

> **Gotcha discovered later:** joining a cluster overwrites the node's storage config with the cluster's. A node installed with ext4/LVM lost its `local-lvm` definition and inherited a phantom `local-zfs` pointing at a non-existent `rpool`. See [`06-lessons-learned.md`](06-lessons-learned.md).

## High availability without shared storage

HA here rides on **ZFS replication**, applied only to the light, near-stateless services (VPN + portfolio):

1. **Replication job** per guest, primary → second ZFS node, every 15 minutes. First run is a full send; subsequent runs ship only the snapshot delta.
2. **HA resource** per guest (`started` state), with **Failback unchecked**.
3. **Node affinity rule** (PVE 9 replaced the old "HA Groups") giving the primary higher priority and the second ZFS node as fallback, non-strict.

On a node failure the guest is fenced and restarted on the second node from the last replica — losing at most ~15 min of (for these services, nonexistent) state changes.

### Why failback is off

With failback on, a guest chases its preferred node back the instant it reappears. On aging desktops a node can recover then wobble, producing a flapping loop of brief outages. Failback off means the guest settles on the fallback node and waits for a **manual** migration once I've confirmed the primary is truly healthy. Trade-off: I move things back by hand (right-click → Migrate; it's a fast delta migration since a replica already exists). LXCs can't live-migrate, so expect a few seconds' downtime — irrelevant for a portfolio or VPN.

## Keeping laptop-class nodes awake

For any node that's a laptop, `/etc/systemd/logind.conf` is set so a closed lid doesn't suspend it:

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```
(then `systemctl restart systemd-logind`; firmware lid settings may also need disabling.)
