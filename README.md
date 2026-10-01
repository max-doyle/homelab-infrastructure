# nerv.geofront — Zero-Trust Virtualized Infrastructure

An enterprise-patterned on-premises infrastructure designed around strict L2/L3 segmentation, isolated management planes, and resilient hypervisor compute.

---

## High-Level Topology

```text
                      +-----------------------------+
                      |   Netgate SG-3100 (Gateway) |
                      |         [HASEGAWA]          |
                      +--------------+--------------+
                                     | 802.1Q Trunk
                      +--------------+--------------+
                      |   UniFi US-8-150W (Switch)  |
                      |        [MAGI-SYSTEM]        |
                      +---+----------+----------+---+
                          |          |          |
         +----------------+          |          +----------------+
         | (VLAN 10/20/50 Trunk)     | (Access VLAN 30)          | (Dedicated MGMT)
+--------+--------+          +-------+-------+          +--------+--------+
|   Proxmox VE    |          | Synology NAS  |          | HP MicroServer  |
|   [unit-01]     |          |    [GOGUL]    |          |   [Server B]    |
+-----------------+          +---------------+          +-----------------+
```
### Segmentation Schema

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

### Core Compute Node (`unit-01.nerv.geofront`)
* **Processor**: Intel Core i7-4790K (4 cores, 8 threads @ 4.00GHz base)
* **Motherboard**: ASRock Z97 Extreme6
* **Memory**: 16GB DDR3
* **Primary Storage**: 500GB M.2 SATA SSD
* **Operating System**: Proxmox VE 9.2 (Debian base)
* **Network Integration**: Single physical interface trunked via 802.1Q-aware Linux bridge (`vmbr0`), binding host management exclusively to tagged VLAN 10.

### Out-of-Band & Backup Tier (`Server B`)
* **Hardware**: HP ProLiant MicroServer Gen8
* **Processor**: Intel Xeon E3-1220L v2 (2 cores, 4 threads, 17W TDP)
* **Memory**: 16GB ECC DDR3
* **Management**: Dedicated physical HP iLO 4 interface assigned static IP on `VLAN 10 (MGMT)` for remote hardware diagnostics and pre-boot bare-metal power cycling.
* **Target Role**: Proxmox Backup Server (PBS) & secondary quorum witness.

### Centralized Storage Tier (`GOGUL`)
* **Hardware**: Synology DS1815+
* **Storage Pools**: 
  * Primary Pool: 4 × 4TB HDDs (Bulk array / archival storage)
  * High-Performance Pool: 3 × 250GB SSDs (Low-latency container storage)
* **Network Placement**: Pinned exclusively to `VLAN 30 (MEDIA/STORAGE)`.

---

## 3. Active Workloads & Services

* **UniFi Network Application (`CT 100`)**: Hosted within an unprivileged Debian 12 (Bookworm) LXC container on `unit-01`, utilizing MongoDB 7.0 and OpenJDK 17 to orchestrate switching profiles and AP provisioning.
* **Identity & Recursive Ingress (In Progress)**:
  * Local split-horizon recursive DNS via Pi-hole + Unbound.
  * Centralized Identity Provider (IdP) via Authentik running WebAuthn/FIDO2.
  * Encrypted mesh access via WireGuard jumpbox (zero exposed listening ports to public WAN).

---

## 4. Key Engineering Implementations

* **VLAN-Aware Virtual Switching**: Configured `/etc/network/interfaces` on Proxmox using `bridge-vlan-aware yes`, turning the Linux bridge into a managed virtual switch. Guest containers and VMs receive direct VLAN tagging via virtual NICs without creating static interface abstractions on the host.
* **Out-of-Band (OOB) Resilience Planning**: Designed headless serial console integration (`ser2net`) over RS-232/rollover interfaces for the Netgate SG-3100 and UniFi US-8-150W switch to maintain administrative access during trunking and control-plane migrations without physical cable swaps.

---

## 5. Security & Sanitization Notice

All configurations, routing definitions, and addressing schemes in this repository are sanitized for public release. Internal IP ranges use RFC 1918 space, and sensitive telemetry, pre-shared keys, and public IP mappings have been scrubbed to reflect strict operational security standards.
