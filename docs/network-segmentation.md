# Network Segmentation & Zone Access Architecture

The `nerv.geofront` environment adheres to least-privilege Zero Trust routing, enforced centrally at the Netgate SG-3100 (`HASEGAWA`) gateway running pfSense. Inter-VLAN communication is denied by default; inter-zone traffic requires explicit, stateful firewall pinholes.

---

## 1. Subnet & VLAN Allocation

All networks are carved out of a primary `10.0.0.0/8` private address space:

| VLAN ID | Name | Subnet / CIDR | Purpose & Workloads | Gateway IP | Ingress Access Policy |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | `MGMT` | `10.0.10.0/24` | Out-of-band & hypervisor control: Proxmox (`.10`), HP iLO 4 (`.11`), UniFi switch (`.2`), Netgate webConfigurator | `10.0.10.1` | Isolated; accessible only via admin workstations |
| **20** | `TRUSTED` | `10.0.20.0/24` | Admin workstations and personal endpoints | `10.0.20.1` | Unrestricted WAN outbound; stateful access to MGMT and SERVICES |
| **30** | `MEDIA_STORAGE` | `10.0.30.0/24` | High-throughput storage traffic; Synology `GOGUL` NFS/SMB endpoints | `10.0.30.1` | Isolated; media consumers & authorized backup targets only |
| **40** | `GUEST` | `10.0.40.0/24` | Mobile endpoints and IoT devices | `10.0.40.1` | WAN egress only; strict RFC 1918 drop rule |
| **50** | `SERVICES` | `10.0.50.0/24` | Core containerized services, reverse proxies, and identity management | `10.0.50.1` | Ingress only via reverse proxy or designated admin ports |
| **66** | `DMZ_LAB` | `10.0.66.0/24` | Isolated testing sandboxes, Kali Linux VMs, and volatile experimental labs | `10.0.66.1` | Complete isolation; zero lateral traversal to internal zones |

---

## 2. Hypervisor Switching Model (`/etc/network/interfaces`)

Rather than binding static Linux bridges to physical NIC sub-interfaces (`vmbr0.10`, `vmbr0.20`), `unit-01` leverages a unified **802.1Q VLAN-aware bridge**:

```text
auto lo
iface lo inet loopback

iface nic1 inet manual

auto vmbr0
iface vmbr0 inet manual
        bridge-ports nic1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094

# Hypervisor Layer-3 Management Interface
auto vmbr0.10
iface vmbr0.10 inet static
        address 10.0.10.10/24
        gateway 10.0.10.1

iface nic0 inet manual
iface nic2 inet manual

source /etc/network/interfaces.d/*
```
### Architectural Rationale

- iface vmbr0 inet manual: Strips Layer-3 addressing from the raw bridge, treating vmbr0 as a pure Layer-2 managed switch ASIC.

- bridge-vlan-aware yes: Directs the Linux kernel to filter 802.1Q tags natively on virtual tap/veth interfaces.

- auto vmbr0.10: Anchors the Proxmox control plane exclusively to VLAN 10, while allowing arbitrary guest containers (e.g., UniFi controller CT 100) and virtual machines to span arbitrary VLANs via simple GUI tag assignment.
