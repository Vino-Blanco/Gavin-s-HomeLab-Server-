# Lessons Learned

Real troubleshooting experiences and key takeaways from building this homelab. These are the things that aren't in any textbook — you learn them by breaking things and fixing them.

---

## Networking

### Keep WAN and LAN on separate subnets
Originally, both vmbr0 (WAN) and vmbr1 (LAN) were on 192.168.0.0/24. This caused routing conflicts because Proxmox didn't know which bridge to use for traffic destined to that subnet. Splitting them — WAN on 192.168.0.0/24 and LAN on 10.0.x.0/24 — resolved everything.

### OPNsense firewall rules use direction "in"
Interface firewall rules in OPNsense evaluate traffic on the **inbound** direction — traffic coming into the firewall from that interface's network. Setting direction to "out" means the rule evaluates traffic going out toward the network, which is almost never what you want. I initially set it to "out" and spent time debugging why the firewall was blocking everything.

### VLAN-aware bridges act like managed switches
Enabling `bridge-vlan-aware yes` on Proxmox's vmbr1 fundamentally changed how the bridge handles traffic. Without it, the bridge is a dumb pass-through. With it, the bridge starts filtering VLANs — and every VM port must be explicitly configured as a trunk or access port, or traffic gets dropped.

### Trunk ports need explicit configuration
When I enabled VLAN-aware on vmbr1, OPNsense's NIC had no trunk configuration. The bridge started filtering tagged traffic, so OPNsense stopped receiving VLANs 10, 20, 30, and 40. The fix was adding `trunks=10;20;30;40` to OPNsense's net1 definition so the bridge knew to forward those tagged VLANs.

### PVID matters for untagged traffic
The switch management interface uses VLAN 1 (untagged). When I enabled VLAN-aware on vmbr1 with `bridge-vids 2-4094`, VLAN 1 was excluded, and untagged management traffic was dropped. Adding `bridge-vids 1-4094` and `bridge-pvid 1` fixed this by telling the bridge to accept and properly handle untagged VLAN 1 traffic.

### The switch has no direct path to the router
The JGS516PE is only connected to the EliteDesk via Port 1. It has no direct cable to the Cox router. This means switch management traffic must flow through vmbr1 — so any change to vmbr1 (like enabling VLAN-aware) can cut off switch management access.

---

## Proxmox

### Bridge changes often need a full reboot
`ifreload -a` applies network config changes, but bridge-level changes (especially enabling/disabling VLAN-aware) often don't take effect cleanly without a full host reboot.

### Proxmox firewall is separate from OPNsense
Proxmox has its own firewall that can be enabled per-VM NIC. This is completely independent of OPNsense's firewall. Having both active can cause confusing double-filtering. In our setup, we disabled Proxmox's firewall on VM NICs and let OPNsense handle all firewall duties.

### qm set requires full NIC definition
You can't just add `trunks=` to an existing NIC — you must redefine the entire net parameter including model, MAC address, and bridge. Also, semicolons in `trunks=10;20;30;40` must be quoted to prevent bash from interpreting them as command separators.

---

## Switch

### Always factory reset second-hand managed switches
The JGS516PE came with an old VLAN config that blocked traffic in unexpected ways. I had to factory reset it before our configuration would work. If you buy a used managed switch, reset it first.

### Don't remove yourself from the management VLAN
When configuring VLANs on the switch, I accidentally removed the management port from VLAN 1, locking ourselves out. The switch had to be factory reset again. Always ensure your management access remains intact before applying VLAN changes.

### JGS516PE doesn't support management VLAN selection
The Netgear "Smart Managed Plus" series doesn't allow you to change which VLAN the management interface uses. It's always on VLAN 1 (untagged). This is a hardware/firmware limitation — fully managed switches allow this.

---

## General

### pfctl -d is a nuclear option
`pfctl -d` disables OPNsense's packet filter entirely, including NAT. This means the firewall stops blocking AND the internet stops working (because NAT is gone). Use it only for emergency troubleshooting and always re-enable with `pfctl -e`.

### Document everything immediately
The complexity of VLAN configurations across three systems (OPNsense, Proxmox, switch) makes it easy to lose track of what's configured where. Documenting every change as it happens prevents confusion during troubleshooting.

### Test after every change
Making multiple changes before testing makes it nearly impossible to identify which change broke something. Change one thing, test, confirm, then move on.
