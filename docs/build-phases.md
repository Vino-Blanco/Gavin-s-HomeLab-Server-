# Build Phases

This document tracks each major phase of the homelab build, what was accomplished, and the current state.

---

## Phase 1 — Proxmox Installation
**Status:** ✅ Complete

Installed Proxmox VE 9.1.7 on the HP EliteDesk G4 600. Configured the management interface on vmbr0 (eno1) at 192.168.0.50, disabled enterprise repositories, and added the free community repo. Mounted a 1TB WD HDD at /mnt/hdd and added it as hdd-storage in the Proxmox web UI for ISOs and backups.

**Key details:**
- Boot drive: Samsung 970 Evo NVMe (Proxmox OS + VM disks)
- Storage drive: 1TB WD HDD (ISOs, backups)
- Management access: https://192.168.0.50:8006

---

## Phase 2 — Ubuntu Server VM
**Status:** ✅ Complete

Deployed Ubuntu Server 24.04.4 LTS as VM 100. Configured with 4GB RAM, 2 cores, and a 32GB virtual disk on local-lvm. Originally set up with a static IP on the flat LAN, later migrated to VLAN 20 (Lab) at 10.0.20.20.

**Key details:**
- VM ID: 100
- NIC: net0 on vmbr1, tag=20 (VLAN 20 access port)
- IP: 10.0.20.20/24, gateway 10.0.20.1
- SSH access: `ssh vino@10.0.20.20`
- Netplan config: /etc/netplan/00-installer-config.yaml

---

## Phase 3 — Docker + Portainer
**Status:** ✅ Complete

Installed Docker Engine on Ubuntu Server and deployed Portainer CE as a container management UI. Portainer runs on port 9000 and provides a web interface for managing Docker containers, images, and networks.

**Key details:**
- Portainer UI: http://10.0.20.20:9000
- Docker socket mounted for full container management

---

## Phase 4 — OPNsense Firewall/Router
**Status:** ✅ Complete

Deployed OPNsense 25.1 as VM 101, functioning as the network's firewall, router, DHCP server, and DNS resolver. Configured with three network interfaces: WAN (vmbr0 to Cox router), LAN (vmbr1 VLAN trunk), and OPT1 (vmbr2, unused).

**Key details:**
- VM ID: 101
- WAN: 192.168.0.135 (DHCP from Cox router)
- LAN: VLAN trunk on vmbr1 (`trunks=10;20;30;40`)
- Web UI: http://10.0.10.1 (Trusted VLAN)
- DNS: Unbound with DNSSEC enabled
- DHCP: Configured on VLANs 10, 20, 30, 40 (range .100-.200 each)

---

## Phase 5 — VLAN Segmentation
**Status:** ✅ Complete

Configured 802.1Q VLAN segmentation across OPNsense, the Netgear JGS516PE managed switch, and Proxmox's virtual bridge. Created four active VLANs: Trusted (10), Lab (20), IoT (30), and Guest (40). Enabled VLAN-aware mode on Proxmox's vmbr1 bridge with proper trunk and access port assignments.

**Key details:**
- Switch Port 1: Trunk — Tagged on VLANs 10, 20, 30, 40; Untagged on VLAN 1
- Switch Port 2: Access — Untagged on VLAN 10, PVID 10 (Gaming PC)
- OPNsense: Creates VLAN sub-interfaces (vlan01-04) on vtnet1 for routing
- Proxmox vmbr1: VLAN-aware bridge with `bridge-vids 1-4094` and `bridge-pvid 1`
- Ubuntu Server VM: Proxmox tag=20 (access port on VLAN 20)
- OPNsense VM: Proxmox trunks=10;20;30;40 (trunk port)
- Inter-VLAN routing: Verified — all VLANs can reach each other through OPNsense

---

## Phase 6 — Firewall Lockdown
**Status:** ✅ Complete

Replaced pass-all firewall rules with restrictive per-VLAN policies. Each VLAN now has explicit allow/deny rules controlling inter-VLAN access and internet connectivity.

**Policy summary:**
- **Trusted (VLAN 10):** Full access to all VLANs and internet
- **Lab (VLAN 20):** Internet access, access to Trusted and OPNsense management, blocked from IoT and Guest
- **IoT (VLAN 30):** Internet only — blocked from all other VLANs and OPNsense
- **Guest (VLAN 40):** Internet only — fully isolated from all other VLANs and OPNsense

**Verification:** Tested inter-VLAN connectivity from Lab VLAN using ping to confirm blocks are enforced.

---

## Phase 7 — Monitoring Stack
**Status:** ✅ Complete

Deployed a full infrastructure monitoring stack using Docker Compose at /opt/monitoring/. The stack provides metrics collection, visualization, SNMP network monitoring, and service availability monitoring.

**Components deployed:**
- **Prometheus** (port 9090) — metrics scraping and storage with 30-day retention
- **Node Exporter** (port 9100) — Linux host metrics (CPU, RAM, disk, network)
- **SNMP Exporter** (port 9116) — OPNsense interface metrics via SNMP v2c
- **Grafana** (port 3000) — dashboard visualization with two data sources (Prometheus + Loki)
- **Uptime Kuma** (port 3001) — service availability monitoring with 7 active monitors

**Grafana dashboards:**
- Node Exporter Full (imported, ID 1860) — comprehensive Linux host metrics
- OPNsense Monitoring (custom) — per-interface traffic with friendly names (WAN, Trusted, Lab, IoT, Guest) via Prometheus metric_relabel_configs

**SNMP details:**
- OPNsense Net-SNMP plugin (v2c)
- Official default SNMP Exporter config (61k lines, if_mib module)
- Interface names relabeled from raw IDs (vtnet0, vlan01) to friendly names in Prometheus

---

## Phase 8 — Log Aggregation
**Status:** ✅ Complete

Deployed centralized log aggregation using Grafana Loki and Grafana Alloy (replaced Promtail, which reached EOL March 2026). Collects Docker container logs and OPNsense firewall logs in a single searchable system.

**Components deployed:**
- **Loki** (port 3100) — log storage with label-based indexing and 7-day retention
- **Grafana Alloy** (port 12345, UDP 5514) — unified log collection agent

**Log sources:**
- Docker container logs — auto-discovered via Docker socket, labeled per container
- OPNsense firewall logs — received via UDP syslog on port 5514, parsed as RFC3164 (BSD format)

**Grafana dashboards:**
- Docker Logs (custom) — live log stream, filtered error logs, log volume by container
- OPNsense syslog queryable in Grafana Explore with LogQL

**Design decision:** Chose Grafana Alloy over Promtail because Promtail reached end-of-life in March 2026. Alloy is the current, actively maintained replacement with broader capabilities (metrics, logs, traces) and a built-in web UI for pipeline visualization.

---

## Phase 9 — Active Directory Lab
**Status:** ⏳ Planned

Rebuild the Active Directory lab from VirtualBox inside Proxmox with proper VLAN segmentation.
