# Security monitoring stack (planned)

Goal: visibility into what's happening on the cluster — host/endpoint auditing and network intrusion detection. This is a project in its own right; it's scoped here before building so the hardware limits drive the design, not the other way round.

## Wazuh — host/endpoint monitoring

Wazuh has two halves:

- **Server** (indexer + manager + dashboard) — **heavy**. The indexer is OpenSearch; realistically 4 GB RAM minimum, 8 GB comfortable. On this cluster the only viable home is **pvedell (16 GB)**, where it will be the single biggest consumer. It must be sized honestly or it OOMs and takes neighbours down.
- **Agents** — lightweight, installed on the things being monitored. The highest-value targets here are the **Proxmox hosts themselves** (file-integrity monitoring, auth/login auditing, rootkit detection on the hypervisors), plus the internet-facing containers.

## Network IDS — Suricata (leaning) vs Snort

An IDS only sees traffic that physically reaches it — dropped into a normal bridged container it sees only its own traffic. Making it useful requires deliberately feeding it traffic, which is a design choice:

- **Per-container visibility** (e.g. the internet-facing portfolio) — feasible by mirroring that container's path.
- **True edge IDS** — ideally sits where all traffic crosses, which here is the TIM router — and that box won't run an IDS or mirror ports. Real edge inspection would mean routing WAN through an **OPNsense/pfSense VM** and running the IDS there: a much larger rearchitecture.

**Suricata over Snort:** multithreaded, same rule syntax, and a cleaner documented pipeline into Wazuh. If the goal is "IDS alerts flowing into Wazuh," Suricata is the path of less resistance.

## Open decision

Two distinct projects hide under "security stack":

1. **Host/endpoint monitoring** (Wazuh server on pvedell + agents on hosts/containers) — very achievable on this hardware.
2. **Network IDS at the edge** (OPNsense/pfSense router VM + Suricata) — a real undertaking that changes the network topology.

Starting with (1) delivers most of the value for the least risk; (2) is a later, deliberate step.
