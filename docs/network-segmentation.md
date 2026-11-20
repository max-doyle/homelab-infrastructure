# Network Segmentation & Zone Access Architecture

The `nerv.geofront` environment adheres to least-privilege Zero Trust routing, enforced centrally at the Netgate SG-3100 (`HASEGAWA`) gateway running pfSense. Inter-VLAN communication is denied by default; cross-zone traffic requires explicit, stateful firewall pinholes backed by object aliases and funneled through a centralized TLS reverse-proxy chokepoint.

---

## 1. Subnet & VLAN Allocation

All networks are carved out of a primary `10.0.0.0/8` private RFC 1918 address space:

| VLAN ID | Name | Subnet / CIDR | Purpose & Workloads | Gateway IP | Ingress Access Policy |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | `MGMT` | `10.0.10.0/24` | Out-of-band & hypervisor control: `unit-01` (`.10`), HP iLO 4 (`.11`), `unit-02` PBS (`.12`), `MAGI-SYSTEM` (`.2`), UniFi `CT 100` (`.24`), `HASEGAWA` (`.1`) | `10.0.10.1` | Isolated; reverse-proxy (`10.0.50.11`) & emergency admin pinholes only |
| **20** | `TRUSTED` | `10.0.20.0/24` | Primary admin workstations (`Hulle-Granz-Cathedral` at `.10`) and personal endpoints | `10.0.20.1` | Unrestricted WAN outbound; stateful access to `SERVICES` (`53`, `443`) |
| **30** | `MEDIA_STORAGE` | `10.0.30.0/24` | High-throughput storage traffic: Synology `GOGUL` (`.2`) iSCSI/NFS/SMB and `TentacleShop` (`.3`) NAS | `10.0.30.1` | Isolated; authorized hypervisor initiators (`TCP 3260`) & proxy targets only |
| **40** | `GUEST` | `10.0.40.0/24` | Untrusted mobile endpoints and IoT appliances | `10.0.40.1` | WAN egress only; strict RFC 1918 drop rule |
| **50** | `SERVICES` | `10.0.50.0/24` | Core containerized infrastructure: `CT 101` Pi-hole/Unbound DNS (`.10`), `CT 102` Nginx Proxy Manager (`.11`), future IdP | `10.0.50.1` | Unified TLS ingress (`TCP 443`) from `TRUSTED`; no direct WAN ingress |
| **66** | `DMZ_LAB` | `10.0.66.0/24` | Isolated testing sandboxes, Kali Linux VMs, and volatile experimental labs | `10.0.66.1` | Complete isolation; zero lateral traversal to internal zones |

---

## 2. Hypervisor Switching Model (`/etc/network/interfaces`)

Rather than binding static Linux bridges to physical NIC sub-interfaces (`vmbr0.10`, `vmbr0.20`), `unit-01` leverages a unified **802.1Q VLAN-aware bridge**:

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

### Architectural Rationale

* **`iface vmbr0 inet manual`**: Strips Layer-3 addressing from the raw bridge, treating `vmbr0` as a pure Layer-2 managed switch fabric.
* **`bridge-vlan-aware yes`**: Directs the Linux kernel to filter 802.1Q tags natively on virtual tap/veth interfaces.
* **`auto vmbr0.10`**: Anchors the Proxmox host control plane exclusively to `VLAN 10`, while allowing guest LXC containers (`CT 100` on `VLAN 10`, `CT 101` & `CT 102` on `VLAN 50`) and VMs to span arbitrary VLANs via virtual NIC tag assignment.

---

## 3. Zero-Trust Ingress & Firewall Alias Architecture (`HASEGAWA`)

To eliminate broad subnet rules and reduce workstation exposure to management and storage planes, administrative web access is decoupled into a **two-hop reverse-proxy pipeline**:

    [TRUSTED: 10.0.20.10] ──(HTTPS :443 / *.nerv.geofront)──▶ [SERVICES: 10.0.50.11 (CT 102 NPM)]
                                                                            │
                                      ┌─────────────────────────────────────┴─────────────────────────────────────┐
                                      ▼ (HTTPS :8006/:8007/:8443/:443)                                            ▼ (HTTPS :5001/:443)
                           [MGMT: 10.0.10.0/24]                                                       [STORAGE: 10.0.30.0/24]
                     (unit-01, unit-02, iLO, UniFi, pfSense)                                           (GOGUL DSM, TentacleShop)

### Firewall Object Aliases

Rules on `HASEGAWA` reference compiled IP and Port aliases rather than raw `/24` CIDR blocks or scattered single-host rules:

* **`NPM_Backend_Hosts` (IP Alias)**: `10.0.10.1` (`HASEGAWA`), `10.0.10.10` (`unit-01`), `10.0.10.11` (`iLO 4`), `10.0.10.12` (`unit-02`), `10.0.10.24` (`MAGI-SYSTEM` Controller), `10.0.30.2` (`GOGUL`), `10.0.30.3` (`TentacleShop`).
* **`NPM_Backend_Ports` (Port Alias)**: `443` (HTTPS / iLO / QTS / Pi-hole), `5001` (Synology DSM), `8006` (Proxmox VE), `8007` (Proxmox Backup Server), `8443` (UniFi Network Application).
* **`SERVICES_WebAdmin_Hosts` (IP Alias)**: `10.0.50.10` (`CT 101`), `10.0.50.11` (`CT 102`).

### Core Inter-VLAN Policy Matrix

1. **DNS Resolution (`TRUSTED` / `MGMT` / `STORAGE` -> `10.0.50.10:53`)**: Permits `UDP/TCP 53` to Pi-hole + Unbound (`CT 101`), which resolves all `*.nerv.geofront` FQDNs to `10.0.50.11` while validating external queries via DNSSEC (`127.0.0.1#5335`).
2. **Unified TLS Ingress (`TRUSTED` -> `10.0.50.11:443`)**: Admin workstations connect exclusively over standard HTTPS (`443`) to Nginx Proxy Manager (`CT 102`), which terminates TLS using a SAN-compliant `*.nerv.geofront` certificate signed by the internal `NERV Geofront Internal Root CA`.
3. **Proxy Backend Egress (`10.0.50.11` -> `NPM_Backend_Hosts` : `NPM_Backend_Ports`)**: A single stateful rule on the `SERVICES` interface authorizes `CT 102` to hairpin TLS sessions to backend management interfaces across `VLAN 10` and `VLAN 30`.
4. **Storage Fabric (`unit-01` `10.0.10.10` -> `GOGUL` `10.0.30.2:3260`)**: Dedicated stateful pinhole permitting iSCSI block transport (`TCP 3260`) between the hypervisor initiator and the `containertruck` RAID-1 flash target.