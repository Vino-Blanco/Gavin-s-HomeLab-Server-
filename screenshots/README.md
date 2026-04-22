# Screenshots

Visual documentation of every major component in the homelab. Each screenshot shows a real, working configuration -- nothing staged or mocked up.

---

## Proxmox Hypervisor

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 01 | [Proxmox Summary](01-proxmox-summary.png) | Datacenter overview with both VMs running, resource usage across the node |
| 02 | [Proxmox Network](02-proxmox-network.png) | Network device layout -- physical NICs (eno1, enp2s0f0, enp2s0f1) and virtual bridges (vmbr0, vmbr1, vmbr2) |
| 03 | [VM 100 Hardware](03-proxmox-vm100-hardware.png) | Ubuntu Server VM config -- 4GB RAM, 2 cores, 32GB disk, net0 on vmbr1 with tag=20 |
| 04 | [VM 101 Hardware](04-proxmox-vm101-hardware.png) | OPNsense VM config -- 2GB RAM, 2 cores, 3 NICs (WAN, LAN trunk, OPT1) with trunks=10;20;30;40 |
| 05 | [Bridge VLAN Config](05-proxmox-bridge-vlan.png) | Proxmox shell showing VLAN-aware bridge config -- tap100i0 on VLAN 20, OPNsense trunk carrying VLANs 10/20/30/40 |

## Client Verification

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 06 | [Main PC ipconfig](06-main-pc-ipconfig.png) | Windows ipconfig output -- Ethernet on 10.0.10.10 (VLAN 10 Trusted), Wi-Fi on 192.168.0.128 (WAN) |
| 07 | [Main PC Ping Tests](07-main-pc-ping-tests.png) | Connectivity verification -- pinging gateway (10.0.10.1), Ubuntu Server (10.0.20.20), and internet (8.8.8.8) |
| 08 | [Ubuntu Server IP](08-ubuntu-server-ip.png) | `ip a` output showing ens18 at 10.0.20.20/24 on VLAN 20 with Docker bridge network |
| 09 | [Ubuntu Netplan](09-ubuntu-server-netplan.png) | Static network config -- 10.0.20.20/24, gateway 10.0.20.1, DNS 8.8.8.8/8.8.4.4 |

## OPNsense Firewall

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 10 | [OPNsense Dashboard](10-opnsense-dashboard.png) | System overview -- uptime, CPU, memory, disk, interface statistics, services status, traffic graphs |
| 11 | [Interfaces Overview](11-opnsense-interfaces-overview.png) | All interfaces with IPs -- WAN (192.168.0.135), LAN (10.0.0.1), Trusted/Lab/IoT/Guest VLANs |
| 12 | [VLAN Devices](12-opnsense-vlan-devices.png) | VLAN sub-interfaces -- vlan01-04 on parent vtnet1 with tags 10, 20, 30, 40 |

## OPNsense DHCP

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 13 | [DHCP Trusted](13-opnsense-dhcp-trusted.png) | VLAN 10 DHCP -- subnet 10.0.10.0, range .100-.200, gateway 10.0.10.1 |
| 14 | [DHCP Lab](14-opnsense-dhcp-lab.png) | VLAN 20 DHCP -- subnet 10.0.20.0, range .100-.200, gateway 10.0.20.1 |
| 15 | [DHCP IoT](15-opnsense-dhcp-iot.png) | VLAN 30 DHCP -- subnet 10.0.30.0, range .100-.200, gateway 10.0.30.1 |
| 16 | [DHCP Guest](16-opnsense-dhcp-guest.png) | VLAN 40 DHCP -- subnet 10.0.40.0, range .100-.200, gateway 10.0.40.1 |

## Firewall Rules

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 17 | [FW Rules Trusted](17-opnsense-fw-rules-trusted.png) | Single pass-all rule -- Trusted gets full access to everything |
| 18 | [FW Rules Lab](18-opnsense-fw-rules-lab.png) | Allow to Trusted, block to IoT and Guest, allow internet |
| 19 | [FW Rules IoT](19-opnsense-fw-rules-IoT.png) | Block to all other VLANs and OPNsense, allow internet only |
| 20 | [FW Rules Guest](20-opnsense-fw-rules-guest.png) | Fully isolated -- block to Trusted, Lab, IoT, and OPNsense, allow internet only |

## Switch Configuration

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 21 | [VLAN Config](21-switch-vlan-config.png) | 802.1Q enabled, VLANs 1/10/20/30/40 created with port membership |
| 22 | [VLAN 1 Membership](22-switch-vlan1-membership.png) | All 16 ports untagged on VLAN 1 (management) |
| 23 | [VLAN 10 Membership](23-switch-vlan10-membership.png) | Port 1 tagged (trunk), Port 2 untagged (access for Main PC) |
| 24 | [Port PVID](24-switch-port-pvid.png) | Port 1 PVID=1 (trunk default), Port 2 PVID=10 (Trusted access port) |

## Monitoring Dashboards

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 25 | [Node Exporter Full](25-grafana-node-exporter-full.png) | Linux host metrics -- CPU, memory, network traffic, disk usage over 24 hours |
| 26 | [OPNsense Monitoring](26-grafana-opnsense-monitoring.png) | Custom dashboard -- per-interface traffic with friendly names (WAN, Trusted, Lab, IoT, Guest), interface count, SNMP scrape time |
| 27 | [Docker Logs](27-grafana-docker-logs.png) | Log aggregation dashboard -- error logs, container log stream, log volume by container |
| 28 | [OPNsense Syslog](28-grafana-opnsense-syslog-explore.png) | Firewall logs in Grafana Explore -- LogQL query showing block/pass events with IPs, ports, and protocols |
| 29 | [Prometheus Targets](29-prometheus-target-health.png) | All 3 scrape targets healthy -- node-exporter, prometheus, snmp-opnsense |

## Service Dashboards

| # | Screenshot | What it shows |
|---|-----------|---------------|
| 30 | [Uptime Kuma](30-uptime-kuma.png) | All 7 monitors up -- Grafana, Node Exporter, Prometheus, OPNsense, Proxmox, Loki, Alloy |
| 31 | [Portainer](31-portainer-dashboard.png) | Docker environment overview -- container count, images, volumes, system resources |
