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
- Firewall: Pass-all rules on each VLAN (to be locked down in a future phase)

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
**Status:** ⏳ Planned

Replace pass-all firewall rules with restrictive policies:
- Trusted: Full access to all VLANs and internet
- Lab: Internet access, limited access to Trusted
- IoT: Internet only, no inter-VLAN access
- Guest: Internet only, fully isolated

---

## Phase 7 — Monitoring Stack
**Status:** ⏳ Planned

Deploy Grafana + InfluxDB + Telegraf for infrastructure monitoring — CPU, RAM, disk, network traffic across all VMs and the Proxmox host.

---

## Phase 8 — Active Directory Lab
**Status:** ⏳ Planned

Rebuild the Active Directory lab from VirtualBox inside Proxmox with proper VLAN segmentation.
