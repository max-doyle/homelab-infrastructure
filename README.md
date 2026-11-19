# nerv.geofront — Zero-Trust Virtualized Infrastructure

An enterprise-patterned on-premises infrastructure designed around strict 802.1Q L2/L3 segmentation, isolated out-of-band management planes, split-horizon recursive DNS, and centralized TLS ingress termination.

---

## 1. High-Level Topology & Ingress Architecture

```text
                      +-----------------------------+
                      |   Netgate SG-3100 (Gateway) |
                      |         [HASEGAWA]          |
                      +--------------+--------------+
                                     | 802.1Q Trunk (VLANs 10, 20, 30, 40, 50, 66)
                      +--------------+--------------+
                      |   UniFi US-8-150W (Switch)  |
                      |        [MAGI-SYSTEM]        |
                      +---+----------+----------+---+
                          |          |          |
         +----------------+          |          +----------------+
         | (VLAN 10/20/50/66 Trunk)  | (Access VLAN 30)          | (Dedicated MGMT & OS)
+--------+--------+          +-------+-------+          +--------+--------+
|   Proxmox VE    |          | Storage Tier  |          | Backup & OOB    |
|   [unit-01]     |          | [GOGUL / NAS] |          |   [unit-02]     |
+-----------------+          +---------------+          +-----------------+
```
### Segmentation Schema
Inter-VLAN routing is denied by default at the pfSense gateway (HASEGAWA). Cross-zone traffic is governed by stateful firewall aliases (NPM_Backend_Hosts, NPM_Backend_Ports) funneled through a centralized reverse-proxy ingress chokepoint.

| VLAN ID | Subnet CIDR | Security Zone | Purpose & Workloads | Ingress Policy |
| :--- | :--- | :--- | :--- | :--- |
| **10** | `10.0.10.0/24` | **MGMT** | Proxmox hypervisor, iLO 4, switch & gateway management | Isolated; admin workstation pinholes only |
| **20** | `10.0.20.0/24` | **TRUSTED** | Primary administrative workstations & personal devices | Full outbound; stateful access to MGMT/SERVICES |
| **30** | `10.0.30.0/24` | **MEDIA / STORAGE** | High-throughput storage arrays (NFS/SMB/iSCSI) | Isolated; media consumers & backup hosts only |
| **40** | `10.0.40.0/24` | **GUEST** | Untrusted endpoints & mobile devices | WAN egress only; strict RFC 1918 block |
| **50** | `10.0.50.0/24` | **SERVICES** | Core containerized applications & internal identity | Internal proxy access only; no direct WAN ingress |
| **66** | `10.0.66.0/24` | **DMZ / LAB** | Sandboxed workloads, Kali testing instances, volatile labs | Zero lateral internal access; strictly ringfenced |

---

## 2. Compute & Hypervisor Infrastructure

The compute tier runs on Proxmox VE (Debian base), utilizing a kernel-level 802.1Q VLAN-aware bridge to decouple the hypervisor control plane from untagged physical traffic.

### Primary Compute Node (`unit-01.nerv.geofront` - `10.0.10.10`)
* **Platform**: Proxmox VE 9.2 (Debian base)
* **Network Integration**: Single physical interface trunked via 802.1Q-aware Linux bridge (`vmbr0`), binding host management exclusively to tagged VLAN 10.

### Out-of-Band & Backup Tier (`unit-02.nerv.geofront` - `10.0.10.12`)
* **Platform**: Platform: Proxmox Backup Server (PBS) backed by a mirrored ZFS datastore (`backup-pool`) with client-side AES-256-GCM encryption and weekly scrub/verification schedules.
* **Out-of-Band (OOB) BMC**: Dedicated physical HP iLO 4 interface pinned to `VLAN 10` (MGMT) (`10.0.10.11`) on a separate physical switch port for pre-boot diagnostics and bare-metal power cycling.
  
### Centralized Storage Tier (`VLAN 30` — `10.0.30.0/24`)
* **Primary Storage Array (`GOGUL` - `10.0.30.2`)**: 
  * **Bulk Pool**: 4 × 4TB HDDs (Bulk array / archival storage)
  * **Flash Block Pool**: 2 × 500GB RAID-1 mirror erving low-latency iSCSI LUN block storage to unit-01 container workloads.
* **Secondary NAS Node** (`NAS` - `10.0.30.3`): Pinned exclusively to `VLAN 30 (MEDIA/STORAGE)`.

---

## 3. Active Workloads & Services

* **Network Orchestration (`CT 100` — `magi-system.nerv.geofront`)**: Unprivileged Debian 12 LXC container on `VLAN 10` (`10.0.10.24`) running the UniFi Network Application (MongoDB 7.0 + OpenJDK 17) for switch and AP profile management.

* **Split-Horizon Recursive DNS (`CT 101` — `pi-hole.nerv.geofront`)**: Unprivileged Debian 12 LXC container on `VLAN 50` (`10.0.50.10`) pairing Pi-hole v6 sinkholing with a local Unbound recursive resolver (`127.0.0.1#5335`) enforcing DNSSEC root validation and authoritative local resolution for `*.nerv.geofront`.

* **Reverse Proxy & Internal PKI (`CT 102` — `nginx.nerv.geofront`)**: Unprivileged Debian 12 LXC container on `VLAN 50` (`10.0.50.11`) running Nginx Proxy Manager behind a custom Internal Root Certificate Authority and SAN-compliant `*.nerv.geofront` wildcard TLS certificate.

* Identity & Remote Access (Roadmap):
    * Centralized Identity Provider (IdP) via Authentik enforcing WebAuthn/FIDO2 MFA at the reverse-proxy chokepoint.
    * Peer-to-peer WireGuard / Headscale jumpbox with zero exposed WAN listening ports.
---

## 4. Key Engineering Implementations

* **802.1Q VLAN-Aware Virtual Switching**: Configured `/etc/network/interfaces` on Proxmox using `bridge-vlan-aware yes`, turning the Linux bridge into a managed virtual switch. Guest containers and VMs receive direct VLAN tagging via virtual NICs without creating static interface abstractions on the host.
* **Out-of-Band (OOB) Resilience Planning**: Designed headless serial console integration (`ser2net`) over RS-232/rollover interfaces for the Netgate SG-3100 and UniFi US-8-150W switch to maintain administrative access during trunking and control-plane migrations without physical cable swaps.

---
## 5. Operational Documentation & Post-Mortems
- **Network Architecture**: docs/network-segmentation.md • docs/network/topology-reference.md
- **Operational Runbooks**: Runbook-containertruck-maintenance-lifecycle-control.md • Runbook-unit-02-pbs.md
- **Incident Post-Mortems**: docs/postmortems/

## 6. Security & Sanitization Notice

All configurations, routing definitions, and addressing schemes in this repository are sanitized for public release. Internal IP ranges use RFC 1918 space, and sensitive telemetry, pre-shared keys, and public IP mappings have been scrubbed to reflect strict operational security standards.
