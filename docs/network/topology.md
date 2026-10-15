# nerv.geofront Network Topology & Management Plane Baseline

## Status as of 2026-10-15
- **Hypervisor (unit-01)**: Proxmox VE isolated on VLAN 10 (10.0.10.10/24) via bridge-vlan-aware trunking.
- **Switch Fabric (MAGI-SYSTEM)**: UniFi US-8-150W management interface bound to VLAN 10 (10.0.10.2/24) via Network Override.
- **Controller (CT 100)**: UniFi Network Application migrated to VLAN 10 (10.0.10.24/24), informing over TCP 8080.
- **Workstation (Hulle-Granz-Cathedral)**: Pinned to VLAN 20 (10.0.20.10/24) via Port 6 access profile.
- **Legacy Subnet (192.168.1.0/24)**: Vacated; 0 active DHCP leases.
