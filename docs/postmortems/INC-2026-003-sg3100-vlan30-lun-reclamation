# Incident Post-Mortem: Inter-VLAN Storage Blackhole & Orphaned 8.8 TB Block Allocation

| Metadata Field | Value |
| :--- | :--- |
| **Incident ID** | `INC-2026-002` |
| **Target / Impacted System** | Netgate SG-3100 (`HASEGAWA`), Synology DS1815+ (`GOGUL`) |
| **Severity Level** | P2 (Degraded / Storage Plane Unreachability & Capacity Starvation) |
| **Date & Time (UTC/Local)** | 2026-10-29 12:45 EDT |
| **Resolution Status** | Resolved / Root Cause Identified & Capacity Reclaimed |
| **Author / Lead Operator** | Max Doyle / Lead Infrastructure Operator |

---

## 1. Executive Summary
During the migration of storage array `GOGUL` to dedicated VLAN 30 (`10.0.30.2`), all Layer 3 connectivity (ICMP/TCP) dropped between administrative workstations and the storage plane despite correct pfSense interface configurations and firewall pinholes. Subsequent storage auditing revealed Volume 1 was starved at 85% capacity due to an unmapped, thick-provisioned 8.8 TB legacy block allocation (`LUN-1`). Hardware switch register tagging restored inter-VLAN routing, and non-destructive initiator auditing allowed the verified destruction of `LUN-1`, reclaiming 8.8 TB and restoring Volume 1 capacity to ~1% baseline.

---

## 2. Telemetry & Observable Symptoms
* **Data Plane / Workloads**: Layer-2 physical link on UniFi Switch (`MAGI-SYSTEM` Port 7) negotiated at 1000 Mbps with active RX/TX counters. Layer-3 egress packets entering `HASEGAWA` on VLAN 30 were dropped in hardware silicon before reaching the kernel routing table or ARP cache.
* **Control / Management Plane**: Synology DSM reachable via web interface only prior to VLAN re-assignment. After cutover, `https://10.0.30.2:5001` and TCP port 3260 (iSCSI) timed out completely.
* **Network & ARP Telemetry**: Gateway ARP tables on pfSense (`mvneta1.30`) initially showed no entry or `incomplete` for `10.0.30.2`.
* **Storage Allocation**: Volume 1 reported 8.9 TB / 10.5 TB (84-85% utilization) with only 109.5 GB of active files in shared folders. SAN Manager reported an unmapped 8.8 TB thick-provisioned LUN.

---

## 3. Incident Timeline
| Timestamp (UTC/Local) | Event / Action Taken | Observed Response / State |
| :--- | :--- | :--- |
| **T0** | Re-profiled `MAGI-SYSTEM` Port 7 to VLAN 30 Access mode and re-IP'd `GOGUL` to `10.0.30.2`. | `GOGUL` unroutable across subnets; ICMP and TCP 5001/3260 failed. |
| **T+15m** | Diagnosed Netgate SG-3100 switch architecture. Inspected `Interfaces -> Switches -> VLANs`. | Discovered VLAN 30 was omitted from the Marvell 88E6141 hardware switch matrix. |
| **T+20m** | Programmed 802.1Q tags `1t, 5t` across all active subnets (10, 20, 30, 40, 50, 66). | Layer-3 routing and ARP populated immediately; DSM reached at `10.0.30.2`. |
| **T+30m** | Attempted iSCSI connection via Windows initiator to audit 8.8 TB `LUN-1`. | Initiator threw `Unable to Login to the target` due to multi-session lockouts on legacy target. |
| **T+40m** | Staged isolated read-only target `Target-LUN1-Audit` bound to `LUN-1`. | Initiator attached instantly; Windows Disk Management confirmed raw unpartitioned space. |
| **T+50m** | Destroyed `LUN-1` and removed temporary firewall pinholes. | 8.8 TB raw capacity released; Volume 1 utilization dropped from 85% to 1%. |

---

## 4. Root Cause Analysis (RCA)
* **Trigger 1 (Switch Routing Blackhole)**: The Netgate SG-3100 integrates an internal Marvell 88E6141 switch chip that abstracts physical ports from the FreeBSD kernel (`mvneta1`). While the pfSense logical VLAN interface (`mvneta1.30`) was defined, the underlying switch ASIC register was never programmed to forward 802.1Q Tag 30 between Port 1 (physical trunk) and Port 5 (CPU uplink).
* **Trigger 2 (Capacity Starvation)**: `LUN-1` was provisioned as a "Thick Provisioned" block device during early homelab staging. Thick provisioning pre-allocates and locks the entire block table on Btrfs/ext4 volumes regardless of actual guest filesystem usage. Because it was never mapped to a target or formatted, 8.8 TB sat orphaned as dead allocation.

---

## 5. Remediation & Recovery Steps
1. **ASIC Switch Table Enforcement**:
   - Declared VLAN 30 and all future security zones in pfSense under `Interfaces -> Switches -> VLANs` with Member Ports set to `1t, 5t`.
2. **Ephemeral Target Auditing**:
   - Created clean, temporary iSCSI target `Target-LUN1-Audit` configured with `Enable multiple sessions` and read-only LUN permissions.
   - Connected `Hulle-Granz-Cathedral` via `iscsicpl.exe` to inspect partition tables non-destructively in `diskmgmt.msc`.
3. **Destruction & Reclamation**:
   - Unmapped and deleted `LUN-1` in Synology SAN Manager.
   - Deleted temporary diagnostic pass rules on pfSense `TRUSTED` interface.

---

## 6. Preventative & Architectural Action Items
* [x] **Short-Term (Config / Process)**: Batch-tag all planned enterprise VLANs (`10, 20, 30, 40, 50, 66`) with `1t, 5t` in the SG-3100 Marvell switch table to prevent future inter-VLAN blackholes.
* [x] **Short-Term (Storage Hygiene)**: Verify block-level provisioning policies; mandate Thin Provisioning with space reclamation (TRIM/UNMAP) for hypervisor LUNs.
* [ ] **Medium-Term (Documentation / Git)**: Commit `INC-2026-002` to `homelab-infrastructure/docs/post-mortems/` to preserve operational history for portfolio review.
